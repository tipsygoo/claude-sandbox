# Create the git repo for the setup

## Description

Create one git repository that will hold everything needed to rebuild this setup: the Containerfile (the image recipe) and the claude-box launcher script. Nothing is built yet. This step only creates the folder structure and the version history.

### 2.1 Check that git is installed

```bash
# Print the git version. If it's missing, the second command installs it.
git --version || sudo apt install -y git
```

### 2.2 Set your git identity (one time per machine)

```bash
# Every commit records who made it. Replace with your own name and email.
# --global means "for all repos on this machine", stored in ~/.gitconfig.
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Name the default branch "main" for new repos.
git config --global init.defaultBranch main
```

