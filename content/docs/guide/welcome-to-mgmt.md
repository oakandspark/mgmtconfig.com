# Preface (for editors)


> 📚 Design of this document:
>
> - A gradual introduction with each section building on the previous.
> - Real examples demonstrating what was just covered.
> - Avoiding jargon, especially internal implementation details, such as “DAG”, unless absolutely necessary. As an example, [Elm](https://guide.elm-lang.org) does a fantastic job of explaining how to use a functional programming language *without* burdening the reader in high-level mathematical terms that are unavoidable in language docs like Haskell. It’s OK to refer to these technical terms, “in mgmt, this is called x”

> ⚠️ EDITING TODO: Review for consistent use of terminology:
> - Verbs:  a resource *executes*, right?
> - Terminology overload: Notify in mcl, Refresh internal to mgmt, and in this document, Signalling. I also use ‘signalling’ to include send/recv.
> - Should we use tabs for indenting code here? If so, we should also add css to set tab-size.

# Welcome to mgmt config

This document will guide you through your first steps learning how to use mgmt while exploring its behavior and programming syntax with real examples and real use cases.

mcl is a strongly-typed, functional language that we use to program in mgmt.

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

### Running with mgmt

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

### Continuous Convergence

In the example above, mgmt stayed running even after converging. This is because mgmt continuously watches for changes (divergence) and will respond immediately and as needed.

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

## Resource Definitions

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

Let's dive into the _pkg_ resource for managing system packages. This example also introduces two mgmt features: autogrouping and lists.

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


## Services

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

When configurating a system, you may need certain steps performed in a specific order. For example: installing a package, changing its configuration file, and ensuring the service is running — in that order! Additionally, if the service configuration changes, you want to notify the service of that change, right?

🌶️ Special sauce: In mgmt, **all resources are executed in concurrently**. This allows mgmt to resolve as much as possible in the shortest amount of time. There is no order unless you describe order.

You can impose order on resources by defining relationships. Relationships can be expressed two different ways in mcl, and both ways are equivalent. Use whichever is most convenient for you.

For the examples below, we will consider two resources and their relationship: An sshd package and sshd service. There's no service to start until the package is installed, so we need to establish an order. We can imagine them visually:

![A diagram showing two boxes with an arrow between them. Each box represents one resource](relationship-pkg-svc.svg)

## Relationships using arrow `->` operator

Note: these examples use Fedora Linux. Other linux distros may use different names for their equivalent package and services.

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

The syntax for arrow (`->`) relationships is: 

`Kind["name"] -> Kind2["name2"]`

The _kind_ is a capitalized version of the resource name, and the string inside the `[brackets]` is the resource's name. 

## Relationship Params: Depend and Before

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

Each relationship requires only one definition. That is, you can use an arrow, or one _Before_, or one _Depend_, for a single relationship.

## Relationship Rejected: Cycles

All relationships have a direction, as in "A before B" or "B after A". 

mgmt will not allow a relationship to create a loop, such as (A before B, B before C, C before A). A relationship loop is called a "cycle" and mgmt will report an error. Here's a simple example of a cycle:

```puppet { .m-2 }
file "/tmp/hello.txt" { }
file "/tmp/world.txt" { }
File["/tmp/hello.txt"] -> File["/tmp/world.txt"]
File["/tmp/world.txt"] -> File["/tmp/hello.txt"]
```

Visually, we can imagine it with two arrows (edges) in each direction between two resources:

![A diagram showing two boxes with two arrows between, each going a different direction. each box represents one resource](relationship-cycle-example.svg)

Because both files want to be "before" each other, we have a loop with no beginning or end, and mgmt will show an error:

```text { .m-2 }
16:53:02 gapi exited with error: not a dag
resource graph has cycles
```

## not a dag? cycles?

The visuals above show small examples of a structure called a graph, and it is how mgmt represents and executes your infrastructure. Graphs have vertexes and edges, and here's how those concepts map to what we've learned about mgmt, so far:

* Vertex: A resource, like a file or pkg.
* Edge: A relationship between two resources.
* Direction: The arrow `->` operator and Before/Depend params

A special kind of graph called a dag is used inside mgmt. A dag, or directed acyclic graph, is a math and computer science term that describes a graph where all edges have a direction and are not allowed to form a path through the graph that allows a loop, or cycle. 

# Programming in mgmt

This section introduces built-in functions, variables, and conditionals. We will use those features to program mgmt to handle differences between Linux distributions.

So far, we’ve been describing a single desired state - in essence, the resource graph has been static, or unchanging, throughout mgmt's life and remains the same no matter where it runs. Let's do more!

_mcl_ allows decision-making that change the resource graph. An example above even hinted at the need for this, "other linux distros may use different names" for packages and services.

For our ssh service, Fedora calls it "sshd", and Debian calls it "ssh".

To solve our problem, we will import and use a built-in function, [`os.release`](../../../docs/functions/#os.release), to determine our Linux distribution and use that information to decide which service name to use. 


```puppet { .m-2 }
# Tell mgmt to let us use 'os' functions
import "os"

# Store the os release information in the $release variable
# This information is a 'struct' type that has an "id" field
# The "id" field is a string containing an OS identifier
$release = os.release()

if $release->id == "fedora" {
  # Fedora calls this service "sshd"
  svc "sshd" {
    startup => "enabled",
    state => "running",
  }
} 

# We can handle both Debian and Ubuntu together
if $release->id == "debian" or $release->id == "ubuntu" {
  # Debian calls this service "ssh"
  svc "ssh" {
    startup => "enabled",
    state => "running",
  }
}
```

Our _mcl_ above will produce a different resource graph depending on what machine it runs on.

### Variables

We used the variable, `$release`, to store the result of `os.release()` function. Variables *immutable* meaning they may only be assigned once, and they are *block scoped*. Here are some examples that use variables:

* Assignment: `$name = "James"`
* In a resource name: `user $name { ... }`
* In a resource param: 
  ```puppet { .m-2 }
  file "/var/cache/james" {
      owner => $name,
  }
  ```
* In a string: `"Hello, ${name}"`
* In a format function: `fmt.printf("Hello, %s", $name)`

#### Formatting Text with Variables

_mcl_ is a strongly-typed language. All values have a type, and parameters are typed. You may find that if you assign a number to a variable, it can't be used where mgmt expects a string. In these cases, you'll want to use the `fmt.printf` function to format the number in a string:

```puppet { .m-2 }
import "fmt"
$port = 8000

file "/etc/nginx/nginx.conf" {
  state => "exists",
  content => fmt.printf("server {\n  listen %d;\n}", $port),
}
```

We needed `fmt.printf()` here because `$port` is a number and cannot be used _as_ a string. For example, if we set the content and try to use `${port}` in inside the content string, we'll get a type error - the following uses will cause mgmt to show an error - `content => "server {\n listen ${port};\n}"`

```text
17:44:59 error: cli parse error: could not unify types: type error: str != int
```

Only string values are allowed in `"${variable}"` string interpolation, and `$port`, above, is a number. Most params, such as resource params, function params, etc, accept only a specific type. The documentation for each resource and function will tell you what types are required for each param.

#### Scope of Variables

Variables are block scoped, meaning they are not accessible outside of the block where they are assigned. A block is the stuff between `{` and `}`. mgmt will write an error message if a variable doesn't exist.

```puppet { .m-2 }
$release = os.release()
if $release->id == "fedora" {
    $ssh_service = "sshd"
if $release->id == "debian" {
    $ssh_service = "ssh"
}

# This will be an error: 
# '$ssh_service' variable does not exist in this scope
svc $ssh_service {
    startup => "enabled",
    state => "running",
}
```

mgmt will report that the variable doesn't exist:

```text
17:54:57 cli: lang: ast: var `$ssh_service` does not exist in this scope: variable-scope-error.mcl @ 11:5-11:17

svc $ssh_service {
    ^^^^^^^^^^^^
```

### Functions

Functions are a way to perform computation and also a way observe parts of your system without making changes. Like resources, function values may change in real-time.

> ⁉️Questions for the editor 
> 
> Question: Do all functions have a 'streaming' aspect where their values can change over time? If not, how would a user know? Should they not care?

Just like *resources*, **function** results can change, in real time, reflecting observations of your machine.

The simplest example is time: mgmt's _datetime_ functions observe the clock and report the time. Ever marching forward, time functions will produce new values as the clock changes. Let's try a small example using the _print_ resource to have mgmt log a message with the current time:


```puppet { .m-2 }
import "fmt"
import "datetime"

$now = datetime.now()

print "time check" {
  msg => fmt.printf("The current time is %s", datetime.format($now, "2006-01-02 15:04:05")),
}
```

This is the first example where mgmt truly begins to shine! 

The variable `$now` stores the result of the `datetime.now()` function, and that function's result changes periodically. A new function result causes a chain reaction: A new `datetime.now()` result transitively changes the `msg` setting for our _print_ resource!

* `$now`'s value is updated when datetime.now() updates (every second, in our example)
* causing `datetime.format()` result to be re-evaluated with the new `$now` value
* causing `fmt.printf` to be re-evaluated
* causing our *Print\["time check"\]* resource have its `msg` param re-evaluated.
* when the `msg` param changes, mgmt prints a new message from this resource.

Try this example yourself using a _file_ resource instead of a _print_ resource!

```puppet { .m-2 }
file "/tmp/clock.txt" {
  msg => fmt.printf("The current time is %s\n", datetime.format($now, "2006-01-02 15:04:05")),
}
```

# Further Reading

The following are all beyond "day one" success education, so I have excluded them. This section should be structured as "Further reading" for each of these concepts:

* the type system
* classes
* send/recv
* notification: Notify/Listen
* Meta parameters
* Exported resources
* modules, import
* deploy
* etcd
