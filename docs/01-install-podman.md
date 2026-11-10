# Install rootless Podman (Ubuntu 24.04 Host)

## Description

Install Podman, the tool that runs containers, and confirm it works without root. Rootless matters here: if something escapes the container, it lands as your unprivileged user, not as root on the host.

### 1.1 Install the packages

```bash
sudo apt update

# Install two packages:
#   podman  - the container engine itself (Ubuntu 24.04 ships the 4.9 series)
#   uidmap  - provides newuidmap/newgidmap, the helpers that let an
#             unprivileged user map extra user IDs into a container
#             (this is what makes "rootless" work)
sudo apt install -y podman uidmap
```

sudo is only needed here, for installing. After this step you run Podman as your normal user.

### 1.2 Verify the install

```bash
# Print the installed version.
# Expected: podman version 4.9.x
podman --version

# Ask Podman whether it is running rootless for your current user.
# Expected output: true
podman info --format '{{.Host.Security.Rootless}}'


# Check that your user has a range of "subordinate" IDs assigned.
# Containers use these as fake user IDs (see 1.4 for details).
# Expected: one line in each file, like  youruser:100000:65536
grep "^$USER:" /etc/subuid /etc/subgid
```

### 1.3 Run a first container

```bash
# Download an Ubuntu 26.04 image and run one command inside it:
#   run   - create and start a container
#   --rm  - delete the container automatically when the command finishes
#   docker.io/library/ubuntu:26.04 - the image; the full registry path
#           avoids any "which registry?" prompt
#   cat /etc/os-release - the command to execute inside the container
#
# The first run downloads the image, so it takes a moment.
# Expected: output that mentions Ubuntu 26.04
podman run --rm docker.io/library/ubuntu:26.04 cat /etc/os-release
```

### 1.4 Optional: see the rootless mapping

```bash
# Run a command inside Podman's user namespace and show its ID map.
# Expected output looks like:
#            0       1000          1
#            1     100000      65536
podman unshare cat /proc/self/uid_map
```