## Semantic Versioning with GitVersion (GitFlow)

Versioning software artifacts (Assemblies, NuGet, or NPM packages) has been proved to be a challenging procedure. Today, [Semantic Versioning](https://semver.org/) is a practice that has been on the rise over the last few years, but how to apply it to your projects?. Here is were [GitVersion](https://gitversion.net/) come in place.
> GitVersion is a tool that generates a Semantic Version number based on your Git history. The version number generated from GitVersion can then be used for various different purposes.

To test how GitVersion helps us get version numbers, today we are going to use  [GitFlow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) as a branching strategy. There are several ways to [install](https://gitversion.net/docs/usage/cli/installation) GitVersion. Here we are using the .NET global tool option:

```powershell
dotnet tool install --global GitVersion.Tool --version 5.*
``` 

Let's create a new [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) repository:

```powershell
mkdir gitversion-giflow
cd .\gitversion-giflow\
git init
``` 
Now it is time to see our first GitVersion command:

```powershell
dotnet-gitversion init
``` 

We are going thru the configuration process depending on the branch strategy mentioned before:

```powershell
GitVersion init will guide you through setting GitVersion up to work for you

Which would you like to change?

0) Save changes and exit
1) Exit without saving

2) Run getting started wizard

3) Set next version number
4) Branch specific configuration
5) Branch Increment mode (per commit/after tag) (Current: )
6) Assembly versioning scheme (Current: )
7) Setup build scripts
``` 
Select the option 2:

```powershell
The way you will use GitVersion will change a lot based on your branching strategy. What branching strategy will you be using:

1) GitFlow (or similar)
2) GitHubFlow
3) Unsure, tell me more
``` 
Select the option 1:

```powershell
By default GitVersion will only increment the version of the 'develop' branch every commit, all other branches will increment when tagged

What do you want the default increment mode to be (can be overriden per branch):

1) Follow SemVer and only increment when a release has been tagged (continuous delivery mode)
2) Increment based on branch config every commit (continuous deployment mode)
3) Each merged branch against main will increment the version (mainline mode)
4) Skip
``` 
Again option 1. As the last step, the wizard is going to show us to the first menu, select option 0 (`Save changes and exit`). Run the following command to see the current GitVersion configuration:

```powershell
dotnet-gitversion /showconfig
``` 

GitVersion added a file called GitVersion.yml. 

```powershell
mode: ContinuousDelivery
branches: {}
ignore:
  sha: []
merge-message-formats: {}

``` 

Let's add the first commit with this file:

```powershell
git add .
git commit -m 'initial commit' 
``` 

Then run the `dotnet-gitversion` command:

```powershell
{
  "Minor": 1,
  "Patch": 0,
  "PreReleaseTag": "",
  "PreReleaseTagWithDash": "",
  "PreReleaseLabel": "",
  "PreReleaseLabelWithDash": "",
  "PreReleaseNumber": null,
  "WeightedPreReleaseNumber": 60000,
  "BuildMetaData": 0,
  "BuildMetaDataPadded": "0000",
  "FullBuildMetaData": "0.Branch.master.Sha.17de78f354ad96dc7766c2d1242634cc903d8e2f",
  "MajorMinorPatch": "0.1.0",
  "SemVer": "0.1.0",
  "LegacySemVer": "0.1.0",
  "LegacySemVerPadded": "0.1.0",
  "AssemblySemVer": "0.1.0.0",
  "AssemblySemFileVer": "0.1.0.0",
  "FullSemVer": "0.1.0+0",
  "InformationalVersion": "0.1.0+0.Branch.master.Sha.17de78f354ad96dc7766c2d1242634cc903d8e2f",
  "BranchName": "master",
  "EscapedBranchName": "master",
  "Sha": "17de78f354ad96dc7766c2d1242634cc903d8e2f",
  "ShortSha": "17de78f",
  "NuGetVersionV2": "0.1.0",
  "NuGetVersion": "0.1.0",
  "NuGetPreReleaseTagV2": "",
  "NuGetPreReleaseTag": "",
  "VersionSourceSha": "17de78f354ad96dc7766c2d1242634cc903d8e2f",
  "CommitsSinceVersionSource": 0,
  "CommitsSinceVersionSourcePadded": "0000",
  "UncommittedChanges": 0,
  "CommitDate": "2022-05-22"
}
``` 

There is a lot of information, but the one that we will use will be FullSemVer. Notice that GitVersion uses by default version number `0.1.0`. Create a `develop` branch from `master`:

```powershell
git checkout -b develop
``` 

And then create a `feature` branch from `develop`:

```powershell
git checkout -b feature/myFeatureA
``` 

Run the command `dotnet-gitversion /showvariable FullSemVer`:

```powershell
0.1.0-myFeatureA.1+0
``` 

Add another commit (we are going to use empty commits by practicality):

```powershell
git commit -m 'featurea A: change 1' --allow-empty
``` 

Run again `dotnet-gitversion /showvariable FullSemVer` to see how the last number is moving:

```powershell
0.1.0-myFeatureA.1+1
``` 

Time to finalize the `feature` branch:

```powershell
git checkout develop
git merge --no-ff feature/myFeatureA
git branch -d feature/myFeatureA
``` 

Run `dotnet-gitversion /showvariable FullSemVer`:

```powershell
0.1.0-alpha.2
``` 

Create a new `feature` branch and repeat all steps until finalizing it and see how the version number changes in the `develop` branch:

```powershell
0.1.0-alpha.4
``` 

Run the command `git log --oneline --graph` to see the log history for the `develop` branch:


```powershell
*   fe48f23 (HEAD -> develop) Merge branch 'feature/myFeatureB' into develop
|\
| * 4ad9aa8 featurea B: change 1
|/
*   20481b3 Merge branch 'feature/myFeatureA' into develop
|\
| * df2170f featurea A: change 1
|/
* 17de78f (master) initial commit
``` 

Now it is time to create a `release` branch:

```powershell
git checkout -b release/1.0.0 develop
``` 

Run `dotnet-gitversion /showvariable FullSemVer`:

```powershell
1.0.0-beta.1+0
``` 

Add a commit to simulate a bug fix under the `release` branch:

```powershell
git commit -m 'bug fix' --allow-empty
``` 
 And check the version number:

```powershell
1.0.0-beta.1+1
```

Assume that we want to release this branch to production:

```powershell
git checkout master
git merge --no-ff release/1.0.0
git tag '1.0.0'
git checkout develop
git merge --no-ff release/1.0.0
git branch -d release/1.0.0
``` 

Now running `dotnet-gitversion /showvariable FullSemVer` under `master` will see `1.0.0` because the tag tells GitVersion that this is the new version number. If you check the version number in the `develop` branch, GitVersion figures out `1.1.0-alpha.2` as a new value. Let's do a hotfix now:

```powershell
git checkout master
git checkout -b hotfix/1.0.1 
``` 

Add a commit:

```powershell
git commit -m 'bug fix' --allow-empty
``` 

Run `dotnet-gitversion /showvariable FullSemVer`:

```powershell
1.0.1-beta.1+7
``` 

And finalize a `hotfix` branch:

```powershell
git checkout master
git merge --no-ff hotfix/1.0.1
git tag  '1.0.1'
git checkout develop
git merge --no-ff hotfix/1.0.1
git branch -d hotfix/1.0.1
``` 

Run `dotnet-gitversion /showvariable FullSemVer` under `master` branch to see `1.0.1` as version number.

To see more examples using GitFlow, you can check the official documentation [here](https://gitversion.net/docs/learn/branching-strategies/gitflow/examples).
