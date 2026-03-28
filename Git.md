# Git

tags: #git #github #version-control
source: [[Isaac's Note]]

---

## Why use Git?

### Git is a distributed version control system

- It's a system that records changes to our files over time
- We can recall specific version of those files at any given time
- Many people can easily collaborate on a project and have their own version of project files on their computer

### Git is allowed to do

- Store revisions in a project history in just one directory
- Rewind to any revision in the project
- Work on new features without messing up the main codebase
- Easily collaborate with other programmers

### Github

- Online service that hosts our projects
- Share our code with other developers
- Developers can download the projects and work on them
- They can re-upload their edits and merge them with the main codebase

---

## Installing Git & Setting up

Install git with homebrew:

```bash
brew install git
```

Check git version:

```bash
git --version
```

Set up user name and email:

```bash
git config --global user.name "isaaclee"
git config --global user.email "isaaclee@example.com"
```

---

## How Git Works

### Repositories (Repo's)

A repository is where git stores all your project files and the history of changes.

### Commits

Each commit is a snapshot of your project at a specific point in time.

---

## Creating a Repository

### Create a Repo

```bash
git init
```

### Check Status

```bash
git status
```

→ Shows untracked files: files not yet added to git

---

## Git Basic Commit

### Add file to git

```bash
git add *
# or specific file:
git add school_list.xlsx
```

### Save file to git

```bash
git commit -m "commit message"
```

### Read the log with commit messages

```bash
git log
```

### Add Github link to local repo

```bash
git remote add origin [github repo HTTPS URL]
```

### Push the local repo to github

First push:

```bash
git push -u origin main
# or:
git push -u origin master
```

Subsequent pushes:

```bash
git push -u
```

### Pull repo from github

```bash
git clone url
```

---

## Git Conflicts

### A and B file conflicts

If B tries to push after A has already pushed changes, B will get a conflict error.

### How to fix conflicts

```bash
git pull
```

The terminal will show which lines are different. Choose which version to keep, then:

```bash
git add .
git commit -m "resolved merge conflict"
```

---

## Git Branch

### Create new branch

```bash
git checkout -b <branch name>
```

### Merge the branch to main

**[Method 1] Use Github pull request to merge**

Push branch to Github and create a Pull Request through the UI.

If conflict:

```bash
git switch main
git pull
git switch Isaac-branch
git rebase main
git rebase --continue
```

**[Method 2] Merge on local then push to github**

```bash
git switch main
git merge <branch name>
git push
```

---

## Git Errors

### fatal: The current branch master has no upstream branch

```bash
git push --set-upstream origin master
```
