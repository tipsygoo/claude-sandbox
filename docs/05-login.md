# Log in to Claude Code and verify it persists

## Description

Log in to Claude Code once from inside the container, confirm the login survives the container being deleted, and confirm that different projects keep separate session lists for `claude -r`.

**Why every command here uses `--open`.** The launcher's default mode (restricted) reaches the internet only through the allowlist proxy, and you build that in Step 7. Until it exists, restricted mode refuses to start. So in this step you use `claude-box --open`, which gives the container full internet access. Your login is saved, so once the proxy is ready you switch to restricted mode and do **not** need to log in again.

## Before you start

- Step 4 is done: `claude-box` is on your `PATH` and `claude-box --open` opens a shell.
- You have a Claude account (a Pro, Max, Team or Enterprise subscription, or a Claude Console account).
- You are using an **empty test folder**, not a real project. In open mode nothing limits where data inside the container can be sent, and your login token is inside it, so only run trusted things during this step.

## Key terms

- **Login token**: the secret your account hands to Claude Code after you sign in, so you don't sign in every time.
- **Volume**: storage managed by Podman that outlives any one container. Here it is called `claude-state`, and it holds your login and session history.
- **Session**: one conversation with Claude, saved so you can come back to it with `claude -r`.

## How login works inside a container

The container has no web browser. Claude Code handles this: it prints a login address, you open that address in the browser on **your own computer**, and the browser shows you a code to paste back into the terminal.

Your login token is saved in a file called `.credentials.json` inside `~/.claude` in the container, which is the `claude-state` volume. So the login lives in the volume, not in the image and not in your project.

## 5.1 Start Claude Code

```bash
# Use an empty test folder, not a real project.
mkdir -p ~/sandbox-test && cd ~/sandbox-test

# Run Claude Code inside the container (open mode, because the proxy doesn't exist yet).
# "claude" is passed through to the container and replaces its default "bash" command.
claude-box --open claude
```

**Expected:** a `WARNING: OPEN MODE` line, then Claude Code's first-run setup with a login prompt. Choose the account type you use. For most people that is **Claude account with subscription**.

## 5.2 Complete the login

1. Claude Code prints a login address. Select it with your mouse and copy it from the terminal.
2. Paste the address into the browser on **your own computer** and sign in.
3. The browser shows a code. Copy it.
4. Paste it into the terminal at the `Paste code here if prompted` prompt, then press `Enter`.
5. The terminal shows `Login successful`. Press `Enter` to continue.

Pressing `c` at the login screen copies the address to your clipboard, but that may not work inside a container, so selecting the text is the reliable way.

## 5.3 Verify the login

Inside Claude Code, type:

```
/status
```

**Expected:** your account (email and organization) and the login method. The `Setting sources` line should also mention `Enterprise managed settings (file)`. That is the policy file baked into the image (Step 9).

Exit with `/exit` (or `Ctrl+D`).

## 5.4 Check the credentials file

```bash
# Open a shell in the container (no arguments = the default bash).
claude-box --open
```

Inside the container:

```bash
# List the file's permissions.
# Expected: -rw------- (only its owner can read it) and owner "claude".
# Do NOT print its contents: it holds your login token.
ls -l ~/.claude/.credentials.json

exit
```

## 5.5 Test persistence

```bash
# The previous container was deleted when you exited (the launcher uses --rm).
# Start a brand new one.
claude-box --open claude
```

**Expected:** Claude Code starts **without** asking you to log in. That proves the login lives in the `claude-state` volume and not in the container. Exit again with `/exit`.

Also confirm on your computer that the volume exists and see where it is stored:

```bash
# Expected: a line with "claude-state"
podman volume ls

# Shows the folder where Podman keeps the volume's files on your computer.
podman volume inspect claude-state --format '{{.Mountpoint}}'
```

## 5.6 Test separate sessions per project

```bash
mkdir -p ~/sandbox-a ~/sandbox-b

# Project A: send one message, for example "say hello", then /exit
cd ~/sandbox-a && claude-box --open claude

# Project B: send one message, for example "say goodbye", then /exit
cd ~/sandbox-b && claude-box --open claude

# Back in A: open the resume picker.
# Expected: it lists the "hello" session only, not B's.
cd ~/sandbox-a && claude-box --open claude -r
```

This works because the launcher mounts each project at its own path (`/workspace/sandbox-a`, `/workspace/sandbox-b`), and Claude files its sessions by folder.

## Useful commands

| Command | What it does |
|---|---|
| `/status` | Shows which account and credential are active |
| `/logout` | Logs out and removes the stored login |
| `/login` | Logs in again, for example when the login expires |
| `claude-box --open claude -r` | Picks an earlier session of the current project to continue |

## Back up, move, or reset your login and history

Everything in `~/.claude` lives in the `claude-state` volume: your login, your session history, and your personal instructions file (`~/.claude/CLAUDE.md`) if you create one in Step 10. Podman has built-in commands to save and restore a volume:

```bash
# BACK UP: save the whole volume into one file.
podman volume export claude-state -o ~/claude-state-backup.tar

# RESTORE on a fresh machine: create an empty volume, then load the backup into it.
podman volume create claude-state
podman volume import claude-state ~/claude-state-backup.tar

# RESET: delete the volume, including the login and all session history.
# Export first if you want to keep anything. This cannot be undone.
podman volume rm claude-state
```

**Keep that backup file private.** It contains your login token. Do not share it and do not commit it to a repository. Re-logging in is also easy, so a backup is only worth keeping if you want your session history.

## Security notes

- The `claude-state` volume holds your login token, so anything running inside **any** `claude-box` container can read it. Only run Claude on projects you trust, and keep Claude's permission prompts on.
- Variables on your computer, such as `ANTHROPIC_API_KEY`, are **not** passed into the container, because the launcher only passes `TERM` and its own settings. That is on purpose: the fewer secrets inside the container, the less there is to lose.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Pasting the code doesn't work | Try right-click paste or `Ctrl+Shift+V`. If it still fails, exit, run `claude-box --open` for a shell, then run `claude auth login` there |
| Login fails with an OAuth or redirect error | The browser flow in containers has had bugs in some versions. Copy the exact error and the output of `claude --version` |
| Asked to log in again every time | The volume isn't persisting. Check that `podman volume ls` shows `claude-state` and that `ls -ld ~/.claude` inside the container shows owner `claude` |
| `Permission denied` writing under `~/.claude` | The volume's ownership is wrong. If you have nothing to keep, remove it with `podman volume rm claude-state` and let the launcher recreate it |
| Claude can't connect | In open mode this means your internet connection. Check it, then retry |
| The login expires later | Run `/login` again. Claude Code warns you about three days before it expires |