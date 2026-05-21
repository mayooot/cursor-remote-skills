---
name: cursor-ssh-fix
description: "Use this skill when a user reports Cursor Remote SSH connection failures. Key triggers: 'Could not acquire lock', 'lock_acquisition_failed', 'Cursor SSH not connecting', 'Cursor remote keeps failing', 'cant connect to remote server with Cursor'. Guides the user to share their Cursor version info, then diagnoses whether the issue is a stale lock, a missing/corrupt server binary, or both — and fixes it."
---

# Cursor Remote SSH: Connection Fix

## Overview

When Cursor connects to a remote server via SSH, it installs a `cursor-server` binary on the remote machine and locks a file during installation. If a previous session crashed or disconnected ungracefully, the lock file is left behind with a stale owner PID. New connection attempts keep waiting for the lock and eventually fail with:

```
Could not acquire lock after multiple attempts
disconnectReason: 'lock_acquisition_failed'
```

---

## Step 1 — Get the user's Cursor version info

Ask the user to open Cursor locally and go to **Help → About**, then share the output. You need:

- **Commit hash** (e.g. `0cf8b06883f54e26bb4f0fb8647c9500ccb43310`) — this is the remote server version to install
- **OS** (e.g. `Darwin arm64`) — tells you the user's *local* OS, but the remote server binary must match the *remote* machine's OS/arch

Example output to look for:
```
Version: 3.x.x
Commit: 0cf8b06883f54e26bb4f0fb8647c9500ccb43310
OS: Darwin arm64 ...
```

> The **Commit** hash is what matters. The user's local OS does NOT affect which server binary to download — always match the remote machine.

---

## Step 2 — Kill the stale lock on the remote machine

Have the user SSH into the remote machine and run:

```bash
# Find and kill any lingering cursor-server processes
pkill -f cursor-server

# Remove the stale lock file (TMP_DIR is usually /run/user/<uid>)
rm -f /run/user/$(id -u)/cursor-remote-lock.*
```

Then retry connecting from Cursor. **If it connects now, you're done.**

If it still fails (e.g. the server binary is missing or corrupt), continue to Step 3.

---

## Step 3 — Manually install the cursor-server binary

### Determine the remote platform

| Remote OS | Remote Arch | Use this suffix |
|-----------|-------------|-----------------|
| Linux     | x86_64      | `linux-x64`     |
| Linux     | arm64       | `linux-arm64`   |
| macOS     | x86_64      | `darwin-x64`    |
| macOS     | arm64       | `darwin-arm64`  |

Check on the remote machine if unsure:
```bash
uname -s   # Linux or Darwin
uname -m   # x86_64 or aarch64/arm64
```

### Download URL pattern

```
https://cursor.blob.core.windows.net/remote-releases/<COMMIT>/vscode-reh-<PLATFORM>.tar.gz
```

Example for Linux x64 with commit `0cf8b06...`:
```
https://cursor.blob.core.windows.net/remote-releases/0cf8b06883f54e26bb4f0fb8647c9500ccb43310/vscode-reh-linux-x64.tar.gz
```

### Option A — Remote machine has internet access (preferred)

Run on the remote machine:

```bash
COMMIT=<paste commit hash here>
PLATFORM=linux-x64   # change if needed

# Download
curl -L "https://cursor.blob.core.windows.net/remote-releases/${COMMIT}/vscode-reh-${PLATFORM}.tar.gz" \
  -o /tmp/cursor-server.tar.gz

# Create target directory
mkdir -p ~/.cursor-server/bin/${PLATFORM}/${COMMIT}

# Extract
tar -xzf /tmp/cursor-server.tar.gz \
  -C ~/.cursor-server/bin/${PLATFORM}/${COMMIT} \
  --strip-components=1

# Verify
~/.cursor-server/bin/${PLATFORM}/${COMMIT}/node --version
```

A node version printed (e.g. `v20.18.2`) means success.

### Option B — Remote machine has no internet access

Run on the **local machine** first:

```bash
COMMIT=<paste commit hash here>
PLATFORM=linux-x64   # match the REMOTE machine, not local

curl -L "https://cursor.blob.core.windows.net/remote-releases/${COMMIT}/vscode-reh-${PLATFORM}.tar.gz" \
  -o ~/cursor-server.tar.gz

# Transfer to remote (replace 'zagreus' with your SSH host alias)
scp ~/cursor-server.tar.gz zagreus:/tmp/cursor-server.tar.gz
```

Then SSH in and run the extract + verify steps from Option A.

---

## Step 4 — Clear the lock and reconnect

After the binary is in place, back on the remote machine:

```bash
rm -f /run/user/$(id -u)/cursor-remote-lock.*
pkill -f cursor-server
```

Then reconnect from Cursor on the local machine. It should find the pre-installed binary and skip the download entirely.

---

## Common mistakes

| Mistake | Fix |
|--------|-----|
| Downloading `cursor-linux-x64.tar.gz` instead of `vscode-reh-linux-x64.tar.gz` | The correct package name is `vscode-reh-<platform>` |
| Using local OS to pick the download platform | Always match the **remote** machine's OS/arch |
| Forgetting to delete the lock file after killing the process | Always `rm -f` the lock after `pkill` |
| Curl returns 215 bytes (XML error page) | Wrong URL — double-check the commit hash and platform suffix |

---

## Quick reference: full one-liner for Linux x64

```bash
COMMIT=0cf8b06883f54e26bb4f0fb8647c9500ccb43310 && \
PLATFORM=linux-x64 && \
pkill -f cursor-server; \
rm -f /run/user/$(id -u)/cursor-remote-lock.*; \
curl -L "https://cursor.blob.core.windows.net/remote-releases/${COMMIT}/vscode-reh-${PLATFORM}.tar.gz" -o /tmp/cursor-server.tar.gz && \
mkdir -p ~/.cursor-server/bin/${PLATFORM}/${COMMIT} && \
tar -xzf /tmp/cursor-server.tar.gz -C ~/.cursor-server/bin/${PLATFORM}/${COMMIT} --strip-components=1 && \
~/.cursor-server/bin/${PLATFORM}/${COMMIT}/node --version
```
