## Semantic Versioning with GitVersion (GithubFlow)

In a previous [post](https://blog.raulnq.com/semantic-versioning-with-gitversion-gitflow) we saw how to use [GitVersion](https://gitversion.net/) to generate version numbers following [Semantinc Versioning](https://semver.org/) (using [GitFlow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) as a branching strategy). Today we will do the same exercise but using [GitHubFlow](https://githubflow.github.io/) as a branching strategy. Let's start creating a new [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) repository:

```powershell
mkdir gitversion-githubflow
cd .\gitversion-githubflow\
git init
``` 

Run the command `dotnet-gitversion init` to start the GitVersion setup:

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
Select option 2:

```powershell
The way you will use GitVersion will change a lot based on your branching strategy. What branching strategy will you be using:

1) GitFlow (or similar)
2) GitHubFlow
3) Unsure, tell me more
``` 
Select again option 2:

```powershell
By default GitVersion will only increment the version when tagged

What do you want the default increment mode to be (can be overriden per branch):

1) Follow SemVer and only increment when a release has been tagged (continuous delivery mode)
2) Increment based on branch config every commit (continuous deployment mode)
3) Each merged branch against main will increment the version (mainline mode)
4) Skip
``` 

And finally, option 2. As the last step, the wizard will show us to the first menu, select option 0 (`Save changes and exit`). Run `dotnet-gitversion /showconfig` to see the current GitVersion configuration. GitVersion added a file called GitVersion.yml; we will use this file as our first commit:


```powershell
git add .
git commit -m 'initial commit'
``` 

Then run the `dotnet-gitversion` command:

```powershell
{
  "Major": 0,
  "Minor": 1,
  "Patch": 0,
  "PreReleaseTag": "ci.0",
  "PreReleaseTagWithDash": "-ci.0",
  "PreReleaseLabel": "ci",
  "PreReleaseLabelWithDash": "-ci",
  "PreReleaseNumber": 0,
  "WeightedPreReleaseNumber": 55000,
  "BuildMetaData": null,
  "BuildMetaDataPadded": "",
  "FullBuildMetaData": "Branch.master.Sha.ac95b6b7ced4ad04b43ba4d90a5714be322badd7",
  "MajorMinorPatch": "0.1.0",
  "SemVer": "0.1.0-ci.0",
  "LegacySemVer": "0.1.0-ci0",
  "LegacySemVerPadded": "0.1.0-ci0000",
  "AssemblySemVer": "0.1.0.0",
  "AssemblySemFileVer": "0.1.0.0",
  "FullSemVer": "0.1.0-ci.0",
  "InformationalVersion": "0.1.0-ci.0+Branch.master.Sha.ac95b6b7ced4ad04b43ba4d90a5714be322badd7",
  "BranchName": "master",
  "EscapedBranchName": "master",
  "Sha": "ac95b6b7ced4ad04b43ba4d90a5714be322badd7",
  "ShortSha": "ac95b6b",
  "NuGetVersionV2": "0.1.0-ci0000",
  "NuGetVersion": "0.1.0-ci0000",
  "NuGetPreReleaseTagV2": "ci0000",
  "NuGetPreReleaseTag": "ci0000",
  "VersionSourceSha": "ac95b6b7ced4ad04b43ba4d90a5714be322badd7",
  "CommitsSinceVersionSource": 0,
  "CommitsSinceVersionSourcePadded": "0000",
  "UncommittedChanges": 0,
  "CommitDate": "2022-07-02"
}
``` 

As we can see, the current  `FullSemVer` is `0.1.0-ci.0`. Let's create our first feature branch:

```powershell
git checkout -b feature/MyFeatureA
``` 

Run the command `dotnet-gitversion /showvariable FullSemVer`:

```powershell
0.1.0-MyFeatureA.0
``` 

Add a commit to see how this version number starts to change:

```powershell
git commit -m 'featurea A: first change' --allow-empty
``` 

Run the command `dotnet-gitversion /showvariable FullSemVer` again:

```powershell
0.1.0-MyFeatureA.1
``` 

Let's merge the feature branch to the `master` branch:

```powershell
git checkout master
git merge --no-ff feature/myFeatureA
git branch -d feature/myFeatureA
``` 

Run `dotnet-gitversion /showvariable FullSemVer`:

```powershell
0.1.0-ci.2
``` 

We will repeat the last steps a couple of times to see how `FullSemVer` changes. Here is the result as seen through the `git log --oneline --graph` command:

```powershell
*   eb76de2 (HEAD -> master) Merge branch 'feature/myFeatureC' into master
|\
| * a38b9b5 featurea C: first change
|/
*   d2928f3 Merge branch 'feature/myFeatureB' into master
|\
| * 6f736a8 featurea B: first change
|/
*   2f06433 Merge branch 'feature/myFeatureA' into master
|\
| * 081792b featurea A: first change
|/
* 59ac5ad initial commit
``` 

Run `dotnet-gitversion /showvariable FullSemVer` to see the new value:

```powershell
0.1.0-ci.6
``` 

Great, the version number is moving together with our number of commits. But what else can we do?. Let's tag our `master` branch to see what happens:

```powershell
git tag '1.0.0'
``` 

Run `dotnet-gitversion /showvariable FullSemVer`:

```powershell
1.0.0
``` 

As we can see, the value follows what we tag. Now, we will create a hotfix branch, a new commit, and merge it back to `master`:


```powershell
git checkout -b hotfix/myHotfixA
git commit -m 'hotfix A: first change' --allow-empty
git checkout master
git merge --no-ff hotfix/myHotfixA
git branch -d hotfix/myHotfixA
``` 

Run `dotnet-gitversion /showvariable FullSemVer`:

```powershell
1.0.1-ci.2
``` 

To see more examples using GitHuibFlow, you can check the official documentation [here](https://gitversion.net/docs/learn/branching-strategies/githubflow/examples).


