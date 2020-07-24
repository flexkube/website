# Go module

Recommended way of using Flexkube in your Go project is via [libflexkube](https://github.com/flexkube/libflexkube) library. `libflexkube` uses [Go modules](https://blog.golang.org/using-go-modules) to manage it's dependencies, so it is also recommended for your project to use it.

To add `libflexkube` module to your project, simply run the following command:

```sh
go get github.com/flexkube/libflexkube
```

It will import latest release of `libflexkube` into your project.

With module added, go to [Go examples]({{< relref "/documentation/examples/go" >}}) to see how to use it in your code or see [reference documentation]({{< relref "/documentation/reference/go" >}}) to see all available packages.