Here's everything combined into a single Markdown file.

````markdown
# Git & GitHub Professional Setup Guide

A complete, production-ready checklist for setting up Git and GitHub on a new machine.

---

# Table of Contents

1. Install Git
2. Configure Git Identity
3. Set Default Branch
4. Enable Colored Output
5. Configure Line Endings
6. Set VS Code as Default Editor
7. Configure VS Code as Merge Tool
8. Enable Credential Manager
9. Create an SSH Key
10. Start SSH Agent
11. Copy Public Key
12. Add SSH Key to GitHub
13. Test SSH Connection
14. Verify GitHub Authentication
15. Create a Global `.gitignore`
16. Enable Useful Git Aliases
17. Enable Rebase on Pull
18. Automatically Prune Deleted Branches
19. Cache Credentials
20. Better Default Push
21. Verify Configuration
22. Install GitHub CLI
23. Authenticate GitHub CLI
24. Verify Login
25. Install Git LFS
26. Install Useful Developer Tools
27. Recommended VS Code Extensions
28. Recommended Folder Structure
29. Create Your First Repository
30. Daily Git Workflow
31. Backup Your Configuration
32. Professional Best Practices

---

# 1. Install Git

## Windows

Download Git:

https://git-scm.com/download/win

## macOS

```bash
brew install git
```

## Ubuntu

```bash
sudo apt update
sudo apt install git
```

## Verify Installation

```bash
git --version
```

Example:

```text
git version 2.50.1
```

---

# 2. Configure Git Identity

These values become part of every commit.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Verify:

```bash
git config --global --list
```

---

# 3. Set Default Branch

```bash
git config --global init.defaultBranch main
```

---

# 4. Enable Better Colored Output

```bash
git config --global color.ui auto
```

---

# 5. Configure Line Endings

## Windows

```bash
git config --global core.autocrlf true
```

## macOS/Linux

```bash
git config --global core.autocrlf input
```

---

# 6. Set VS Code as Default Editor

Install Visual Studio Code.

```bash
git config --global core.editor "code --wait"
```

Verify:

```bash
git config --global core.editor
```

---

# 7. Configure VS Code as Merge Tool

```bash
git config --global merge.tool vscode

git config --global mergetool.vscode.cmd "code --wait $MERGED"
```

---

# 8. Enable Credential Manager

Check:

```bash
git config --global credential.helper manager
```

If missing:

```bash
git config --global credential.helper manager-core
```

---

# 9. Create an SSH Key

```bash
ssh-keygen -t ed25519 -C "you@example.com"
```

Press **Enter** through the prompts.

Files created:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

---

# 10. Start SSH Agent

## Windows PowerShell

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic

Start-Service ssh-agent

ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

## macOS/Linux

```bash
eval "$(ssh-agent -s)"

ssh-add ~/.ssh/id_ed25519
```

---

# 11. Copy Public Key

## Windows

```powershell
Get-Content ~/.ssh/id_ed25519.pub
```

## Linux/macOS

```bash
cat ~/.ssh/id_ed25519.pub
```

---

# 12. Add SSH Key to GitHub

1. Open GitHub
2. Go to **Settings**
3. Select **SSH and GPG Keys**
4. Click **New SSH Key**
5. Paste your public key
6. Save

---

# 13. Test SSH Connection

```bash
ssh -T git@github.com
```

Expected output:

```text
Hi username!
You've successfully authenticated.
```

---

# 14. Verify GitHub Authentication

Clone repositories using SSH:

```bash
git clone git@github.com:username/repository.git
```

Prefer SSH over HTTPS whenever possible.

---

# 15. Create a Global `.gitignore`

Create:

```text
~/.gitignore_global
```

Example:

```gitignore
.DS_Store
Thumbs.db
*.log
node_modules/
.env
.vscode/settings.json
coverage/
dist/
build/
```

Configure Git:

```bash
git config --global core.excludesfile ~/.gitignore_global
```

---

# 16. Enable Useful Git Aliases

```bash
git config --global alias.st status
git config --global alias.br branch
git config --global alias.co checkout
git config --global alias.ci commit
git config --global alias.last "log -1 HEAD"
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.undo "reset HEAD~1"
```

Examples:

```bash
git st

git br

git lg
```

---

# 17. Enable Rebase on Pull

```bash
git config --global pull.rebase true
```

Keeps history cleaner.

---

# 18. Automatically Prune Deleted Branches

```bash
git config --global fetch.prune true
```

---

# 19. Cache Credentials

```bash
git config --global credential.helper manager
```

---

# 20. Better Default Push

```bash
git config --global push.default simple
```

---

# 21. Verify Everything

```bash
git config --list
```

---

# 22. Install GitHub CLI

## Windows

```powershell
winget install GitHub.cli
```

## macOS

```bash
brew install gh
```

## Ubuntu

```bash
sudo apt install gh
```

Verify:

```bash
gh --version
```

---

# 23. Authenticate GitHub CLI

```bash
gh auth login
```

Choose:

- GitHub.com
- SSH
- Login with browser

---

# 24. Verify Login

```bash
gh auth status
```

---

# 25. Install Git LFS (Optional)

```bash
git lfs install
```

---

# 26. Install Useful Developer Tools

- Visual Studio Code
- Docker Desktop
- Node.js (LTS)
- Python
- Java (if needed)
- Postman
- Insomnia
- Windows Terminal
- PowerShell 7
- WSL2 (Windows)
- Fira Code
- JetBrains Mono

---

# 27. Recommended VS Code Extensions

- GitLens
- GitHub Pull Requests
- Error Lens
- Prettier
- ESLint
- Docker
- Remote SSH
- Dev Containers
- Markdown All in One
- REST Client
- EditorConfig

---

# 28. Recommended Folder Structure

## Windows

```text
C:\
└── Dev
    ├── Personal
    ├── Work
    ├── OpenSource
    └── Learning
```

## macOS/Linux

```text
~/Developer
├── Personal
├── Work
├── OpenSource
└── Learning
```

---

# 29. Create Your First Repository

```bash
mkdir my-project

cd my-project

git init

echo "# My Project" > README.md

git add .

git commit -m "Initial commit"
```

Connect to GitHub:

```bash
git remote add origin git@github.com:username/my-project.git

git push -u origin main
```

---

# 30. Daily Git Workflow

```bash
git pull

git checkout -b feature/new-feature

# edit files

git add .

git commit -m "Add new feature"

git push origin feature/new-feature
```

After opening and merging a Pull Request:

```bash
git checkout main

git pull

git branch -d feature/new-feature
```

---

# 31. Backup Your Configuration

View current configuration:

```bash
git config --global --list
```

Back up:

```text
~/.gitconfig

~/.ssh/

~/.gitignore_global
```

---

# 32. Professional Best Practices

- Use SSH instead of HTTPS.
- Enable Two-Factor Authentication (2FA).
- Use GitHub CLI (`gh`) for repository and PR management.
- Write meaningful commit messages.
- Follow Conventional Commits if your team uses them.
- Never commit secrets, API keys, passwords, or `.env` files.
- Use feature branches instead of committing directly to `main`.
- Use Pull Requests for code reviews.
- Keep Git, GitHub CLI, and your editor updated.
- Back up your Git configuration and SSH keys regularly.

---

# Final Checklist

- ✅ Git Installed
- ✅ Git Configured
- ✅ Default Branch Set
- ✅ SSH Configured
- ✅ GitHub Authentication Working
- ✅ Credential Manager Enabled
- ✅ Global `.gitignore` Configured
- ✅ Git Aliases Added
- ✅ GitHub CLI Installed
- ✅ Git LFS Installed (Optional)
- ✅ Development Tools Installed
- ✅ VS Code Extensions Installed
- ✅ First Repository Created
- ✅ Daily Workflow Understood
- ✅ Configuration Backed Up

---

## Congratulations!

You now have a **professional, production-ready Git & GitHub development environment** suitable for software development on Windows, macOS, or Linux.
````
