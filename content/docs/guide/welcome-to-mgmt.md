# Welcome to mgmt!

Objectives: Introduce new folks to mgmt - its purpose, etc.

> 📚 Design of this document:
>
> - A gradual introduction with each section building on the previous.
> - Examples at the end of each section demonstrating what was just covered.
> - Avoiding jargon, especially internal implementation details, such as “DAG”, unless absolutely necessary. As an example, [Elm](https://guide.elm-lang.org) does a fantastic job of explaining how to use a functional programming language *without* burdening the reader in high-level mathematical terms that are unavoidable in language docs like Haskell. It’s OK to refer to these technical terms, “in mgmt, this is called <x>”
</aside>

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
    - batching: mgmt will group similar operations together in order to improve performance. mgmt calls this “auto e
    - reactive: it observes your system in real-time and responds to correct for any deviations (chapter: functions and resources)

## Programming mgmt with mcl

mcl is the language we use to program our desired state. mcl is strongly-typed, functional language <etc etc…>

This guide will teach you how to use mcl in mgmt and how mgmt behaves.

### Resources

Configuration is described with an element called a **resource**.
Most kinds of resources in mgmt will be familiar to you and are intended to align with your mental model of your infrastructure: ensuring that a package is installed, a service is disabled, or a user exists.

Resources have three parts: a kind, a name, and params (parameters). For example, the following example code shows a single file _resource_:

```puppet { .m-2 }

file "/tmp/hello.txt" {
  content => "Greetings from mgmt!",
  state => "exists",
}
```

This is a file resource with a name “/tmp/hello.txt” and describes a state where you want the file to exist and contain exactly `Greetings from mgmt!`

You describe the desired state in each resource, and mgmt makes it real by observing the current state and making any necessary changes. An _mcl_ program can describe as many resources as you need.

Keyword: When all resources reach the desired state, **mgmt calls this outcome "converged".**

This _mcl_ code can be executed by mgmt: `mgmt run lang <path>`. mgmt will normally write its logs to stdout, so the execution of our file resource will appear like this:

```text { .m-2 linenos=inline }
$ mgmt run lang hello.mcl

... ( other output omitted for brevity ) ...

10:28:48 engine: file[/tmp/hello.txt]: copy 21 bytes
```

Notice that:
* mgmt did not exit after converging (more on this in the next section)
* mgmt can run as any user, but that user will correct permissions in order to execute some resources.
* The first part '10:28:48' is a timestamp, and yours will be different than the ones in these example.

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
10:37:18 engine: file[/tmp/hello.txt]: copy 21 bytes
```

You can terminate mgmt in your terminal by pressing Ctrl+C.

Each time, mgmt has noticed that the desired state has diverged (the file was removed, or the contents were modified) and it takes action immediately to resolve it.

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

💡 _What’s in a name?_  Many resources will use the name as a default value for a param. The file resource will use name default value for the `path` param.

⚠️Syntax Caution: Every param in a resource must have a trailing comma, including the last one.

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
[svc](/docs/resources/#svc) for managing service.

Remote or local? It depends! mgmt resources do not have a concept of local or remote. While _file_ and _pkg_ will execute locally on a machine, some resources will involve remote services, such as an ec2 instance resource.

> ⁉️Questions for the editor 
> 
> Question: Some resources are driven by external events (file w/ inotify), but others use timers. How frequently is a resource checked? How would the reader learn that? Should the reader not worry about it?

Like files, you will also likely be managing packages and services with mgmt. Here's a few examples:

##### Packages

This example showcases features not yet discussed: autogrouping and lists.

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

This installed our packages, and mgmt did some extra optimization for us! In this case, mgmt knows that multiple packages can be installed in a single step, so it grouped both tmux and cowsay together in a single step. **mgmt calls this "autogrouping"**

You can test mgmt's reflexes by removing the `cowsay` package and watching mgmt respond by reinstalling it.

This example has one last important thing to share! You are likely to be configuring many packages, often with the same params. In cases like this, you can provide a list of names in the _name_ part of the resource description:

```puppet { .m-2 }
pkg ["tmux", "cowsay"] {
  state => "installed",
}

Types of values like strings and lists will be covered in a later section.


##### Services

```puppet { .m-2 }
svc "sshd" {
  startup => "enabled",
  state => "running",
}
```

Running this (as root), we can see mgmt again acting swiftly:

```text { .m-2 linenos=inline }
$ sudo mgmt run lang first-service.mcl
19:08:50 engine: svc[sshd]: service enabled
19:08:50 engine: svc[sshd]: service started
```

## Resource Relationships

We've seen that resources described in _mcl_ will instruct mgmt on what to observe and what the desired state is for each resource. Your desired state may include hundreds or thousands of resources.

When configurating a system, you'll often need steps performed in a specific order. For example: installing a package, changing its configuration file, and ensuring the service is running — in that order! Additionally, if the config file ever changes, you want to notify the service of that change, right?

🌶️ Special sauce: In mgmt, **all resources are executed in concurrently**. This allows mgmt to resolve as much as possible in the shortest amount of time. There is no order unless you describe order.

If you need order, you can impose order on resources by defining relationships between resources.

- Relationships are how mgmt knows about ordering and signaling.
    - Ordering: What resources need to be executed(?) before another?
    - Signalling: When one resource changes, should a linked resource take any specific action? (Internally this is called ‘Refresh’)
    - (out of scope for this chapter) Sending values: When a resource changes, it can produce data that is useful when sent to another resource.
    - Question: How to describe send/recv?
- Question: Resources can produce events (signals and values?) How does a user learn what resoures support notification? (’print’ does not Notify, for example!)
- Defining relationships:
    - Syntax: Kind[name] → Kind2[name2]
        - The arrow → tells mgmt that the left side should be resolved before the right side.
    - Syntax for params:
        - signalling relationships: Notify, Listen
        - order relationship: Depend, Before
- Question: how to know what Notify does to a resource? (Is this documented?)

Visualizing these relationships, we can start to see

examples…

> ⚠️ send+recv is out of scope for this early in the document, might be best reserved for advanced?

## Reusable Parts + Composition

*This section shall introduce: variables and string interpolation.*

Class + include. Reusable blocks of resources. Takes parameters (if desired).

Syntax: `class <name> { ... }`

Examples:

- Class: Reuse
- Module: maybe msyql? Module installs mysql, provides a mysql:user class?

## Behavior Changes - Metaparameters

<aside>
⚠️

Out of scope? Maybe limit it to just the sometimes-useful ones, like retry and delay?

Hidden? Export?

[Metaparameters](https://github.com/purpleidea/mgmt/blob/ba92a9212f3a1b5d4726c824191e3031d3265aab/engine/metaparams.go#L72)

</aside>

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


