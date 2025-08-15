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

## 4. Git Workflow Diagram

Below is a typical Git workflow diagram showing the process from making changes to pushing them to a remote repository:

![Git Workflow Diagram](https://infopalette.com/wp-content/uploads/2024/02/Git-WorkFlow-1-891x1024.gif)

---

## 5. Git Configuration

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

## 6. Connect Git to GitHub (HTTPS + Personal Access Token)

Using HTTPS is simple and works without generating SSH keys. You’ll authenticate using a GitHub username and a **Personal Access Token (PAT)** instead of a password.

**Steps:**

1. Go to GitHub → **Settings → Developer settings → Personal access tokens** → **Tokens (classic)** or **Fine-grained tokens**.
2. Click **Generate new token**, set expiration and scopes (usually `repo` is enough).
3. Copy the token (it will be shown once).
4. In your local Git repository:

```bash
git remote add origin https://github.com/username/repo.git  # Add remote repo
```

5. First push:

```bash
git push -u origin main  # Push local main branch to remote
```

6. Enter GitHub **username** and **token** when prompted.
7. Use Git Credential Manager to store token securely.

---

## 7. Creating a Repository (Local → GitHub)

```bash
mkdir my-project && cd my-project  # Create project folder
git init  # Initialize Git repo
git branch -M main  # Rename branch to main
echo "# My Project" > README.md  # Create README file
git add .  # Stage files
git commit -m "Initial commit"  # Commit changes
git remote add origin https://github.com/username/my-project.git  # Link to GitHub
git push -u origin main  # Push to GitHub
```

---

## 8. Git Commands Cheat Sheet (One-liners)

```bash
git status  # Show current changes and branch status
git add <file>  # Stage specific file
git add .  # Stage all changes
git commit -m "msg"  # Commit staged changes
git push origin <branch>  # Push branch to remote
git pull origin <branch>  # Pull latest changes
git branch  # List branches
git branch <name>  # Create new branch
git switch <name>  # Switch to branch
git checkout <name>  # Switch (old method)
git merge <branch>  # Merge branch into current
git fetch origin  # Fetch changes without merging
git remote -v  # View remote URLs
git remote set-url origin <new_url>  # Change remote URL
git log --oneline --graph  # View compact commit history
git diff  # Show unstaged changes
git stash  # Save uncommitted work temporarily
git stash pop  # Restore stashed work
git reset --hard HEAD  # Discard all local changes
```

---

## 9. Common Workflow

```bash
git pull origin main  # Update local repo
git switch -c feature  # Create and switch to feature branch
# Make changes, then:
git add .
git commit -m "Implement feature"
git push -u origin feature  # Push branch to GitHub
```

---

## 10. Tips

* Always pull before starting work.
* Use `.gitignore` to skip unnecessary files.
* Use `git diff` to review changes before committing.
* Use `git stash` to temporarily save uncommitted changes.

---

## 11. References

* [Git Official Docs](https://git-scm.com/doc)
* [GitHub Guides](https://guides.github.com/)
