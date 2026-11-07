# Claude Sandbox

A rootless Podman setup for running Claude Code in an isolated container.
Only the project folder you launch it from is visible to Claude; everything
runs as an unprivileged user inside the container.

## Dependencies

Install these on the **host** machine before starting the guide.

| Requirement | Version | Why it's needed |
|-------------|---------|-----------------|
| Ubuntu | 24.04 LTS (tested) | Host OS. Other Linux distributions with Podman should work but are untested |
| Podman | 4.9 or newer | Runs the container without root |
| uidmap | any | Lets rootless Podman map user IDs into the container |
| git | any | Clones this repo and tracks changes to it |
| Claude account | - | A Claude subscription or Anthropic Console account to log in to Claude Code |
| Internet access | - | Downloads the base image and Claude Code during the build |

Step 1 of the guide installs Podman and uidmap.

### Provided by the container image

These are installed inside the image, so you do **not** install them on the host:
Ubuntu 26.04 base, Claude Code, git, ripgrep, jq, and Python 3.

## Layout

- `docs/` - the step-by-step setup guide, start at [docs/README.md](docs/README.md)
- `image/Containerfile` - image recipe (added in Step 3)
- `bin/claude-box` - launcher script (added in Step 4)

## Rebuild on a fresh machine

1. Install the dependencies above (guide Step 1).
2. Clone this repo.
3. Build the image from `image/Containerfile` (guide Step 3).
4. Copy `bin/claude-box` into `~/.local/bin` (guide Step 4).
5. Log in again. Your login is stored in a Podman volume on the machine, not in this repo.

## What is not in this repo

Logins, API keys, tokens, and `.env` files. See `.gitignore`.
