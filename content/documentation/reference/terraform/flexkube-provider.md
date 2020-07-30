# Flexkube Provider

The Flexkube (flexkube) provider is used to interact with the resources supported by [libflexkube](https://github.com/flexkube/libflexkube). The provider itself do not require any configuration.

Use the navigation to the left to read about the available resources.

Please note, that while the Terraform provider provides very nice UI for automation, it has currently some bugs ([#48](https://github.com/flexkube/libflexkube/issues/48), [#85](https://github.com/flexkube/libflexkube/issues/85)), which might be annoying in some scenarios.



## Example Usage

```tf
provider "flexkube" {}
```

