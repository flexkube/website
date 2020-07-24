# Terraform provider

## Download the pre-built binary

The easiest way to get Flexkube Terraform provider is to use one of the pre-built release binaries which are available for macOS and Linux.

See [Github Releases](https://github.com/flexkube/libflexkube/releases) page for finding the latest available release.

For example, to download version `v0.3.0` on Linux, execute the following command:

```sh
VERSION=v0.3.0 wget -qO- https://github.com/flexkube/libflexkube/releases/download/$VERSION/terraform-provider-flexkube_$VERSION_linux_amd64.tar.gz | tar zxvf - terraform-provider-flexkube_$VERSION_x4
```

It will download the `terraform-provider-flexkube` binary into your current directory. If you have your Terraform code in the same directory, you can start using it right away, e.g. with `terraform init` command.

If you want the provider to be available in other directories, it is recommended to move the binary into `~/.terraform.d/plugins/` directory. This can be done using the following command:

```sh
mkdir -p ~/.terraform.d/plugins/ && mv terraform-provider-flexkube_v0.3.0_x4 ~/.terraform.d/plugins/
```

## Building from source

For building from source, make sure you have `go` and `git` binaries available in your system.

### Using `go get`

You can install Flexkube Terraform Provider from source using the following command:

```
go get github.com/flexkube/libflexkube/cmd/terraform-provider-flexkube
```

Once done, it is recommended to move the binary into `~/.terraform.d/plugins/` directory to make it available for all Terraform environments:

```sh
mkdir -p ~/.terraform.d/plugins/ && mv $(go env GOPATH)/bin/terraform-provider-flexkube ~/.terraform.d/plugins/terraform-provider-flexkube_v0.3.0_x4
```

### Using `git` and `go build`

To build Terraform provider from source, first clone [libflexkube](https://github.com/flexkube/libflexkube) repository. This can be done using the following command:

```sh
git clone https://github.com/flexkube/libflexkube.git && cd libflexkube
```

Then, to build Terraform Provider binary, run the following command:

```sh
go build ./cmd/terraform-provider-flexkube
```

Once done, it is recommended to move the binary into `~/.terraform.d/plugins/` directory to make it available for all Terraform environments:

```sh
mkdir -p ~/.terraform.d/plugins/ && mv ./terraform-provider-flexkube ~/.terraform.d/plugins/terraform-provider-flexkube
```