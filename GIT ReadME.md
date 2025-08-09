# Git & GitHub Guide

## 1. Introduction to Git

**Git** is a **version control system** that helps track changes in files and collaborate with others.

* Created by Linus Torvalds in 2005.
* Allows you to maintain versions, revert changes, and work in teams.

### Why Use Git?

* Track changes in code/files.
* Collaborate with multiple developers.
* Revert to older versions when needed.

---

## 2. Introduction to GitHub

**GitHub** is a **cloud-based platform** that hosts Git repositories.

* Provides a web interface for Git.
* Supports collaboration, issue tracking, pull requests, and more.

### Difference Between Git & GitHub

| Git                            | GitHub                                  |
| ------------------------------ | --------------------------------------- |
| Local tool for version control | Cloud platform hosting Git repositories |
| Works offline                  | Works online                            |
| Command-line based             | Web interface + CLI support             |

---

## 3. Git Installation 

* **Windows:** Download from [https://git-scm.com](https://git-scm.com)
* **Linux:**

  ```bash
  sudo apt install git -y
  ```
* **Mac:**

  ```bash
  brew install git
  ```

Check installation:

```bash
git --version
```

---

## 4. Git Configuration

Set username and email:

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

Check configured username:

```bash
git config --global user.name
```

Check configured email:

```bash
git config --global user.email
```

Set default branch name:

```bash
git config --global init.defaultBranch main
```

List all Git configurations:

```bash
git config --list
```

---

## 5. Git Workflow Diagram

Below is a typical Git workflow diagram showing the process from making changes to pushing them to a remote repository:

![Git Workflow Diagram](https://infopalette.com/wp-content/uploads/2024/02/Git-WorkFlow-1-891x1024.gif)

---

## 6. Basic Git Commands

### Initialize Repository

```bash
git init
```

### Clone a Repository

```bash
git clone <repo_url>
```

### Check Repository Status

```bash
git status
```

### Stage Changes

```bash
git add <filename>
# or add all changes
git add .
```

### Commit Changes

```bash
git commit -m "Your commit message"
```

### View Commit History

```bash
git log
```

### Push Changes to Remote

```bash
git push origin <branch_name>
```

### Pull Latest Changes

```bash
git pull origin <branch_name>
```

### Fetch Changes (without merging)

```bash
git fetch origin
```

### Merge Branches

```bash
git merge <branch_name>
```

### Cherry-pick a Commit (apply specific commit to current branch)

```bash
git cherry-pick <commit_hash>
```

### Create New Branch

```bash
git branch <branch_name>
```

### Switch to Branch

```bash
git checkout <branch_name>
# or
git switch <branch_name>
```

### Delete Branch

```bash
git branch -d <branch_name>
```

### View All Branches

```bash
git branch -a
```

### View Remote URLs

```bash
git remote -v
```

---

## 7. Common Git Flow Example

1. Clone repository.
2. Create a new branch.
3. Make changes.
4. Stage (`git add .`).
5. Commit (`git commit -m "message"`).
6. Push (`git push origin branch`).
7. Create a Pull Request on GitHub.

---

## 8. Helpful Tips

* Always pull latest changes before starting work.
* Write meaningful commit messages.
* Use `.gitignore` to avoid tracking unnecessary files.
* Use `git stash` to temporarily save uncommitted work.
* Use `git diff` to see changes before committing.
* Use `git cherry-pick` to apply specific commits from one branch to another.

---

## 9. References

* [Git Official Docs](https://git-scm.com/doc)
* [GitHub Guides](https://guides.github.com/)
