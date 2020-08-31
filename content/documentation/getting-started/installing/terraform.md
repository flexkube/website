# Terraform provider

## Using Terraform 0.13 and Terraform Registry

The easiest way to get Flexkube Terraform provider is to pull it from the Terraform Registry. You can do that by adding the following
snippet to `required_providers` block in `terraform` block in your module configuration:
```tf
    flexkube = {
      source  = "flexkube/flexkube"
      version = "0.4.0"
    }
```

So example `versions.tf` file would look like following:
```tf
terraform {
  required_providers {
    flexkube = {
      source  = "flexkube/flexkube"
      version = "0.4.0"
    }
  }
}
```

## Building from source

For building from source, make sure you have `go` and `git` binaries available in your system.

### Using `go get`

You can install Flexkube Terraform Provider from source using the following command:

```
go get github.com/flexkube/libflexkube/cmd/terraform-provider-flexkube
```

Once done, it is recommended to move the binary into `~/.local/share/terraform/plugins/registry.terraform.io/flexkube/flexkube/0.4.0/linux_amd64/` directory to make it available for all Terraform environments:

```sh
mkdir -p ~/.local/share/terraform/plugins/registry.terraform.io/flexkube/flexkube/0.4.0/linux_amd64 && mv $(go env GOPATH)/bin/terraform-provider-flexkube ~/.local/share/terraform/plugins/registry.terraform.io/flexkube/flexkube/0.4.0/linux_amd64/terraform-provider-flexkube
```

### Using `git` and `go build`

To build Terraform provider from source, first clone [terraform-provider-flexkube](https://github.com/flexkube/terraform-provider-flexkube) repository. This can be done using the following command:

```sh
git clone https://github.com/flexkube/terraform-provider-flexkube.git && cd terraform-provider-flexkube
```

Then, to build Terraform Provider binary, run the following command:

```sh
go build
```

Once done, it is recommended to move the binary into `~/.local/share/terraform/plugins/registry.terraform.io/flexkube/flexkube/0.4.0/linux_amd64/` directory to make it available for all Terraform environments:

```sh
mkdir -p ~/.local/share/terraform/plugins/registry.terraform.io/flexkube/flexkube/0.4.0/linux_amd64 && mv ./terraform-provider-flexkube ~/.local/share/terraform/plugins/registry.terraform.io/flexkube/flexkube/0.4.0/linux_amd64/terraform-provider-flexkube
```
