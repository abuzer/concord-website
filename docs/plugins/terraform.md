---
layout: wmt/docs
title:  Terraform Task
side-navigation: wmt/docs-navigation.html
---

# {{ page.title }}

Concord supports interaction with the infrastructure provisioning tool
[Terraform](https://www.terraform.io/) with the `terraform` task as part of any
flow.

- [Usage](#usage)
- [Common Parameters](#common-parameters)
- [Planning the Changes](#planning)
- [Applying the Changes](#applying)
- [Input Variables](#variables)
- [Environment Variables](#env)
- [Output Variables](#output)
- [State Backends](#backends)
- [GIT modules](#git-modules)
- [Examples](#examples)

## Usage

To be able to use the task in a Concord flow, it must be added as a
[dependency](../getting-started/concord-dsl.html#dependencies):

```yaml
configuration:
  dependencies:
  - mvn://com.walmartlabs.concord.plugins:terraform-task:{{ site.concord_plugins_version }}
```

This adds the task to the classpath and allows you to invoke it in any flow.

The task requires the process to run as a part of a Concord project.

<a name="common-parameters"></a>

## Common Parameters

- `action` - (mandatory) action to perform:
  - `plan` - [plan](#planning) the changes;
  - `apply` - [apply](#applying) the changes with or without using a previously
  created plan file;
  - `output` - save the [output variables](#output);
- `backend` - type of a [state backend](#backends) to use:
  - `concord` - (default) use the backend provided by Concord;
  - `none` - use the default file-based backend or the backend configuration
  provided by the user;
- `debug` - boolean value, if `true` the plugin logs additional debug information;
- `extraEnv` - key-value pairs, extra environment variables provided to
the `terraform` process;
- `extraVars` - [variables](#variables) provided to the `terraform` process;
- `ignoreErrors` - boolean value, if `true` any errors that occur during the
execution will be ignored and stored in the `result` variable;
- `stateId` - string value, the name of a state file to use. See
the [State Backends](#backends) section for more details.

<a name="planning"/>

## Planning the Changes

The `plan` action executes `terraform plan` in the process' working directory
or in a directory specified in `dir` parameter:

```yaml
# run `terraform plan` in `${workDir}`
- task: terraform
  in:
    action: plan

# run `terraform plan` to generate a destroy plan
- task: terraform
  in:
    action: plan
    destroy: true

# run `terraform plan` in a specific directory
- task: terraform
  in:
    action: plan
    dir: "myTFStuff"
```

The plugin automatically creates the necessary [backend](#backends)
configuration and runs `terraform init` when necessary.

Parameters:

- `dir` - string value, path to a directory with `*.tf` files. The path must be
  relative to the process' `${workDir}`;
- `plan` - string value, path to a previosly created plan file. The path must
be relative to the process' `${workDir}`;
- `destroy` - boolean value, if true destroy plan is generated. By default,
  apply plan is generated;
- `gitSsh` - see [GIT modules](#git-modules).

The output is stored in a `result` variable that has the following structure:

- `ok` - boolean value, `true` if the execution is successful;
- `hasChanges` - boolean value, `true` if `terraform plan` detected any changes
in the enviroment;
- `output` - string value, output of `terraform plan` (stdout);
- `planPath` - string value, path to the created plan file. The plugin stored
such files as process attachments so they \"survive\" suspending/resuming the
process or restoring from a
[checkpoint](../getting-started/concord-dsl.html#checkpoints). The path is
relative to the process' `${workDir}`;
- `error` - string value, error of the last `terraform` execution (stderr).

The execution's output (stored as `${result.output}`) can be used to output
the plan into the process' log, used in an approval form, Slack notification,
etc.

<a name="applying"/>

## Applying the Changes

The `apply` action executes `terraform apply` in the process' working
directory, in a directory specified in `dir` parameter or using a
previously created `plan` file:

Run `terraform apply` in `${workDir}`:

```yaml
- task: terraform
  in:
    action: apply
```

Run `terraform apply` in a specific directory

```yaml
- task: terraform
  in:
    action: apply
    dir: "myTFStuff"
```

Run `terraform apply` using a plan file

```yaml
- task: terraform
  in:
    action: apply
    dir: "myTFStuff"
    plan: "${result.planPath}" # created by previously executed `plan` action
```

As with the `plan` action, the plugin automatically runs `terraform init` when necessary.

Parameters:

- `dir` - string value, path to a directory with `*.tf` files. The path must be
relative to the process' `${workDir}`;
- `plan` - string value, path to a previosly created plan file. The path must
be relative to the process' `${workDir}`. When using `plan`, the original `dir`
must be specified as well;
- `gitSsh` - see [GIT modules](#git-modules).
- `saveOutput` - boolean value, if `true` the `terraform output` command will
be automatically executed after the `apply` is completed and the result will
be saved in the `result` variable.

The action's output is stored in a `result` variable that has the following
structure:

- `ok` - boolean value, `true` if the execution is successful;
- `output` - string value, output of `terraform apply` (stdout);
- `error` - string value, error of the last `terraform` execution (stderr);
- `data` - map (dict) value, contains the output values. Only if `saveOutput` is `true`.

<a name="variables"/>

## Input Variables

[Input variables](https://www.terraform.io/docs/configuration/variables.html)
can be specified using the `extraVars` parameter:

```yaml
- task: terraform
  in:
    action: plan
    extraVars:
      aVar: "someValue"
      nestedVar:
        x: 123
```

The `extraVars` parameter expects regular `java.util.Map<String, Object>`
objects and supports all JSON-compatible data structures (nested objects,
lists, etc).

Specifying `extraVars` is an equivalent of running `terraform [action] -var-file=/path/to/file.json`.

<a name="env"/>

## Environment Variables

OS-level [environment variables](https://www.terraform.io/docs/commands/environment-variables.html)
can be specified using `extraEnv` parameter:

```yaml
- task: terraform
  in:
    action: plan
    extraEnv:
      HTTPS_PROXY: http://proxy.example.com
      TF_LOG: TRACE
```

<a name="output"/>

## Output Variables

There are two ways how [output](https://www.terraform.io/docs/configuration/outputs.html)
values can be saved - using the `output` action or by adding `saveOutput` to
the `apply` action parameters:

```yaml
- task: terraform
  in:
    action: output
    dir: "myTFStuff" # optional path to *.tf files

# all output values will be saved as a ${result.data} variable
- log: "${result.data}" 
```

```yaml
- task: terraform
  in:
    action: apply
    saveOutput: true
    # the rest of the parameters are the same as with the regular `apply`

- log: "${result.data}"
```

<a name="backends"/>

## State Backends

Concord uses its own [state
backend](https://www.terraform.io/docs/backends/index.html) by default, but any
of the standard Terraform state backends can be configured for use.

When using the Concord state backend, on by default, the state is stored in
Concord's internal [inventory system](../api/inventory.html) using the
`tfState-${projectName}_${repositoryName}` template for the name. If you want to
use a custom name for storing the state, the name can be overridden using the
`stateId` parameter:

```yaml
- task: terraform
  in:
    action: plan
    stateId: "myInventory"
```

To completely remove the state, you can use the inventory [API](../api/inventory.html#delete-inventory). 

Concord also supports the use of all [Terraform's backend types](https://www.terraform.io/docs/backends/types/index.html). To instruct the plugin to use the `s3` backend for storing state, use something like the following:

```yaml
- task: terraform
  in:
    backend:
      s3:
        bucket: "tfstate"
        key: "project"
        region: "us-west-2"
        encrypt: true
        dynamodb_table: "project-lock"
```

You can also disable the use of all state backends by specifying a backend of
`none`. This is effectively the same as the default command line behavior that
uses the filesystem to store state in the `terraform.tfstate`. To instruct the
plugin to use no backend for storing state, use something like the following:

```yaml
- task: terraform
  in:
    action: plan
    backend: none
```

<a name="git-modules"/>

## GIT Modules

Using [Generic GIT
repositories](https://www.terraform.io/docs/modules/sources.html#generic-git-repository)
as modules may require SSH key authentication. The plugin provides a couple ways
to configure the keys.

Using private key files directly:

```yaml
- task: terraform
  in:
    gitSsh:
      privateKeys:
        - "relative/path/to/a/private/key.file"
        - "another/private/key.file"
```

The path must be relative to the process `${workDir}`.

An alternative (and recommended) way is to use Concord [Secrets](../api/secret.html):

```yaml
- task: terraform
  in:
    gitSsh:
      secrets:
        - org: "myOrg" # optional
          secretName: "myKeyPairSecret"
          password: "myS3cr3t" # optional
```

Multiple private key files and Concord Secrets can be used simultaneously.

When running separate `plan` and `apply` actions, only the `plan` part requires
the key configuration.

## Examples

- [minimal AWS example](https://github.com/walmartlabs/concord-plugins/tree/master/tasks/terraform/examples/minimal)
- [minimal Azure example](https://github.com/walmartlabs/concord-plugins/tree/master/tasks/terraform/examples/azure-minimal)
- [approval](https://github.com/walmartlabs/concord-plugins/tree/master/tasks/terraform/examples/approval) - runs `plan`
and `apply` actions separately, uses an approval form to gate the changes;
- [output values](https://github.com/walmartlabs/concord-plugins/tree/master/tasks/terraform/examples/output) - how
to save `output` values and use them in the process.
