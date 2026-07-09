# Welcome to mgmt!

Objectives: Introduce new folks to mgmt - its purpose, etc.

> 📚 Design of this document:
>
> - A gradual introduction with each section building on the previous.
> - Examples at the end of each section demonstrating what was just covered.
> - Avoiding jargon, especially internal implementation details, such as “DAG”, unless absolutely necessary. As an example, [Elm](https://guide.elm-lang.org) does a fantastic job of explaining how to use a functional programming language *without* burdening the reader in high-level mathematical terms that are unavoidable in language docs like Haskell. It’s OK to refer to these technical terms, “in mgmt, this is called x”

> ⚠️ XXX: Review for consistent use of terminology:
> - Verbs:  a resource *executes*, right?
> - Terminology overload: Notify in mcl, Refresh internal to mgmt, and in this document, Signalling. I also use ‘signalling’ to include send/recv.

# Welcome to mgmt config

- Scope and overview
    - You describe your desired configuration of a system, and mgmt makes it real. Mgmt config is an infrastructure as code tool
    - Function/purpose: configuration management, real-time response, one machine or clusters.
    - How to install [link to getting started download docs]
    -
- Features of mgmt config
    - fast: mgmt will perform multiple tasks simultaneously, ensuring that your systems reach their desired state more quickly. (ordering, relationships)
    - batching: mgmt will group similar operations together in order to improve performance. mgmt calls this “auto grouping"
    - reactive: it observes your system in real-time and responds to correct for any deviations (chapter: functions and resources)

# Programming mgmt with mcl

mcl is the language we use to program our desired state. mcl is strongly-typed, functional language <etc etc…>

This guide will teach you how to use mcl in mgmt and how mgmt behaves.

## Resources

Configuration is described with an element called a **resource**.
Most kinds of resources in mgmt will be familiar to you and are intended to align with your mental model of your infrastructure: ensuring that a package is installed, a service is disabled, or a user exists.

Resources have three parts: a kind, a name, and parameters (mgmt calls these 'params'). The following example code shows a single file _resource_:

```puppet { .m-2 }

file "/tmp/hello.txt" {
  content => "Greetings from mgmt!",
  state => "exists",
}
```

This is a file resource with a name “/tmp/hello.txt” and describes a state where you want the file to exist and contain exactly `Greetings from mgmt!`

You describe the desired state in each resource, and mgmt makes it real by observing the current state and making any necessary changes. An _mcl_ program can describe as many resources as you need.

Keyword: When all resources reach the desired state, **mgmt calls this outcome "converged".**

Our _mcl_ code can be executed in mgmt: `mgmt run lang <path>`. Mgmt will normally write its logs to stdout, so the execution of our file resource will appear like this:

```text { .m-2 }
$ mgmt run lang hello.mcl

... ( other output omitted for brevity ) ...

10:28:48 engine: file[/tmp/hello.txt]: copy 21 bytes
```

When you run this, notice the following:
* mgmt did not exit after converging (more on this in the next section)
* mgmt can run as any user, but that user will correct permissions in order to execute some resources.
* The first part '10:28:48' is a timestamp, and your timestamps will be different.

Let's verify that our file was created correctly, and run the following in another terminal:

```text { .m-2 }
$ cat /tmp/hello.txt
Greetings from mgmt!
```

In the example above, mgmt continues running even after converging. This is because mgmt continuously watches for changes (divergence) and will respond immediately and as needed.

Let's explore that! While mgmt is still running, we can modify the file in another terminal and observe mgmt's immediate responses:

```bash-session { .m-2 }
# Remove the file
$ rm /tmp/hello.txt

# mgmt notices and responds:
10:36:34 engine: file[/tmp/hello.txt]: copy 21 bytes

# Change the file's contents
$ echo "mgmt is fast" > /tmp/hello.txt

# mgmt notices and responds:
10:37:18 engine: file[/tmp/hello.txt]: copy 21 bytes
```

You can terminate mgmt in your terminal by pressing ctrl+c.

Each time we removed or modified the file, mgmt notices that the file has diverged from its desired state, and it takes action immediately to resolve it.

Resource definition syntax in mcl looks like this:

```puppet { .m-2 }
# This is a comment
kind "name" {
  # All params must end with a comma
  param1 => value1,

  # Including the final param
  param2 => value2,
}

# Did you know? mgmt has some internal resources 
# like a web server and it can serve files over http? :)
http:server "127.0.0.1:8080" { }
```

💡 _What’s in a name?_  Many resources use the name as a default value for a param. The file resource uses name as the default value for the `path` param.

⚠️Syntax Caution: In a resource, every param must have a trailing comma, including the last one.

The example above can also be written with a different name and setting the path explicitly:

```puppet { .m-2 }
file "a greeting" {
  content => "Greetings from mgmt!",
  path => "/tmp/hello.txt",
  state => "exists",
}
```

mgmt has many kinds of resources, such as 
[pkg](/docs/resources/#pkg) for packages, 
[exec](/docs/resources/#exec) for running other programs, 
[svc](/docs/resources/#svc) for managing service. Check out [the resources reference for more](/docs/resources/)

Can resources be remote or only local? mgmt resources do not have a concept of local or remote. While _file_ and _pkg_ will execute locally on a machine, some resources will involve remote services, such as an ec2 instance resource. You can even export resource definitions from one mgmt to be executed by mgmt on another machine.

> ⁉️Questions for the editor 
> 
> Question: Some resources are driven by external events (file w/ inotify), but others use timers. How frequently is a resource checked? How would the reader learn that? Should the reader not worry about it?

Like files, you will also likely be managing packages and services with mgmt. Here's a few examples:

## Packages

This example showcases introduces two mgmt features: autogrouping and lists.

```puppet { .m-2 }
# Ensure two different packages are installed
pkg "tmux" {
  state => "installed",
}

pkg "cowsay" {
  state => "installed",
}
```

Package management requires root, so we can invoke mgmt with sudo to run this example, after writing the above into a file "first-package.mcl":

```text { .m-2 linenos=inline }
$ sudo mgmt run lang first-package.mcl
... ( other output omitted for brevity ) ...
19:00:19 engine: autogroup: pkg[cowsay] into pkg[tmux]
19:00:19 engine: pkg[tmux]: Check: pkg[autogroup:(tmux,cowsay)]
19:00:19 engine: pkg[tmux]: Apply: pkg[autogroup:(tmux,cowsay)]
19:00:19 engine: pkg[tmux]: Set(installed): pkg[autogroup:(tmux,cowsay)]...
19:00:25 engine: pkg[tmux]: Set(installed) success: pkg[autogroup:(tmux,cowsay)]
```

This installed our packages, and mgmt did some extra optimization for us! In this case, mgmt knows that multiple packages can be installed in a single step, so it grouped both tmux and cowsay together. **mgmt calls this "autogrouping"**

You can test mgmt's reflexes by removing the `cowsay` package and watching mgmt respond by reinstalling it.

This example has one last important thing to share! You may configure many packages, often with the same params. In cases like this, you can provide a list of names in the _name_ part of the resource description:

```puppet { .m-2 }
pkg ["tmux", "cowsay"] {
  state => "installed",
}
```

Types of values like strings and lists will be covered in a later section.


### Services

To finish introducing resources, here is an example that instructs mgmt to ensure the sshd service is running and also enabled on boot.

```puppet { .m-2 }
svc "sshd" {
  startup => "enabled",
  state => "running",
}
```

Running this (as root), we can see mgmt again acting swiftly as it notices that sshd was not enabled at boot nor was it running:

```text { .m-2 }
$ sudo mgmt run lang first-service.mcl
19:08:50 engine: svc[sshd]: service enabled
19:08:50 engine: svc[sshd]: service started
```

# Resource Relationships

We've seen that resources described in _mcl_ will instruct mgmt on what to observe and what the desired state is for each resource. Your desired state may include hundreds or thousands of resources.

When configurating a system, you'll often need steps performed in a specific order. For example: installing a package, changing its configuration file, and ensuring the service is running — in that order! Additionally, if the service configuration changes, you want to notify the service of that change, right?

🌶️ Special sauce: In mgmt, **all resources are executed in concurrently**. This allows mgmt to resolve as much as possible in the shortest amount of time. There is no order unless you describe order.

If you need order, you can impose order on resources by defining relationships between resources. 

Relationships can be expressed two different ways in mcl, and both ways are equivalent. Use whichever is most convenient for you.

For the examples below, we will consider two resources and their relationship: An sshd package and sshd service.

## Relationships using arrow `->` operator

Note, these examples use Fedora Linux. Other linux distros may use different names for their equivalent package and services.

```puppet { .m-2 }
pkg "openssh-server" {
  state => "installed",
}

svc "sshd" {
  startup => "enabled",
  state => "running",
}

# Tell mgmt that the package needs to execute
# before the service
Pkg["openssh-server"] -> Svc["sshd"]
```

The syntax for arrow (`->`) relationships is: `Kind["name"] -> Kind2["name2"]`.

The _kind_ is a capitalized version of the resource name, and the string inside the `[brackets]` is the resource's name. 

## Params: Depend and Before

If it is more convenient, you may also express a relationship inside a resource definition using the params `Before` or `Depend` (capitalization is important). To express "openssh-server package must be executed before its service" we can use either of these:

```puppet { .m-2 }
pkg "openssh-server" {
  state => "installed",
  Before => Svc["sshd"],
}
```

Alternately:

```puppet { .m-2 }
svc "sshd" {
  startup => "enabled",
  state => "running",
  Depend => Pkg["openssh-server"],
}
```

Each relationship requires only one definition. That is, you can use an arrow, or one _Before_, or one _Depend_.

## Relationship Rejections

All relationships have a single direction, as in "A before B" or "B after A". 

mgmt will not allow loops in relationships, such as (A before B, B before C, C before A). A relationship loop is called a "cycle" and mgmt will report an error. Here's a small example:

```puppet { .m-2 }
file "/tmp/hello.txt" { }
file "/tmp/world.txt" { }
File["/tmp/hello.txt"] -> File["/tmp/world.txt"]
File["/tmp/world.txt"] -> File["/tmp/hello.txt"]
```

Running this, mgmt will show an error:

```text { .m-2 }
16:53:02 gapi exited with error: not a dag
resource graph has cycles
```

## not a dag? cycles?

If we visualize these resources and relationships, we can start to see a structure take shape. This structure is called a graph, and it is how mgmt represents and executes your infrastructure.

> XXX: Put a small diagram here?

A special kind of graph called a dag is used inside mgmt. A dag, or directed acyclic graph, is a math and computer science term that describes a graph (a data structure with vertexes and edges) with a requirement that edges have a direction and that a edges are not allowed to form a path through the graph that allows a loop, or cycle. 

Here's how those graph concepts map to what we've learned in mgmt:

* Vertex: A resource
* Edge: A relationship between two resources
* Direction: The arrow `->` operator and Before/Depend params


## Reusable Parts + Composition

*This section shall introduce: variables and string interpolation.*

Class + include. Reusable blocks of resources. Takes parameters (if desired).

Syntax: `class <name> { ... }`

Examples:

- Class: Reuse
- Module: maybe msyql? Module installs mysql, provides a mysql:user class?

## Programming:

So far, we’ve been describing a single desired state - in essence, the description has been static.

The mgmt language (mcl) enables dynamic decisions that change the desired state. We’ve already seen some of this in action with classes and modules (above), and you can take it further by telling mgmt how to control what resources exist, their parameters, and more.

- Variables
    - Assigning: `$name = "James"`
    - Using in as a resource name: `user  $name { ... }`
    - Using in a parameter `parameter => $varname,`
    - Using inside a string “whatever ${varname}”
    - Variable scope TBD
    - Types:
        - Brief, for now: string, number
        - mgmt enforces strict types; you can’t give a number to something expecting a string. Doing so will show an error: <Give example>
- Functions
    - Example: Formatting strings
    - Compute values, or observe a thing (like readfile)
        - Question: Can I claim that functions have no side effects?
    - Values: Can be a stream, such as the function that reads a file’s contents, and the value changes when the file content changes.
        - Question: how to know if a function streams or is single value?
- Conditionals
    - It’s not really “flow control” because it’s not imperative, it’s declarative… what term to use?
    - These are living/dynamic just like resources and functions.


----

# Out of Scope

These are all beyond "day one" success education, so I have excluded them:

* send/recv
* notification: Notify/Listen
* Meta parameters
* export/collect
* modules

# outline

- what is mgmt
    - Scope and overview
        - mgmt config is an infrastructure as code tool
        - How to install [link to getting started download docs]
        - Function/purpose: configuration management, real-time response, one machine or clusters.
    - Features of mgmt config
        - fast: mgmt will perform multiple tasks simultaneously, ensuring that your systems reach their desired state more quickly. (ordering, relationships)
        - batching: mgmt will group similar operations together in order to improve performance. mgmt calls this “auto grouping”
        - reactive: it keeps observing your infrastructure in real-time corrects for any deviations (chapters: resources, functions)
- `mcl`, the mgmt language
    - Basic syntax for resources
    - Reusable parts: classes (no parameters)
    - Making decisions
        - Functions
            - import, function syntax, where to find list of functions.
            - Functions observe their inputs and update outputs when changes are observed
            - Functions are also watched (streams)
                - Examples, os.readfilewait ? others?
        - Variables (after functions because I want to show fmt.printf and templating)
            - Setting, using, string interpolation, templating, fmt.printf
            - Types?
        - Conditionals/branching (requires functions and/or variables)
        - Iteration/loops
        - Parameterized classes (requires variables)
        - Passing data around
    - Language Features
        - Writing your own functions
        - The type system?
- (out of scope, maybe for later)
    - mgmt deploy
        - from “mgmt run lang’ to ‘mgmt deploy lang’
        - etcd: seeds and servers.
        - git vs no git
    - mgmt execution behaviors, like auto grouping, etc?
    - modules
    - clustering (etcd, etc)
    - passing data between machines
        - external lookup/streaming functions?, value+kv resources, send/recv


