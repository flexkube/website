## Flexkube CLI

## Download the pre-built binary

The easiest way to get Flexkube CLI	 is to use one of the pre-built release binaries which are available for macOS and Linux.

See [Github Releases](https://github.com/flexkube/libflexkube/releases) page to find the latest available release.

For example, to download version `v0.4.0` on Linux, execute the following command:

```sh
VERSION=v0.4.0
```

It will download the `flexkube` binary into your current directory. It is recommende to move this binary into one of directories mentioned in your `$PATH` environment variable, e.g. to `~/.local/bin` or `/usr/local/bin`, to make it easy to access.

## Building from source

For building from source, make sure you have `go` and `git` binaries available in your system.

### Using `go get`

You can install Flexkube CLI from source using the following command:

```
go get github.com/flexkube/libflexkube/cmd/flexkube
```

Once done, make sure your Go binary path is included in `$PATH`, so the binary is accessible for execution.

### Using `git` and `go build`

To build Flexkube CLI from source, first clone [libflexkube](https://github.com/flexkube/libflexkube) repository. This can be done using the following command:

```sh
git clone https://github.com/flexkube/libflexkube.git && cd libflexkube
```

Then, to build the binary, run the following command:

```sh
go build ./cmd/flexkube
```

When build is finished, the binary should be in the current directory. It is recommende to move this binary into one of directories mentioned in your `$PATH` environment variable, e.g. to `~/.local/bin` or `/usr/local/bin`, to make it easy to access.
