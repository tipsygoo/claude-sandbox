# Setup guide

Follow the steps in order. Each step ends with a "Done when" checklist;
don't move on until every box is ticked.

| Step | Topic | 
|------|-------|
| [1](01-install-podman.md) | Install rootless Podman |
| [2](02-create-repo.md) | Create the git repo |
| [3](03-containerfile.md) | Write the Containerfile and build the image |
| [4](04-launcher.md) | Create the `claude-box` launcher |
| [5](05-login.md) | Log in and test isolation |
| [6](06-internal-network.md) | Restrict outbound network access |
| [7](07-allowlist-proxy.md) | Build the allowlist proxy |
| [8](08-launcher-network.md) | Switch the launcher to the proxy, add open mode, close DNS leak |
| [9](09-hardening.md) | Further hardening: managed settings and read-only filesystem |
| [10](10-global-claude-md.md) | Global CLAUDE.md rules for every container |