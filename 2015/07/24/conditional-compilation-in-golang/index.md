# Conditional compilation in golang

- Author: Kunal Kushwaha (https://kunalkushwaha.github.io/about/)
- Published: 2015-07-24
- Tags: Go
- Canonical: https://kunalkushwaha.github.io/2015/07/24/conditional-compilation-in-golang/

## Conditional compiling in golang.

While hacking around experimental builds of docker, I learned about conditional compiling in golang.
As any project grew or ported to multiple architecture/platforms the need of conditional compiling is obvious.

Unlike C golang, do not have macros. Golang make use of build tags to achieve this.

#### Build Tags:

Build tags are basically annotation in source code. generally it is first line of file, where build tags are defined.

`// +build <tags>`

These tags are searched by `go build` tool before passing to go compiler. These tags can be passed while go build command like

`$ go build -tags <tags>`

Here are few points to remember regarding tags.

- Tags specifies architecture and platforms are special. i.e. No need to pass such tags, `go build` is smart enough to pass them.
  - e.g. linux, windows, darwin, 386, arch etc
- Tags can be passed with negation i.e. !
- A blank line must be present between build tags and code, else that line will be ignored.
- OR condition for tags can be specified with space.
- AND condition can be specified with `,`

If some tag is defined in file and is not passed in `go build`` command, it will be ignored.

Lets see this with example with custom tag.

- Here we will take three files

1. main.go
2. include.go
3. exclude.go

```go
○ cat main.go
package main

import (
"fmt"
)

func init() {
        fmt.Println("Hello from Main init()")
}

func main() {
        fmt.Println("Hello from main")
}
```

```go

○ cat include.go

package main

import "fmt"

func init() {
        fmt.Println("Hello from IncludeInit()")
}
```

```go
○ cat exclude.go
// +build exclude

package main

import "fmt"

func init(){
        fmt.Println("Hello from exclude")
}
```

As you can see, here exclude file has build tag of `exclude`.

Now if do `go build`, It will not compile exclude.go.

```
○ go build

○ ./temp
Hello from IncludeInit()
Hello from Main init()
Hello from main
```

Now let try using buildtag exclude.

```
○ go build -tags exclude

○ ./temp
Hello from exclude
Hello from IncludeInit()
Hello from Main init()
Hello from main
```

You can see now, the exclude.go is part of binary.

One more interesting thing with golang is `go list` tool.
Using the `go list`, without compiling, you can see which files will be compiled, with given gotags or default.

I found `go list --help` not great, but the summary of this tool is a go template can be used to check the list of files that will be included in the build.
A simple example as below will help you to understand better.

```
○ go list -f '{{.GoFiles}}' --tags exclude
[exclude.go include.go main.go]

○ go list -f '{{.GoFiles}}'
[include.go main.go]
```

As you can see, if exclude tag is not used, only include.go and main.go files are used.

#### So it that all about conditional compiling in golang?

No, There is even simpler method available too :-) and it is file-name suffixes.

In golang, there are few go environment variables defined.
GOOS and GOARCH are two of such.

- GOOS defines the current OS
- GOARCH defines the current machine architecture.

`go build` matches value of these two environment variables in filename and if it matches then only include for compiling.

So by naming files like `exclude_linux.go`, it will be included only if compiled in linux system.
`exclude_linux_amd64.go` will ensure, it will be inlcuded only if OS is linux and architecture of system in 64 bit.

You may try these combination.

#### Which one should be used?

- For custom tags build tags are the only options.
- For platform and architecture specific, if you plan to use cross build on one systems, build tags will be preferable.
- Also, if same file does work for multiple platforms, build tags will be better choice.

For rest, file sufixes can be used.

Hope this info will be useful.
