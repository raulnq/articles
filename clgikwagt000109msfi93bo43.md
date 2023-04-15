---
title: "Running Nuke inside a GitHub Action"
datePublished: Sat Apr 15 2023 22:57:33 GMT+0000 (Coordinated Universal Time)
cuid: clgikwagt000109msfi93bo43
slug: running-nuke-inside-a-github-action
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1681523851878/dc0677ea-6f10-4a95-8798-bde055f01561.png
tags: github, net, cicd, github-actions-1, nuke

---

This post is the official continuation of the post, [Nuke: Deploy ASP. NET Web App to Azure](https://blog.raulnq.com/nuke-deploy-asp-net-web-app-to-azure) (try to check it before and download the initial code from [here](https://github.com/raulnq/nuke-sandbox/tree/original)). Here we'll see how easy it's to run [Nuke](https://nuke.build/docs/introduction/) inside a GitHub Action. We'll create two workflows, the first to compile the application on every push and the other to deploy the application on demand. We can generate the workflow files by adding the `GitHubActions` attribute at the top of the `Build` class:

```csharp
[GitHubActions(
    "compile",
    GitHubActionsImage.UbuntuLatest,
    OnPushBranches = new[] { "main" },
    InvokedTargets = new[] { nameof(Compile) })]
```

Let's run the `nuke --help` command to generate the `compile.yml` workflow file(actually, every execution of the `nuke` command will generate the file):

```yaml
name: compile

on:
  push:
    branches:
      - main

jobs:
  ubuntu-latest:
    name: ubuntu-latest
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: Cache .nuke/temp, ~/.nuget/packages
        uses: actions/cache@v2
        with:
          path: |
            .nuke/temp
            ~/.nuget/packages
          key: ${{ runner.os }}-${{ hashFiles('**/global.json', '**/*.csproj') }}
      - name: Run './build.cmd Compile'
        run: ./build.cmd Compile
```

In the second workflow, we want to generate an artifact as part of the execution. Change the `Zip` target as follows:

```csharp

    Target Zip => _ => _
        .DependsOn(Publish)
        .Produces(ArtifactDirectory / "*.zip")
        .Executes(() =>
        {
            ZipFile.CreateFromDirectory(OutputDirectory, ArtifactDirectory / "deployment.zip");
        });
```

The attribute to add will be:

```csharp
[GitHubActions(
    "deploy",
    GitHubActionsImage.UbuntuLatest,
    On = new[] { GitHubActionsTrigger.WorkflowDispatch },
    InvokedTargets = new[] { nameof(Deploy) },
    ImportSecrets = new[] { nameof(WebAppPassword)},
    AutoGenerate = false)]
```

In this case, the attribute has the `AutoGenerate` set to `false` because we want to control the generation explicitly by running the following command `nuke --generate-configuration GitHubActions_deploy --host GitHubActions`:

```yaml
name: deploy

on: [workflow_dispatch]

jobs:
  ubuntu-latest:
    name: ubuntu-latest
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: Cache .nuke/temp, ~/.nuget/packages
        uses: actions/cache@v2
        with:
          path: |
            .nuke/temp
            ~/.nuget/packages
          key: ${{ runner.os }}-${{ hashFiles('**/global.json', '**/*.csproj') }}
      - name: Run './build.cmd Deploy'
        run: ./build.cmd Deploy
        env:
          WebAppPassword: ${{ secrets.WEB_APP_PASSWORD }}
      - uses: actions/upload-artifact@v1
        with:
          name: artifact
          path: artifact
```

Modify the `deploy.yml` file to complete the parameters needed by the `Deploy` target as follow:

```yaml
name: deploy

on: [workflow_dispatch]

jobs:
  ubuntu-latest:
    name: ubuntu-latest
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - name: Cache .nuke/temp, ~/.nuget/packages
        uses: actions/cache@v2
        with:
          path: |
            .nuke/temp
            ~/.nuget/packages
          key: ${{ runner.os }}-${{ hashFiles('**/global.json', '**/*.csproj') }}
      - name: Run './build.cmd Deploy'
        run: ./build.cmd Deploy
        env:
          WebAppPassword: ${{ secrets.WEB_APP_PASSWORD }}
          WebAppUser: $nuke-sandbox-app
          WebAppName: nuke-sandbox-app
      - uses: actions/upload-artifact@v1
        with:
          name: artifact
          path: artifact
```

By default on Windows, the `build.cmd` and `build.sh` are not recognized as executable. We can set the executable flag on any file using the `update-index` git sub-command:

```bash
git update-index --chmod=+x .\build.sh
git update-index --chmod=+x .\build.cmd
```

Go to GitHub and add the `WEB_APP_PASSWORD` secret:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681597711302/13651fa2-90f1-4574-958b-f218099cc03f.png align="center")

Push all the files to GitHub to start seeing the workflows in action:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681597784152/db6d2632-68ef-47a8-8aa6-e40588e97632.png align="center")

Run the `deploy` workflow to see the following output:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681598385584/241e8c3f-b0ad-4bdc-963c-39bed6930b96.png align="center")

You can find the final code [here](https://github.com/raulnq/nuke-sandbox). Thanks, and happy coding.