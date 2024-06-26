summary: Getting started with Go
description: Building and testing with Go and Please, as well as managing third-party dependencies via go_repo
id: go_intro
categories: beginner
tags: medium
status: Published
authors: Jon Poole
Feedback Link: https://github.com/thought-machine/please

# Getting started with Go
## Overview
Duration: 4

### Prerequisites
- You must have Please installed: [Install please](https://please.build/quickstart.html)
- Go must be installed: [Install Go](https://golang.org/doc/install#install)

### What you'll learn
- Configuring Please for Go using the Go plugin
- Creating an executable Go binary
- Adding Go packages to your project
- Testing your code
- Including third-party libraries

### What if I get stuck?

The final result of running through this codelab can be found
[here](https://github.com/thought-machine/please-codelabs/tree/main/getting_started_go) for reference. If you really get
stuck you can find us on [gitter](https://gitter.im/please-build/Lobby)!

## Initialising your project
Duration: 2

The easiest way to get started is from an existing Go module:

```text
$ mkdir getting_started_go && cd getting_started_go
$ plz init 
$ plz init plugin go
$ go mod init github.com/example/module 
```


### So what just happened?
You will see this has created a number of files in your working folder:
```text
$ tree -a
  .
  ├── go.mod
  ├── pleasew
  ├── plugins
  │   └── BUILD
  └── .plzconfig
```

The `go.mod` file was generated by `go` and contains information about the Go module. While Please doesn't directly use
this file, it can be useful for integrating your project with the Go ecosystem and IDEs. You may remove it if you wish.

The `pleasew` script is a wrapper script that will automatically install Please if it's not already! This
means Please projects are portable and can always be built via
`git clone https://... example_module && cd example_module && ./pleasew build`.

The `plugins/BUILD` is a file generated by `plz init plugin go` which defines a build target for the Go plugin.

The file `.plzconfig` contains the project configuration for Please. Please will have initialised this with the Go 
plugin configuration for us:

### `.plzconfig`
```
[parse]
preloadsubincludes = ///go//build_defs:go ; Makes the Go rules available automatically in BUILD files

[Plugin "go"]
Target = //plugins:go
```

This configures the Go plugin, and makes the build definitions available in the parse context throughout the repo
automatically. Alternatively, if you're not using Go everywhere, you can remove the `preloadsubincludes` config and add 
`subinclude("///go//build_defs:go")` to each `BUILD` file that needs access to Go rules.

Read the [config](/config.html) and [go plugin config](/plugins.html#go.config) docs for more information on 
configuration.

Finally, the `plz-out` directory contains artifacts built by plz.

## Setting up our import path
Duration: 1

As we've initialised a Go module, all imports should be resolved relative to the module name. To instruct Please to
use this import path, we have to configure the Go plugin as such:

### `.plzconfig`
```text
[Plugin "go"]
Target = //plugins:go
ImportPath = github.com/example/module ; Should match the module name in go.mod
```

## Setting up your toolchain
Duration: 2

If you have followed the [Golang quickstart guide](https://go.dev/doc/tutorial/getting-started), or if you're using 
1.20 or newer, there's a good chance additional configuration is required. There are two options for configuring your
Go toolchain with Please. 

### Recommended: managed toolchain

The simplest way is to let Please manage your toolchain for you. The `go_toolchain()` rule will download the Go 
toolchain, compiling the standard library if necessary. Simply add the following rule to your project:

### `third_party/go/BUILD`
```python
go_toolchain(
    name = "toolchain",
    version = "1.20",
)
```

And then configure the Go plugin to use it like so:
### `.plzconfig`
```text
[Plugin "go"]
Target = //plugins:go
ImportPath = github.com/example/module
GoTool = //third_party/go:toolchain|go
```

### Using Go from the system PATH

By default, Please will look for Go in the following locations:
```
/usr/local/bin:/usr/bin:/bin
```

If you have Please installed elsewhere, you must configure the path like so:

### `.plzconfig`
```text
[Build]
Path = /usr/local/go/bin:/usr/local/bin:/usr/bin:/bin
```

Additionally, from version 1.20, golang no longer includes the standard library with its distribution. To use 1.20 from
the path with Please, you must install it. This can be done like so: 

```text
$ GODEBUG="installgoroot=all" go install std
```

## Hello, world!
Duration: 4

Now we have a Please project, it's time to start adding some code to it! Let's create a "hello world" Go program:

### `src/main.go`
```go
package main

import "fmt"

func main(){
	fmt.Println("Hello, world!")
}
```

We now need to tell Please about our Go code. Please projects define metadata about the targets that are available to be
built in `BUILD` files. Let's create a `BUILD` file to build this program:

### `src/BUILD`
```python
go_binary(
  name = "main",
  srcs = ["main.go"],
)
```

That's it! You can now run this with:
```text
$ plz run //src:main
Hello, world!
```

There's a lot going on here; first off, `go_binary()` is one of the [go plugin functions](/plugins.html#go). This build
function creates a "build target" in the `src` package. A package, in the Please sense, is any directory that contains a
`BUILD` file.

Each build target can be identified by a build label in the format `//path/to/package:label`, i.e. `//src:main`.
There are a number of things you can do with a build target such as `plz build //src:main`, however, as you've seen,
if the target is a binary, you may run it with `plz run`.

## Adding packages
Duration: 5

Let's add a `src/greetings` package to our Go project:

### `src/greetings/greetings.go`
```go
package greetings

import (
    "math/rand"
)

var greetings = []string{
    "Hello",
    "Bonjour",
    "Marhabaan",
}

func Greeting() string {
  return greetings[rand.Intn(len(greetings))]
}
```

We then need to tell Please how to compile this library:

### `src/greetings/BUILD`
```python
go_library(
    name = "greetings",
    srcs = ["greetings.go"],
    visibility = ["//src/..."],
)
```

We can then build it like so:

```text
$ plz build //src/greetings
Build finished; total time 290ms, incrementality 50.0%. Outputs:
//src/greetings:greetings:
  plz-out/gen/src/greetings/greetings.a
```

Here we can see that the output of a `go_library` rule is a `.a` file which is stored in
`plz-out/gen/src/greetings/greetings.a`. This is a [static library archive](https://en.wikipedia.org/wiki/Static_library)
representing the compiled output of our package.

We have also provided a `visibility` list to this rule. This is used to control where this `go_library()` rule can be
used within our project. In this case, any rule under `src`, denoted by the `...` syntax.

NB: This syntax can also be used on the command line e.g. `plz build //src/...`

## Using our new package
Duration: 2
To maintain a principled model for incremental and hermetic builds, Please requires that rules are explicit about their
inputs and outputs. To use this new package in our "hello world" program, we have to add it as a dependency:

### `src/BUILD`
```python
go_binary(
    name = "main",
    srcs = ["main.go"],
    # NB: if the package and rule name are the same, you may omit the name i.e. this could be just //src/greetings
    deps = ["//src/greetings:greetings"],
)
```

You can see we use a build label to refer to another rule here. Please will make sure that this rule is built before
making its outputs available to our rule here.

Then update `src/main.go`:
### `src/main.go`
```go
package main

import (
    "fmt"

    "github.com/example/module/src/greetings"
)

func main(){
    fmt.Printf("%s, world!\n", greetings.Greeting())
}
```

Give it a whirl:

```text
$ plz run //src:main
Bonjour, world!
```

## Testing our code
Duration: 5

Let's create a very simple test for our library:
### `src/greetings/greetings_test.go`
```go
package greetings

import "testing"

func TestGreeting(t *testing.T) {
    if Greeting() == "" {
        panic("Greeting failed to produce a result")
    }
}
```

We then need to tell Please about our tests:
### `src/greetings/BUILD`
```python
go_library(
    name = "greetings",
    srcs = ["greetings.go"],
    visibility = ["//src/..."],
)

go_test(
    name = "greetings_test",
    srcs = ["greetings_test.go"],
    # Here we have used the shorthand `:greetings` label format. This format can be used to refer to a rule in the same
    # package and is shorthand for `//src/greetings:greetings`.
    deps = [":greetings"],
)
```

We've used `go_test()`. This is a special build rule that is considered a test. These rules can be executed as such:
```text
$ plz test //src/...
//src/greetings:greetings_test 1 test run in 3ms; 1 passed
1 test target and 1 test run in 3ms; 1 passed. Total time 90ms.
```

Please will run all the tests it finds under `//src/...`, and aggregate the results up. This works even across
languages allowing you to test your whole project with a single command.

### External tests

Go has a concept of "external" tests. This means that tests can exist in the same folder as the production code, but
they have a different package. Please supports this through the `external = True` argument on `go_test()`:

### `src/greetings/greetings_test.go`
```go
package greetings_test

import (
    "testing"
    
    // We now need to import the "production" package 
    "github.com/example/module/src/greetings"
)

func TestGreeting(t *testing.T) {
    if greetings.Greeting() == "" {
        panic("Greeting failed to produce a result")
    }
}
```

### `src/greetings/BUILD`
```python
go_library(
    name = "greetings",
    srcs = ["greetings.go"],
    visibility = ["//src/..."],
)

go_test(
    name = "greetings_test",
    srcs = ["greetings_test.go"],
    deps = [":greetings"],
    external = True,
)
```

Check if it works:
```text
$ plz test //src/...
//src/greetings:greetings_test 1 test run in 3ms; 1 passed
  1 test target and 1 test run in 3ms; 1 passed. Total time 90ms.
```
## Third-party dependencies
Duration: 7

To add third party dependencies to Please, the easiest way is to use `///go//tools:please_go` to resolve them, and then
add them to `third_party/go/BUILD`. Let's add `github.com/stretchr/testify`:

```text
$ plz run ///go//tools:please_go -- get github.com/stretchr/testify@v1.8.2
go_repo(module="github.com/stretchr/objx", version="v0.5.0")
go_repo(module="gopkg.in/yaml.v3", version="v3.0.1")
go_repo(module="gopkg.in/check.v1", version="v0.0.0-20161208181325-20d25e280405")
go_repo(module="github.com/stretchr/testify", version="v1.8.2")
go_repo(module="github.com/davecgh/go-spew", version="v1.1.1")
go_repo(module="github.com/pmezard/go-difflib", version="v1.0.0")
```

We can then add them to `third_party/go/BUILD`:
```python
# We give direct modules a name and install list so we can reference them nicely
go_repo(
    name = "testify",
    module = "github.com/stretchr/testify", 
    version="v1.8.2",
    # We add the subset of packages we actually depend on here
    install = [
        "assert",
        "require",
    ]
)

# Indirect modules are referenced internally, so we don't have to name them if we don't want to. They can still be 
# referenced by the following build label naming convention: ///third_party/go/github.com_owner_repo//package.
#
# NB: Any slashes in the module name will be replaced by _ 
go_repo(module="github.com/davecgh/go-spew", version="v1.1.1")
go_repo(module="github.com/pmezard/go-difflib", version="v1.0.0")
go_repo(module="github.com/stretchr/objx", version="v0.5.0")
go_repo(module="gopkg.in/yaml.v3", version="v3.0.1")
go_repo(module="gopkg.in/check.v1", version="v0.0.0-20161208181325-20d25e280405")
```

More information as to how `go_repo` works can be found 
[here](/plugins.html#go_repo).

NB: This build label looks a little different. That's because it's referencing a build target in a subrepo. 
### Updating our tests

We can now use this library in our tests:

### `src/greetings/greetings_test.go`
```go
package greetings_test

import (
    "testing"

    "github.com/stretchr/testify/assert"

    "github.com/example/module/src/greetings"
)

func TestGreeting(t *testing.T) {
    assert.NotEqual(t, greetings.Greeting(), "")
}
```

### `src/greetings/BUILD`
```python
go_library(
    name = "greetings",
    srcs = ["greetings.go"],
    visibility = ["//src/..."],
)

go_test(
    name = "greetings_test",
    srcs = ["greetings_test.go"],
    deps = [
        ":greetings",
        # Could use a subrepo label i.e. ///third_party/go/github.com_stretchr_testify//assert instead if we want
        "//third_party/go:testify",
    ],
    external = True,
)
```

And then we can check it all works:
```text
$ plz test
//src/greetings:greetings_test 1 test run in 3ms; 1 passed
1 test target and 1 test run; 1 passed.
Total time: 480ms real, 0s compute.
```

## What next?
Duration: 1

Hopefully you now have an idea as to how to build Go with Please. Please is capable of so much more though!

- [Please basics](/basics.html) - A more general introduction to Please. It covers a lot of what we have in this
tutorial in more detail.
- [Go plugin rules](/plugins.html#go) - See the rest of the Go plugin rules and config.
- [Built-in rules](/lexicon.html#go) - See the rest of the built in rules.
- [Config](/config.html) - See the available config options for Please.
- [Command line interface](/commands.html) - Please has a powerful command line interface. Interrogate the build graph,
determine files changes since master, watch rules and build them automatically as things change and much more! Use
`plz help`, and explore this rich set of commands!

Otherwise, why not try one of the other codelabs!
