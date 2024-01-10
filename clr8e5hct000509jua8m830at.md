---
title: "GitHub Packages: Publishing NuGet Packages using NUKE (with GitVersion and GitHub Actions)"
datePublished: Wed Jan 10 2024 23:07:48 GMT+0000 (Coordinated Universal Time)
cuid: clr8e5hct000509jua8m830at
slug: github-packages-publishing-nuget-packages-using-nuke-with-gitversion-and-github-actions
tags: net, github-actions, nuke, gitversion, github-packages

---

> GitHub Packages is a platform for hosting and managing packages, including containers and other dependencies. GitHub Packages combines your source code and packages in one place to provide integrated permissions management and billing, so you can centralize your software development on GitHub.

In this article, we will further explore the capabilities of [Nuke](https://nuke.build/) as an automation tool. Specifically, we will establish a CI/CD pipeline to upload a NuGet Package to GitHub Packages.

## The Library

Let's create an empty class library, and run the following commands:

```powershell
dotnet new classlib -o MyLib
dotnet new sln -n MyLib
dotnet sln add --in-root MyLib
```

## Install Nuke

```powershell
dotnet tool install Nuke.GlobalTool --global
nuke :setup
```

Open the solution, navigate to the `build_` project, and delete all the default targets in the `Build.cs` file.

## Create a Personal Access Token (PAT)

Login into our GitHub account, go to `Settings`, `Developer Settings`, `Personal Access Tokens`, and `Tokens (classic)`. Click the `Generate new token (classic)` button:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1704909613299/8fc8a200-550c-431b-bf0d-54a51efe72ff.png align="center")

Click `Generate token` and copy the token for future use, in our case, to test our pipeline locally.

## Adding a new Nuget Source

The first target will add GitHub Packages as a new NuGet source:

```csharp
[Parameter()]
readonly string GitHubUser = GitHubActions.Instance?.RepositoryOwner;

[Parameter()]
[Secret] 
readonly string GitHubToken;

Target AddSource => _ => _      
	.Requires(() => GitHubUser)
	.Requires(() => GitHubToken)
	.Executes(() =>
	{
		try
		{
			DotNetNuGetAddSource(s => s
		   .SetName("github")
		   .SetUsername(GitHubUser)
		   .SetPassword(GitHubToken)
		   .EnableStorePasswordInClearText()
		   .SetSource($"https://nuget.pkg.github.com/{GitHubUser}/index.json"));
		}
		catch
		{
			Log.Information("Source already added");
		}
	   ;
	});
```

The `GitHubUser` and `GitHubToken` are required for the target. The `GitHubUser` will be initialized with the `GITHUB_REPOSITORY_OWNER` environment variable when run as part of a GitHub Action. The `--store-password-in-clear-text` option was enabled because GitHub Actions does not support encryption.

## Create the Nuget Package

To generate a version number for our package, we will use [GitVersion](https://gitversion.net/). Execute the following command:

```powershell
nuke :add-package GitVersion.Tool
```

We will define two targets, one to clean up the output directory and the other to create the package:

```csharp
[GitVersion]
readonly GitVersion GitVersion;

AbsolutePath PackagesDirectory => RootDirectory / "packages";

Target Clean => _ => _
	.Executes(() =>
	{
		PackagesDirectory.CreateOrCleanDirectory();
	});

Target Pack => _ => _
	.DependsOn(Clean)
	.DependsOn(AddSource)
	.Executes(() =>
	{
		DotNetPack(s => s
		.SetProject(RootDirectory / "MyLib")
		.SetOutputDirectory(PackagesDirectory)
		.SetPackageProjectUrl($"https://github.com/{GitHubUser}/github-packages-nuke")
		.SetVersion(GitVersion.SemVer)
		.SetPackageId("MyLib")
		.SetAuthors($"{GitHubUser}")
		.SetDescription("MyLib nuget package")
		.SetConfiguration(Configuration));
	});
```

The `Pack` target will depend on both the `AddSource` and `Clean` targets. A `PackagesDirectory` is defined to output the packages in that location.

## Push the Nuget Package

The final target pushes all the NuGet packages to our output folder and depends on the `Pack` target:

```csharp
Target Push => _ => _
	.DependsOn(Pack)  
	.Executes(() =>
	{
		DotNetNuGetPush(s => s
		.SetTargetPath(PackagesDirectory / "*.nupkg")
		.SetApiKey(GitHubToken)
		.SetSource("github"));
	});
```

Run `nuke Push --GitHubUser <GITHUB_USER> --GitHubToken <NUGET_TOKEN>` to view our package on GitHub:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1704913262835/f19dff4f-34da-4452-8edc-6cb8dca4b02f.png align="center")

## Create a GitHub Action Workflow

We can generate a GitHub Action workflow from our target definitions by adding the `GitHubActions` attribute at the top of the `Build` class:

```csharp
[GitHubActions(
    "Push",
    GitHubActionsImage.UbuntuLatest,
    On = new[] { GitHubActionsTrigger.WorkflowDispatch },
    InvokedTargets = new[] { nameof(Push) }, 
    EnableGitHubToken = true, 
    AutoGenerate = false)]
```

The workflow's name will be `Push`, invoking the target with the same name and triggered manually using the `GitHubActionsTrigger.WorkflowDispatch` property. The `EnableGitHubToken` property instructs the generator to include an environment variable with the `GITHUB_TOKEN`. Run `nuke --generate-configuration GitHubActions_Push --host GitHubActions` to generate the `Push.yml` file under the path `.github\workflows`.

The `actions/checkout@v3` is optimized by default to fetch a single commit, but GitVersion requires the entire commit history. That's why we are customizing that workflow as follows:

```yaml
name: Push

on: [workflow_dispatch]

jobs:
  ubuntu-latest:
    name: ubuntu-latest
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      - name: 'Cache: .nuke/temp, ~/.nuget/packages'
        uses: actions/cache@v3
        with:
          path: |
            .nuke/temp
            ~/.nuget/packages
          key: ${{ runner.os }}-${{ hashFiles('**/global.json', '**/*.csproj', '**/Directory.Packages.props') }}
      - name: 'Run: Push'
        run: ./build.cmd Push
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Another option would be to include an additional step, such as:

```yaml
- name: Fetch tags
  run: git fetch --prune --unshallow --tags
```

To push our code to the repository, run the following commands:

```powershell
git add .
git commit -m "initial commit"
git push
```

To update our package version, create a tag by using the following command:

```powershell
git tag 2.0.0
git push origin --tags
```

Before executing the workflow, we need to grant sufficient permissions to the `GITHUB_TOKEN`. In our repository, go to `Settings`, `Actions`, `General`, and change the `Workflow permissions` to `Read and write permissions`:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1704924675180/4110f7e4-dfbf-4722-a58d-9666625d4d72.png align="center")

Navigate to `Packages`, then to `Package Settings`, and click on the `Add Repository` button to add our repository. Then, change the Role to `Admin`:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1704924951205/2f9bf38c-7b20-4313-8da6-e1d8da1fee33.png align="center")

Now we are ready to run the workflow:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1704926670281/94f252b4-c891-48de-a821-64890e57ca67.png align="center")

Keep in mind that we can use the PAT (as a secret in our repository) instead of the `GITHUB_TOKEN`, which would allow us to bypass all the configuration we did for it.

You can find the final code [here](https://github.com/raulnq/github-packages-nuke). Thanks, and happy coding.