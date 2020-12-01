---
title: "[Beta] Source operations"
weight: 13
sidebarTitle: "Source operations"
tier:
  - Elite
  - Enterprise
---

An application can define a number of operations that apply to its source code and that can be automated.

{{< note >}}
Source Operations are currently in Beta.  While the syntax is not expected to change, some behavior might in the future.
{{< /note >}}

A basic, common source operation could be to automatically update dependencies. For instance, with composer:

```yaml
source:
    operations:
        update:
            command: |
                set -e
                composer update
                git add composer.lock
                git commit -m "Update Composer dependencies."
```

The `update` key is the name of the operation. It is arbitrary, and multiple source operations can be defined.

(You may wish to include more robust error handling than this example.)

The environment resource gets a new `source-operation` action which can be triggered by the CLI:

```bash
platform source-operation:run update
```

The `source-operation:run` command takes the command name to run. Additional variables can be added to inject into the environment of the source operation.  They will be interpreted the same way as any other [variable](/development/variables.md) set through the UI or CLI, which means you need an `env:` prefix to expose them as a Unix environment variable.  They can then be referenced by the source operation like any other variable.

```bash
platform source-operation:run update --variable env:FOO=bar --variable env:BAZ=beep
```

When this operation is triggered:

* A clean Git checkout of the current environment HEAD commit is created; this checkout doesn't have any remotes, has all the tags defined in the project, but only has the current environment branch.
* Sequentially, for each application that has defined this operation, the operation command is launched in the container image of the application.  The environment will have all of the variables available during the build phase, optionally overridden by the variables specified in the operation payload.
* At the end of the process, if any commits were created, the new commits are pushed to the repository and the normal build process of the environment is triggered.

Note that these operations run in an isolated container: it is not part of the runtime cluster of the environment, and doesn't require the environment to be running.  Also be aware that if multiple applications in a single project both result in a new commit, that will appear as two distinct commits in the Git history but only a single new build/deploy cycle will occur.

## Source Operations with an external Git integration

Git integration can be configured to send commits made to the Platform.sh Git remote, to the upstream repository instead. This means that if a source operation did generate a new commit, the commit will be pushed to the upstream repository.

{{< note >}}
Currently, this configuration requires the `enable_codesource_integration_push` setting to be turned on by a Platform.sh staff and is only available to selected Beta customers.
{{< /note >}}

Source Operations can only be triggered on environment created by a branch, and not to environment created by a Pull Request on the external upstream (GitHub, Bitbucket, Gitlab). If you trigger a source operation on an environment created by a Pull Request on the external upstream, you will receive the following error:
`[ApiFeatureMissingException] This project does not support source operations.`.

## Automated Source Operations using cron

You can use cron to automatically run your source operations.

{{< note >}}
Automated source operations using cron requires to [get an API token and install the CLI in your application container](/development/cli/api-tokens.md).
{{< /note >}}

Once the CLI is installed in your application container and an API token has been configured, you can add a cron task to run your source operations once a day. We do not recommend triggering source operations on your `master` production environment, but rather on a dedicated environment which you can use for testing before deployment.

The example below synchronizes the `update-dependencies` environment with its parent before running the `update` source operation:

```yaml
crons:
    update:
        # Run the 'update' source operation every day at midnight.
        spec: '0 0 * * *'
        cmd: |
            set -e
            if [ "$PLATFORM_BRANCH" = update-dependencies ]; then
                platform environment:sync code data --no-wait --yes
                platform source-operation:run update --no-wait --yes
            fi
```
