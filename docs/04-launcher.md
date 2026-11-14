# Install and test the launcher

## Description

Put the `claude-box` command on your computer and test it. The launcher starts a locked-down container for **the project in your current folder** and shows only that folder to the container.

**Heads-up about modes.** The launcher has two network modes:

- **Restricted** (the default): the container reaches the internet only through the allowlist proxy, which you build in Step 7.
- **Open** (`--open`): the container has full internet access. It prints a warning every time.

The proxy doesn't exist yet, so in this step you test with `--open`. That is fine for a smoke test in an empty folder. From Step 7 on, you will use restricted mode for real work.

## Before you start

- Step 3 is done: the image `claude-box:dev` exists (`podman images claude-box` shows it).

## Key terms

- **Launcher**: a small script (`bin/claude-box`) that runs the long `podman run` command for you, with all the safety options already filled in.
- **PATH**: the list of folders your shell searches when you type a command. A program in one of those folders can be run from anywhere.
- **Symlink**: a shortcut to a file. The shortcut is a few bytes, and opening it runs the real file.
- **Bind mount**: showing a folder from your computer inside the container. Changes made in it are real changes to your files.
- **Volume**: storage managed by Podman that outlives any one container. It holds Claude's login and history.

## 4.1 Review the launcher

You don't type anything in this section. The script is already in your clone:

```bash
cd ~/claude-sandbox

# Page through the script. Space = next page, q = quit.
less bin/claude-box
```

### The safety lines at the top

- `set -euo pipefail` makes the script stop at the first error, on any unset variable, or if a piped command fails, instead of carrying on half-working.
- `PROJECT="$(pwd -P)"` records the folder you ran the command from, with shortcuts resolved to the real path.
- The `case` block refuses to run from your home folder, `/`, or system folders. Without it, running `claude-box` from `~` would show your whole home folder to the container.

### The mode switch and the proxy block

- `--open` must be the first thing you type after `claude-box`. It switches to open mode and is removed from the arguments, so it isn't passed on to the container.
- The `if [[ "$MODE" == "restricted" ]]` block makes sure the proxy is running, looks up its address, and sets the proxy options. You can skip over it for now. Steps 6 to 8 explain it.

### The `podman run` options

| Option | What it does |
|---|---|
| `exec` | Replaces the script with the container command, so it ends when you exit the container |
| `--rm` | Deletes the container when you exit. The image and your volume stay |
| `-it` | Gives you an interactive terminal, so you can type into the container |
| `--name ...` | Names the container after your project. `$$` is the script's process number, which keeps names unique if you open two at once |
| `--userns=keep-id:uid=1000,gid=1000` | Maps **you** to the container's `claude` user (ID 1000). Files you create in the project belong to you on your computer. You can think of it as you wearing a mask: the container sees `claude`, your computer sees your own user |
| `--cap-drop=all` | Removes every special Linux privilege (changing network settings, file ownership, and so on) |
| `--security-opt=no-new-privileges` | Stops programs inside from gaining extra privileges |
| `--pids-limit=512` | Caps the number of processes, so a runaway program can't freeze your machine |
| `--memory=4g` | Caps the container's memory at 4 GB |
| `--tmpfs /tmp:...` | A scratch folder for temporary files, kept in memory and wiped on exit |
| `--read-only` | Locks the container's system files so nothing inside can change them |
| `--tmpfs /home/claude:...` | A scratch home folder in memory, wiped on exit and capped at 256 MB. Programs often need somewhere to write small files, and this gives them a safe place. Step 9 explains it in detail |
| `-e TERM` | Passes your terminal type through, so colors and arrow keys work |
| `-e CLAUDE_BOX_MODE=...` | Sets a variable inside the container so you can see which mode you're in |
| `"${NET_ARGS[@]}"` | Inserts the network options. In open mode the list is empty, so the container gets Podman's default network |
| `-v claude-state:/home/claude/.claude` | Mounts the volume that holds Claude's login and history. Podman creates it the first time you run the launcher |
| `-v "$PROJECT":"/workspace/$NAME"` | Shows your project folder inside the container. It is the **only** folder from your computer the container sees |
| `--workdir ...` | Starts you inside the project folder |
| `"$IMAGE" "$@"` | The image to start, then everything you typed after `claude-box`. With nothing typed, the image's default command opens a shell. With `claude`, it runs Claude Code |

The project is mounted at `/workspace/<project-name>`, not a single shared path. Claude files its sessions by folder, so a separate path per project keeps each project's sessions apart.

## 4.2 Put the launcher on your PATH

```bash
# Make sure the script is executable (it normally already is).
chmod +x ~/claude-sandbox/bin/claude-box

# Create the folder for personal commands, if it doesn't exist yet.
mkdir -p ~/.local/bin

# Create a shortcut (symlink) to the script in that folder.
# -s = symbolic link, -f = replace an existing one.
# Because it is a shortcut, updates to the script in the repo apply immediately.
ln -sf ~/claude-sandbox/bin/claude-box ~/.local/bin/claude-box

# Check that your shell can find it.
# Expected: /home/<you>/.local/bin/claude-box
command -v claude-box
```

To see that it is a shortcut, run `ls -l ~/.local/bin/claude-box`. It shows an arrow pointing to the real script in your repo.

If `command -v` prints nothing, `~/.local/bin` isn't on your `PATH` yet. Log out and back in, or run `export PATH="$HOME/.local/bin:$PATH"` for the current terminal.

## 4.3 Test it in open mode

Use an empty throwaway folder, not the repo:

```bash
mkdir -p ~/sandbox-test
cd ~/sandbox-test
claude-box --open
```

**Expected:** a line that starts with `WARNING: OPEN MODE`, then a prompt like `claude@abc123:/workspace/sandbox-test$`.

Run these inside the container, and compare each with the expected result:

```bash
pwd
# Expected: /workspace/sandbox-test

id
# Expected: uid=1000(claude) gid=1000(claude) ...

echo $CLAUDE_BOX_MODE
# Expected: open

ls /home
# Expected: claude   (you can't see your real home folder)

grep -E 'CapEff|NoNewPrivs' /proc/self/status
# Expected: CapEff all zeros (no special privileges), NoNewPrivs: 1

touch /usr/test
# Expected: "Read-only file system" (the system files are locked)

touch ~/scratch && echo ok
# Expected: ok (the home scratch folder is writable, but temporary)

touch hello && echo ok
# Expected: ok (your project folder is writable)

claude --version
# Expected: a version number followed by "(Claude Code)"

exit
```

Now check on your computer that the file made inside belongs to **you**:

```bash
# -n shows numeric IDs. The owner number should match the output of "id -u".
ls -ln ~/sandbox-test/hello
id -u
```

If the two numbers match, the mapping works: the container's `claude` user really is you, wearing a mask.

## 4.4 Test the safety checks

```bash
# 1. Running from your home folder must be refused.
# Expected: "Refusing to mount /home/<you>. cd into a project folder first."
cd ~ && claude-box --open

# 2. Arguments are passed through to the container.
# Expected: the Claude Code version, then it exits.
cd ~/sandbox-test && claude-box --open claude --version

# 3. Restricted mode (the default) needs the proxy, which doesn't exist yet.
# Expected: "Proxy container 'claude-proxy' not found. Build and run it first (guide step 7)."
# That message is correct. It means the safety check works.
claude-box
```

## 4.5 Updating later

The launcher is a shortcut to the script in your clone, so `git pull` updates it. No reinstall is needed.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `command not found: claude-box` | `~/.local/bin` isn't on your `PATH`. See 4.2 |
| An error mentioning `keep-id` or `--userns` | Your Podman is older than expected. Report the output of `podman --version` and the exact error |
| `unknown mount option` on a `--tmpfs` line | Same cause: an older Podman. Report `podman --version` and the exact error |
| A warning that memory or process limits are ignored | Rootless limits need systemd to pass resource controls to your user. It is harmless for now, so report the exact warning if you want it fixed |
| `Permission denied` writing in the project folder | The mapping isn't applied. Report the output of `ls -ln` on the project folder and of `id -u` |
| `the container name ... is already in use` | A container with that name is still running. Run `podman ps` to find it |
| `Refusing to mount ...` | You ran it from a broad folder (home, `/`, a system folder). `cd` into a project folder first |