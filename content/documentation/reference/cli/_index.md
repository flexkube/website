# Flexkube CLI (flexkube)

This section includes the reference documentation for the Flexkube CLI (`flexkube`), it's subcommands and flags and configuration syntax and options.

To learn about available subcommands, see [Commands]({{< relref "/documentation/reference/cli/Commands" >}}) section.

All commands consume `config.yaml` file from the current directory and may produce `state.yaml` file, which will also be used on next runs.

## Configuration

`config.yaml` file contains configuration for all subcommands. It's syntax is described in [Configuration]({{< relref "/documentation/reference/cli/configuration" >}}) section.

## State

`state.yaml` file contains information about created resources, like container IDs, generated certificates etc. It is important to keep this file if you want to be able to update or remove your resources using `flexkube` CLI.

You can examine this file to find how how configuration options got transpiled into containers flags, environment variables and configuration files.
