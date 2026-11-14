# Get the repo

## Description

Download the project onto your computer and check that all the pieces are there.

## Before you start

- Step 1 is done: Podman is installed and the test container ran.

## Key terms

- **git**: a tool that tracks changes to files and lets you copy whole projects.
- **repository (repo)**: a project folder together with its full history.
- **clone**: download a complete copy of a repository.

## 2.1 Check that git is installed

```bash
# Print the git version. If git is missing, the second command installs it.
# "||" means "if the first command fails, run the next one".
git --version || sudo apt install -y git
```

**Expected:** a version number, for example `git version 2.43.0`.

## 2.2 Clone the repo

```bash
# Download the project into a new folder called "claude-sandbox" in your home folder.
# Replace <REPO_URL> with the repository's https address (https://github.com/...).
git clone <REPO_URL> ~/claude-sandbox
```

- Use the `https://` address. A public repository needs no account or key.
- Keep the folder name and location (`~/claude-sandbox`). Later steps refer to that path.

**Expected:** a line starting with `Cloning into`, some progress lines, and no error.

## 2.3 Look around

```bash
# Go into the project folder.
cd ~/claude-sandbox

# List everything in it, including hidden files (-A), one per line (-1).
ls -A1
```

**Expected:** at least these entries: `.git`, `.gitignore`, `LICENSE`, `README.md`, `bin`, `docs`, `image`, `proxy`.

What each part is for:

| Path | What it is | Used in |
|---|---|---|
| `bin/claude-box` | The launcher script that starts a locked-down container for your project | Steps 4 and 8 |
| `image/Containerfile` | The recipe for the image that runs Claude Code | Step 3 |
| `image/managed-settings.json` | Claude Code rules baked into the image (secret files blocked, bypass mode off) | Step 9 |
| `image/CLAUDE.md` | General instructions Claude reads at the start of every session | Step 10 |
| `proxy/Containerfile` | The recipe for the proxy image, the only way out to the internet | Step 7 |
| `proxy/squid.conf` | The proxy's rules | Step 7 |
| `proxy/allowlist.txt` | The default list of websites the proxy lets through (the Claude hosts). Your own sites go in a separate file you create in Step 7 | Step 7 |
| `docs/` | This guide | |

## 2.4 Check that all the pieces are present

```bash
# For each file in the list, print OK if it exists or MISSING if it doesn't.
#   "for f in ...; do ...; done" repeats the commands once per file name
#   "[ -f "$f" ]" is true when "$f" is an existing regular file
for f in bin/claude-box image/Containerfile image/managed-settings.json image/CLAUDE.md \
         proxy/Containerfile proxy/squid.conf proxy/allowlist.txt; do
  if [ -f "$f" ]; then echo "OK      $f"; else echo "MISSING $f"; fi
done
```

**Expected:** seven lines, all starting with `OK`.

## 2.5 Read before you run

Everything in this repo will run on your computer: a launcher script and the recipes for two containers. Skimming them first is the cheapest security check there is, and it also shows you what each step will do.

```bash
# Page through a file. Space = next page, q = quit.
less bin/claude-box
less image/Containerfile
less proxy/squid.conf
```

What to look for:

- In `bin/claude-box`, the launcher mounts **only the folder you run it from**, and refuses to run from your home folder.
- In `image/Containerfile`, nothing runs as root at the end (`USER claude`).
- In `proxy/squid.conf`, the last access rule is `http_access deny all`, so anything not on the allowlist is refused.

## 2.6 Check where your copy came from

```bash
# Show the address this copy was downloaded from.
# Expected: "origin" followed by the address you cloned, listed twice (fetch and push).
git remote -v

# Show the five most recent changes, newest first.
git log --oneline -5
```

## 2.7 Getting updates later

```bash
# Download and apply the latest changes from the repository.
cd ~/claude-sandbox
git pull
```

If the update changed anything in `image/` or `proxy/`, rebuild the matching image (the steps that build them explain how).

If you have edited a file that comes with the repository, `git status` will list it as modified. That is normal. If `git pull` complains about your changes, send them to a safe place first with `git stash`, run `git pull`, then bring them back with `git stash pop`. Your own proxy allowlist additions are not a problem: they live in `proxy/allowlist.local.txt`, which git ignores (Step 7).

## Troubleshooting

| Symptom | Fix |
|---|---|
| `git: command not found` | Run `sudo apt install -y git` |
| `Repository not found` | Check the address for typos. A private repository needs an account or key |
| `Permission denied (publickey)` | You used the `git@github.com:...` address. Use the `https://...` address instead |
| `destination path ... already exists` | A `claude-sandbox` folder is already there. Rename it with `mv ~/claude-sandbox ~/claude-sandbox.old` and clone again |
| An entry shows `MISSING` in 2.4 | Run `git status` and `git log --oneline -5`. If the clone was incomplete, delete the folder and clone again |
