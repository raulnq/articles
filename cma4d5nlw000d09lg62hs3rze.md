---
title: "The Power of Rebasing Before Merging"
datePublished: Wed Apr 30 2025 20:03:17 GMT+0000 (Coordinated Universal Time)
cuid: cma4d5nlw000d09lg62hs3rze
slug: the-power-of-rebasing-before-merging
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1745946897458/2aef898e-a445-46f5-b161-a94dc5a40177.png
tags: git, git-rebase

---

When using Git, understanding how to rebase our feature branch onto a target branch before merging is a must, particularly in teams that prefer to maintain a linear commit history. Let's explore the common situations we might encounter when using the `git rebase` command.

> Rebasing is like saying: "Take all the work I did on my feature branch, pretend I started it *today* based on the latest version of the main code, and put my changes on top of that." It cleans up the history by making it look like development happened sequentially, even when it didn't.

As the first step, we will clone a GitHub repository by running the commands:

```powershell
git clone <MY_REPO_URL>
cd <MY_REPO_DIR>
```

> Before creating any text files, run the following command to set `UTF-8` as the default encoding (PowerShell):
> 
> ```powershell
> $PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
> ```

## No Conflicts

In this scenario, we will add changes to the `main` and `feature` branches that will not cause conflicts during the rebase. Add a file in the `main` branch:

```powershell
echo "Line 1 from main" > file.txt
git add file.txt
git commit -m "Commit 1 from main"
git push origin main
```

Create the `feature` branch from the `main` branch:

```powershell
git checkout -b feature
```

Go back to the `main` branch and add a second file:

```powershell
git checkout main
echo "Line 1 from main" > another_file.txt
git add another_file.txt
git commit -m "Commit 2 from main"
git push origin main
```

Modify the `file.txt` file in the `feature` branch:

```powershell
git checkout feature
echo "Line 2 from feature" >> file.txt
git add file.txt
git commit -m "Commit 1 from feature"
```

Run `git log --oneline --graph --all` to see the current commit history:

```powershell
* d956da2 (HEAD -> feature) Commit 1 from feature
| * 7e59e9b (origin/main, origin/HEAD, main) Commit 2 from main
|/
* a282a9d Commit 1 from main
* e315dd5 Initial commit
```

> Always pull the latest remote changes with `git fetch origin` before rebasing.

Rebase the `feature` branch:

```powershell
git rebase origin/main 
```

As a result, the rebase is executed successfully. Finally, push the rebased branch:

```powershell
git push origin feature --force-with-lease
```

To simulate a pull request, run:

```powershell
git checkout main
git merge feature
git push origin main
```

Check the commit history:

```powershell
* caa0728 (HEAD -> main, origin/main, origin/feature, origin/HEAD, feature) Commit 1 from feature
* 7e59e9b Commit 2 from main
* a282a9d Commit 1 from main
* e315dd5 Initial commit
```

## Modify Conflict

In this second scenario, we will update the same file in both branches. Go back to the `main` branch and modify the `file.txt` file:

```powershell
git checkout main
echo "Line 2 from main" >> file.txt
git add file.txt
git commit -m "Commit 3 from main"
git push origin main
```

Modify the `file.txt` file in the `feature` branch:

```powershell
git checkout feature
echo "Line 3 from feature" >> file.txt
git add file.txt
git commit -m "Commit 2 from feature"
```

See the current commit history:

```powershell
* 7083f03 (HEAD -> feature) Commit 2 from feature
| * 0825d7e (origin/main, origin/HEAD, main) Commit 3 from main
|/
* caa0728 (origin/feature) Commit 1 from feature
* 7e59e9b Commit 2 from main
* a282a9d Commit 1 from main
* e315dd5 Initial commit
```

Rebase the `feature` branch by running `git rebase origin/main`:

```powershell
Auto-merging file.txt
CONFLICT (content): Merge conflict in file.txt
error: could not apply 7083f03... Commit 2 from feature
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
Could not apply 7083f03... Commit 2 from feature
```

Open the `file.txt` file in your favorite editor:

```plaintext
Line 1 from main
Line 2 from feature
<<<<<<< HEAD
Line 2 from main
=======
Line 3 from feature
>>>>>>> 7083f03 (Commit 2 from feature)    
```

To resolve the conflict, in this case, we will keep the feature branch change:

```plaintext
Line 1 from main
Line 2 from feature
Line 3 from feature
```

Save the file and run `git add file.txt`. Since this is the only conflict, we can continue with the rebase by running `git rebase --continue`. This will open a text editor where we can update the commit message based on the changes we have made. Save and close the editor by pressing `ESC`, then typing `:wq`, and pressing `ENTER`. Push the rebased branch:

```powershell
git push origin feature --force-with-lease
```

Let’s simulate a pull request:

```powershell
git checkout main
git merge feature
git push origin main
```

Check the commit history:

```powershell
* 2c5e5d5 (HEAD -> main, origin/main, origin/feature, origin/HEAD, feature) Commit 2 from feature
* 0825d7e Commit 3 from main
* caa0728 Commit 1 from feature
* 7e59e9b Commit 2 from main
* a282a9d Commit 1 from main
* e315dd5 Initial commit
```

## Delete Conflict

In the final scenario, we will update a file in one branch and delete it in the other. Go back to the `main` branch and delete the `another_file.txt` file:

```powershell
git checkout main
rm another_file.txt
git rm another_file.txt
git commit -m "Commit 4 from main"
git push origin main
```

> The `rm` command is not need . The command `git rm` is is sufficient on its own, as it removes the file from the working directory *and* stages the deletion.

Modify the `another_file.txt` file in the `feature` branch:

```powershell
git checkout feature
echo "Line 3 from feature" >> another_file.txt
git add another_file.txt
git commit -m "Commit 3 from feature"
```

See the current commit history:

```powershell
* 9cc564d (HEAD -> feature) Commit 3 from feature
| * f7bf8a3 (origin/main, origin/HEAD, main) Commit 4 from main
|/
* 2c5e5d5 (origin/feature) Commit 2 from feature
* 0825d7e Commit 3 from main
* caa0728 Commit 1 from feature
* 7e59e9b Commit 2 from main
* a282a9d Commit 1 from main
* e315dd5 Initial commit
```

Rebase the `feature` branch by running `git rebase origin/main`:

```powershell
CONFLICT (modify/delete): another_file.txt deleted in HEAD and modified in 9cc564d (Commit 3 from feature).  Version 9cc564d (Commit 3 from feature) of another_file.txt left in tree.
error: could not apply 9cc564d... Commit 3 from feature
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
Could not apply 9cc564d... Commit 3 from feature
```

Resolve the conflict, in this case by deleting the file:

```powershell
rm another_file.txt
git rm another_file.txt
```

Continue the rebase by running `git rebase --continue`. Then push the rebased branch by running `git push origin feature --force-with-lease`. Check the commit history:

```powershell
* f7bf8a3 (HEAD -> feature, origin/main, origin/feature, origin/HEAD, main) Commit 4 from main
* 2c5e5d5 Commit 2 from feature
* 0825d7e Commit 3 from main
* caa0728 Commit 1 from feature
* 7e59e9b Commit 2 from main
* a282a9d Commit 1 from main
* e315dd5 Initial commit
```

Note that `Commit 3 from feature` is no longer shown as a separate commit because the file it modified was removed during conflict resolution, making its changes empty. If things go wrong or we get confused, we can always stop and reset:

```powershell
git rebase --abort
```

Thanks, and happy coding.