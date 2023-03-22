---
title: "Migrating a repository from Bitbucket to GitHub"
datePublished: Wed Mar 22 2023 00:26:10 GMT+0000 (Coordinated Universal Time)
cuid: clfiy1y8q000209l4dueb2dt5
slug: migrating-a-repository-from-bitbucket-to-github
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1679435615369/ad162394-6567-4ae7-a46e-6e08ca740a9e.png
tags: github, git, bitbucket

---

Several years ago, we started using Bitbucket because it provided unlimited private repositories. But today, GitHub also offers the same plus useful tools and integrations. So, we started to migrate repositories from Bitbucket to Github.

  
Let's start by looking at the commit history (master branch) of the repository we want to migrate. Run `git log --oneline --graph`:

```bash
*   2849da4 (HEAD -> master, tag: 1.0.0, origin/master, origin/HEAD) Pull request #2: Update README.md
|\
| * f5ec292 (origin/release/1.0.0, release/1.0.0) Update README.md
|/
* ce82b26 Create README.md
```

And the remote branches with `git branch -r`:

```bash
  origin/HEAD -> origin/master
  origin/develop
  origin/master
  origin/release/1.0.0
```

The idea is to move everything to the new repository using the minimum number of git commands.

### Step 1: Create

Create a new repository on GitHub. The repository needs to be empty, with no README file, no license file, and no .gitignore file:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679440426687/5149b272-d49e-4524-82cc-3b59634fcbf5.png align="center")

### Step 2: Clone

Run `git clone --bare https://raulnq@bitbucket.org/raulnq/old-repository.git`:

```shell
Cloning into bare repository 'old-repository.git'...
remote: Enumerating objects: 8, done.
Receiving objects: 100% (8/8), done.
remote: Counting objects: 100% (8/8), done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 8 (delta 1), reused 0 (delta 0), pack-reused 0
Resolving deltas: 100% (1/1), done.
```

### Step 3: Push

Run `cd old-repository.git` and `git push --mirror https://github.com/raulnq/new-repository.git`:

```bash
Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 20 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (8/8), 834 bytes | 834.00 KiB/s, done.
Total 8 (delta 1), reused 8 (delta 1), pack-reused 0
remote: Resolving deltas: 100% (1/1), done.
To https://github.com/raulnq/new-repository.git
 * [new branch]      develop -> develop
 * [new branch]      master -> master
 * [new branch]      release/1.0.0 -> release/1.0.0
 * [new tag]         1.0.0 -> 1.0.0
```

### Step 4: Check

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1679440791587/709c564e-af5b-4e94-aa1f-31f0fb744500.png align="center")

And that's it. You can clone your new repository and start to work on it. Thank you, and happy coding.