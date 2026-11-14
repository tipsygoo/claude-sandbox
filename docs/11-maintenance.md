# Maintenance and backups

## Description

Keep the sandbox healthy over time, and make sure you can get everything back, especially your Claude sessions, after a reinstall or on a new computer.

## Before you start

- Steps 1 to 10 are done, and you have used Claude at least once, so there is something worth backing up.

## Key terms

- **Backup**: a copy of your data, kept somewhere else, that you can restore from.
- **Volume**: storage managed by Podman that outlives any container. `claude-state` holds your Claude data.
- **Export / import**: Podman's commands to save a volume into one file and to load it back.
- **tar file**: a single file that packs a whole folder tree. Your backup is a `.tar` file.
- **Prune**: a command that deletes things Podman considers unused. Useful, and dangerous in this project. See 11.7.

## 11.1 What needs maintaining

| What | Why | How often | Section |
|---|---|---|---|
| Back up `claude-state` and your local allowlist | They hold your sessions, login and personal rules, and they are **not** in git | Before a reinstall, and now and then | 11.2 |
| Update Claude Code | The image keeps the version it was built with | When a new release is out | 11.5 |
| Update this repo | Fixes and improvements | When you want them | 11.6 |
| Free disk space | Rebuilds leave old images behind | Occasionally | 11.7 |
| Review the proxy log | See what Claude tried to reach | Now and then | 11.8 |

## 11.2 Back up your data

### What to back up

| What | Where it lives | Why back it up |
|---|---|---|
| The `claude-state` volume | Managed by Podman on your computer | Your session history (what `claude -r` lists), your login, your personal `CLAUDE.md` and your Claude settings |
| `proxy/allowlist.local.txt` | Your clone. Git ignores it | It is your own list of allowed sites, and a fresh clone won't have it |
| Your projects | Their own repositories | They are not part of this project. Keep them in git as usual |

The repo itself is already safe if you pushed it, so it doesn't need a backup.

### Make a backup

```bash
# 1. Close every claude-box session first, then check that none are running.
# Expected: no containers whose name starts with "claude-box-" (the proxy may be listed).
podman ps

# 2. Create a private folder for backups.
#    chmod 700 means only you can open this folder.
mkdir -p ~/backups && chmod 700 ~/backups

# 3. Save the volume into a dated file.
#    $(date +%F) becomes today's date, for example 2026-11-14.
podman volume export claude-state -o ~/backups/claude-state-$(date +%F).tar

# 4. Make the file readable by you only.
chmod 600 ~/backups/claude-state-*.tar

# 5. Also copy your local allowlist, which git ignores.
cp ~/claude-sandbox/proxy/allowlist.local.txt ~/backups/allowlist.local-$(date +%F).txt
```

Check that the backup is real:

```bash
# List the first lines of the file's contents.
# Expected: file names such as .credentials.json and a "projects" folder.
tar -tf ~/backups/claude-state-$(date +%F).tar | head -n 20

# Show the file size.
ls -lh ~/backups/
```

The exact file names can differ between Claude Code versions. What matters is that the listing isn't empty.

### The backup is sensitive

It contains your **login token**, and your full conversation history, which can include code or secrets you pasted into a chat. So:

- Never commit it, never email it, and never put it in a shared folder.
- Keep it on an encrypted disk, or encrypt the file itself (below).

### Optional: encrypt the backup

```bash
# Install gpg if the command is missing.
sudo apt install -y gnupg

# Encrypt with a passphrase. It asks you to choose one, and creates
# claude-state-DATE.tar.gpg. Use a passphrase you will not forget.
gpg --symmetric --cipher-algo AES256 ~/backups/claude-state-$(date +%F).tar

# Remove the unencrypted copy once you have checked the encrypted one exists.
rm ~/backups/claude-state-$(date +%F).tar

# To decrypt later (it asks for the passphrase):
gpg --output ~/backups/claude-state-YYYY-MM-DD.tar --decrypt ~/backups/claude-state-YYYY-MM-DD.tar.gpg
```

If you lose the passphrase, the backup cannot be opened.

### Optional: keep only the newest backups

```bash
# List the backups, newest first (ls -1t), skip the 5 newest (tail -n +6),
# and delete the rest (xargs rm). Run it without "xargs ..." first to see what it would delete.
ls -1t ~/backups/claude-state-*.tar | tail -n +6 | xargs -r rm --
```

If you encrypt your backups, change `.tar` to `.tar.gpg` in the pattern.

## 11.3 Restore on a fresh machine

Do this after you have installed Podman (Step 1) and cloned the repo (Step 2).

```bash
cd ~/claude-sandbox

# 1. Put your local allowlist back BEFORE starting the proxy.
#    Adjust the file name to your backup's date.
cp ~/backups/allowlist.local-YYYY-MM-DD.txt proxy/allowlist.local.txt

# 2. Build the images, create the networks, start the proxy, and install the launcher.
#    Follow Steps 3, 6, 7 and 4 of the guide (or the Quick start in the README).

# 3. Restore the volume BEFORE the first claude-box launch.
#    If you already launched claude-box on this new machine, an empty "claude-state"
#    volume exists. Remove it first, but ONLY if it is new and holds nothing you need:
#        podman volume rm claude-state
podman volume create claude-state
podman volume import claude-state ~/backups/claude-state-YYYY-MM-DD.tar
```

Then bring your projects back with the **same folder names** they had before, for example by cloning them into the same places.

Check that everything came back:

```bash
cd ~/path/to/one-of-your-projects

# Expected: Claude starts without a login prompt.
claude-box claude
```

Exit, then run `claude-box claude -r`. **Expected:** the earlier sessions of that project are listed.

### Why the folder names matter

Claude files its sessions by the folder it was started in, and the launcher mounts every project at `/workspace/<folder name>`. A project folder with a different name is a different place to Claude, so its old sessions won't show up in `claude -r`. If you rename a project folder, either rename it back or accept that the earlier sessions stay with the old name.

### If the login is expired

A backup keeps the login from the day it was made. If Claude asks you to log in, or says the login expired, run `/login`. Your sessions and settings are not affected.

## 11.4 Test a restore safely

Don't wait for a disaster to find out whether your backup works. This test restores into a **separate** volume, so it cannot touch your real `claude-state`:

```bash
# 1. Create a test volume and load your backup into it.
podman volume create claude-state-test
podman volume import claude-state-test ~/backups/claude-state-YYYY-MM-DD.tar

# 2. Look inside it with a one-off container.
#    --userns=keep-id:...   same user mapping as the launcher
#    --network none         no network at all
#    :ro                    mounted read-only, so the test can't change anything
# Expected: your credentials file, owned by claude, and a "projects" folder.
podman run --rm --userns=keep-id:uid=1000,gid=1000 --network none \
  -v claude-state-test:/data:ro claude-box:dev ls -la /data

# 3. List the saved projects.
# Expected: one folder per project, named after its path, for example "-workspace-sandbox-test".
podman run --rm --userns=keep-id:uid=1000,gid=1000 --network none \
  -v claude-state-test:/data:ro claude-box:dev ls /data/projects

# 4. Delete the test volume.
podman volume rm claude-state-test
```

If step 2 shows your files and step 3 lists your projects, the backup is good.

## 11.5 Update Claude Code

The image keeps the Claude Code version it was built with. To check whether a newer one exists:

```bash
# Run a throwaway container as root and ask apt which version is installed
# and which is available. This changes nothing in the image.
podman run --rm --user root claude-box:dev bash -c 'apt-get update -qq && apt-cache policy claude-code'
```

If `Installed:` and `Candidate:` match, you are up to date. If they differ, rebuild without the saved layers:

```bash
cd ~/claude-sandbox

# --no-cache ignores saved layers, so the Claude Code install step runs again.
podman build --no-cache -t claude-box:dev image/
```

Your sessions and login are in the volume, so a rebuild doesn't touch them.

## 11.6 Update this repo

```bash
cd ~/claude-sandbox
git pull
```

Then rebuild only what changed:

| What `git pull` changed | You need to |
|---|---|
| `image/Containerfile`, `image/managed-settings.json`, or `image/CLAUDE.md` | `podman build -t claude-box:dev image/` |
| `proxy/squid.conf` or `proxy/Containerfile` | `podman build -t claude-proxy:dev proxy/`, then `podman rm -f claude-proxy` and the `podman run` from Step 7 (7.4) |
| `proxy/allowlist.txt` | `podman restart claude-proxy` |
| `bin/claude-box` | Nothing. The launcher takes effect on the next launch |
| Anything in `docs/` or `README.md` | Nothing |

`git diff --stat HEAD@{1}` right after a pull shows which files changed.

## 11.7 Free disk space, and what never to prune

Rebuilding the image leaves the previous build behind as an unnamed ("dangling") image. To see how much space Podman uses and to clean those up:

```bash
# Show how much space images, containers and volumes use.
podman system df

# List images. Old builds appear with <none> as their name.
podman images

# Remove the unnamed leftovers. This is safe.
podman image prune
```

Also check your disk with `df -h ~`.

### Commands that are dangerous in this project

Podman's `prune` commands delete whatever is not in use **at that moment**. In this project, that is often something you need:

| Command | What it would delete here |
|---|---|
| `podman volume prune` | `claude-state`, whenever no session is open: your login, sessions, and personal rules |
| `podman system prune --volumes` | The same, plus more |
| `podman system prune` | A stopped `claude-proxy`, and the networks `claude-internal` and `claude-external` when nothing is attached |
| `podman container prune` | A stopped `claude-proxy` |

**Don't run these.** The commands that are safe here are `podman image prune` and removing a specific thing by name.

If a prune removed the proxy or the networks, rebuild them with Steps 6 and 7. If it removed `claude-state`, restore it from a backup (11.3).

## 11.8 Review what Claude reaches

```bash
# Show the last 50 requests the proxy handled.
podman exec claude-proxy tail -n 50 /var/log/squid/access.log
```

- `TCP_TUNNEL/200` lines are allowed requests.
- `TCP_DENIED/403` lines are blocked ones. A site that keeps showing up and that you actually need is a candidate for `proxy/allowlist.local.txt` (Step 7). A site you don't recognize is worth a closer look.

## 11.9 A simple routine

| When | What to do |
|---|---|
| Before reinstalling your OS or replacing your computer | Make a backup (11.2) and test it (11.4) |
| About once a month | Back up (11.2), check for a Claude Code update (11.5), `git pull` (11.6), `podman image prune` (11.7), glance at the proxy log (11.8) |
| After a long or important session you would hate to lose | Back up (11.2) |

## Troubleshooting

| Symptom | Fix |
|---|---|
| `podman volume export` fails or the file looks empty | Close all `claude-box` sessions and retry. Check that `claude-state` exists with `podman volume ls` |
| `podman volume create claude-state` says it already exists | A volume with that name is already there. If it is a new empty one, remove it first (11.3). If it holds data you need, stop and back it up first |
| `podman volume import` fails | Copy the exact error. Make sure the file is a real backup (`tar -tf FILE | head`) and that you are importing into a volume that exists |
| `claude -r` doesn't list old sessions after a restore | The project folder name differs from the one used when the sessions were made. Check it with `ls /data/projects` in the test from 11.4 |
| A login prompt appears after restoring | The saved login has expired. Run `/login` |
| The backup file is very large | Long histories add up. Check the size with `ls -lh ~/backups/`, and prune old backups (11.2) |
| After a prune, `claude-box` says `Proxy container 'claude-proxy' not found` | Rebuild the networks and the proxy: Steps 6 and 7 |
