---
title: "Some commonly used git commands"
date: 2023-10-27 15:58:20
categories: ["Git"]
tags:
---

> 记录一些我需要常用的 git 命令行

### Link to a remote repo
If you want to push you local stuff to a remote repository in Github, do the following commands

First go the the target directory, and commit the updates.

```console
git init
git add .
git commit -m "commit message"
```

Then link your local repo with the remote repo. By convention, we name the remote repo as origin.
```console
git remote add origin <remote_repo_url>
```

You can also rename your remote repo 
```console
git remote rename <old_name> <new_name>
```

### Git push
If this is your first time to push the local content to a remote repo, use the following command. The **-u (--set-upstream)** flag is used to set the upstream banch for the current local branch.

> Don't forget to use it when pushing a branch for the first time to establish a relationship between the local and remote branches. 
>
> For instance, when you use ```git push -u origin master```, it not only pushes your local **master** on the remote repository but also sets the upstream branch for your local **master** branch to **origin/master**.
> 
>This allows you to use shorter commands like ```git push``` or `git pull`in the future without specifying the remote and branch explicitly.

```console
git push -u <remote_repo_name> <branch_name>

git push <remote_name> <branch_name>

git push 
```

### Git pull
The `git pull` command is a combination of `git fetch` and `git merge`. It is used to fetch and download content from a remote repo and immediately update the local repo to match the content. **<branch_name>** is the remote branch yuo wanna pull from. If not specified, it pulls from the branch your local branch is tracking.

If you want to pull from a different remote branch, you might need to handle the conflicting changes between you local branch and the remote branch you're pulling from. 

```console
git pull origin <branch_name>
```

You can omit the branch name if you are already on the branch from which you want to update.
```console
git pull origin 

git pull 
```

### Push you local branch to the remote repo
If you want to create a new branch and push it to the remote repo, execute the following commands

```console
git checkout -b <new_branch>
```

Use **-u** if you encounter an error indicating that the upstream branch does not exist.
```console
git push origin <new_branch>

git push -u origin <new_branch>
```

### Move to a remote branch

First list all the remote branches.
```console
git branch -r 
```

Then create a local branch that tracks the remote branch
```console
git checkout -b <feature_branch> origin/<feature_branch>
```


### Delete the local branch and apply to the remote repo
Before you try to delete a local branch, checkout another local branch first
```console
git branch -d <branch_name>
```

Apply the changes to the remote repo
```console
git push oriin --delete <branch_name> 
```



### Listing the remote repos
To checkout the list of your remote repos, the following command will return the name and the url of the remote repos.

```console
git remote -v
```

### Sync with the name of your remote repo
If you rename you remote repo, you need to keep your local repo in sync.

```console
git remote set-url origin <new_repository_url>
```

### Git commands about branch

rename your current branch to <new_branch_name>
```console
git branch -M <new_branch_name>
```

Move to another branch
```console
git checkout <another_branch_name>
```

Delete a local branch, use **-D** to enfore the delete even if there are changes that haven't been merged.
```console
git branch -d <branch_name>
git branch -D <branch_name>
```





