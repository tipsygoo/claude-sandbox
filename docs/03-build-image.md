# Step 3: Build the image

## Description

Turn the recipe in `image/Containerfile` into an image named `claude-box:dev`, and check that it works.

## Before you start

- Step 1 (Podman installed) and Step 2 (repo cloned to `~/claude-sandbox`) are done.
- You have internet access, because the build downloads packages and Claude Code.

## Key terms

- **Image**: a packed, read-only template holding an operating system and software, like a frozen computer setup.
- **Container**: a running copy started from an image. You can start many from one image, and they are disposable.
- **Containerfile**: the text recipe Podman follows to build an image. (Docker calls the same thing a Dockerfile.)
- **Layer**: every step of the recipe adds a layer. Podman reuses layers that haven't changed, which makes rebuilds fast.
- **Tag**: the label after the colon in an image name. In `claude-box:dev`, `claude-box` is the name and `dev` is the tag.

## 3.1 Review what you are about to build

You don't type anything in this section. The recipe is already in your clone, so open it and read along:

```bash
cd ~/claude-sandbox

# Page through the recipe. Space = next page, q = quit.
less image/Containerfile
```

A Containerfile is read from top to bottom, and each `RUN` or `COPY` adds a layer.

**`CMD ["bash"]`**
The default command when a container starts: open a shell. Anything you type after the image name replaces it, so `podman run claude-box:dev claude` runs Claude Code instead. There is deliberately no fixed program, so the same image can open a shell or run any tool.

## 3.2 Build the image

```bash
cd ~/claude-sandbox

# Build an image from the Containerfile in the image/ folder:
#   -t claude-box:dev   the name:tag to give the result
#   image/              the "build context": the only folder the build can see
#                       (it holds the Containerfile and the two files it copies)
#
# The first build takes a few minutes. Each step is printed as it runs.
podman build -t claude-box:dev image/
```

**Expected:** the output ends without errors, and the last line shows the new image's ID.

## 3.3 Verify

```bash
# 1. The image is listed. Expected: a line for localhost/claude-box with tag "dev".
podman images claude-box

# 2. Claude Code is installed. "claude --version" replaces the default "bash" command.
#    Expected: a version number followed by "(Claude Code)".
podman run --rm claude-box:dev claude --version

# 3. The container user is "claude", not root.
#    Expected: uid=1000(claude) gid=1000(claude) ...
podman run --rm claude-box:dev id

# 4. The other tools are present. Expected: a version line for each.
podman run --rm claude-box:dev bash -c 'git --version && python3 --version && rg --version | head -1'

# 5. The policy files are in place and owned by root.
#    Expected: CLAUDE.md and managed-settings.json, both owned by root.
podman run --rm claude-box:dev ls -l /etc/claude-code/
```

`--rm` deletes each test container as soon as it finishes. The image stays.

## 3.4 About the image

- **Where it lives:** in your home folder, under `~/.local/share/containers/storage`. List images with `podman images`.
- **The name `claude-box:dev`** is the one the launcher in Step 4 uses. If you build under a different name, the launcher will not find it.
- **Images are disposable.** You can delete one with `podman rmi claude-box:dev` and rebuild it at any time from the Containerfile.

## 3.5 Updating later

The image is a snapshot, so Claude Code inside it stays at the version from the day you built it. To check whether a newer release exists:

```bash
# Run a throwaway container as root and ask apt which version is installed
# and which is available. This changes nothing in the image.
podman run --rm --user root claude-box:dev bash -c 'apt-get update -qq && apt-cache policy claude-code'
```

If the `Installed:` and `Candidate:` lines match, you are up to date. If they differ, rebuild without the saved layers:

```bash
# --no-cache ignores saved layers, so the Claude Code install step runs again.
podman build --no-cache -t claude-box:dev image/
```

Also rebuild whenever you change `image/Containerfile`, `image/managed-settings.json`, or `image/CLAUDE.md`, because those are copied into the image at build time.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `useradd: UID 1000 is not unique` | Another user holds ID 1000 in the base image. Run `podman run --rm docker.io/library/ubuntu:26.04 getent passwd 1000` and report the output |
| apt complains about the repository key (`NO_PUBKEY` or "not signed") | Copy the exact error. The first thing to try is adding `gnupg` to the package list in the first `RUN` |
| `curl` fails during the build | Check your internet connection and retry |
| `Unable to locate package claude-code` | The repository line did not take effect. Check the build output around that step |
| `COPY failed: file not found` | You built from the wrong folder. Run the command from `~/claude-sandbox` and end it with `image/` |
| Build stops with "no space left on device" | Free disk space (`df -h ~`), then rebuild |
