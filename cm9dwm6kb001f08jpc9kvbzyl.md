---
title: "How to Squash Commits on a Remote Git Branch"
datePublished: Sat Apr 12 2025 07:38:14 GMT+0000 (Coordinated Universal Time)
cuid: cm9dwm6kb001f08jpc9kvbzyl
slug: how-to-squash-commits-on-a-remote-git-branch
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1744397775208/5baff763-58d6-4abd-ab4c-81daa71ef255.png
tags: git

---

Squashing commits is a powerful tool, but like any tool that changes history, it should be used carefully. The main goal is to make the Git history cleaner, easier to understand, and simpler for others. While developing a feature, we make many small, incremental commits. These commits show the process of development but usually don't add value to the final history. Squashing them into one (or a few) well-described commits that represent the completed feature or logical parts makes the history much easier to understand later. So, let's start by making sure our local branch is up-to-date:

```powershell
git checkout <MY_BRANCH>
```

Display the log history:

```powershell
git log --pretty=format:"%h %s" --graph
```

This is what the outcome will look like:

```powershell
* 51434fc Update 4
* 7939622 Update 3
* 6be53e7 Update 2
* e8ab8e0 Update 1
* 04d0a65 Initial commit
```

The first step is to rebase back to a commit before the sequence of commits we want to squash. There are three ways to start a rebase:

* Rebase back `N` commits to squash the last `N` commits.
    

```powershell
git rebase -i HEAD~4
```

* Rebase back to a specific commit hash. In this case, we need to use the hash of the commit before the first one we want to squash:
    

```powershell
git rebase -i 04d0a65
```

* Rebase onto the base branch to squash all commits on our feature branch since it diverged from a base branch.
    

```powershell
git rebase -i origin/main
```

All three options above will open a text editor with a list of the commits we specified, the oldest first:

```powershell
pick e8ab8e0 Update 1
pick 6be53e7 Update 2
pick 7939622 Update 3
pick 51434fc Update 4
```

To squash commits, keep the first commit we want to retain as `pick`. Then, change the `pick` command for the subsequent commits we want to merge into the first one to `squash`.

```powershell
pick e8ab8e0 Update 1
squash 6be53e7 Update 2
squash 7939622 Update 3
squash 51434fc Update 4
```

> We can exit the editor at any time by pressing `ESC`, typing `:q!`, and then pressing `ENTER`.

Save and close the editor by pressing `ESC`, then typing `:wq`, and pressing `ENTER`. Git will open another editor instance allowing, us to combine and edit the commit messages for the resulting single commit.

```powershell
# This is a combination of 4 commits.
# This is the 1st commit message:
Big update
```

Save and close the editor and check our log history to see the changes:

```powershell
* 4ce3087 Big update
* 04d0a65 Initial commit
```

Since we have rewritten the history, a regular `git push` will be rejected. We need to force the push. Using `--force-with-lease` is strongly recommended over `--force` because it checks if the remote branch has changed since we last fetched.

```powershell
git push --force-with-lease origin <MY_BRANCH>
```

Thanks, and happy coding.