# cursor-remote-skills

Cursor Remote SSH fails when connecting to a dev server with slow or
restricted network access. The install script times out or can't acquire
the lock, leaving the connection stuck.

This repo has one runbook for that one problem.

## The error

```
Could not acquire lock after multiple attempts
disconnectReason: 'lock_acquisition_failed'
```

## Fix

See [SKILL.md](./SKILL.md)
for the full step-by-step, or follow the short version below.

**1. Kill the stale lock on the remote machine**

```bash
pkill -f cursor-server
rm -f /run/user/$(id -u)/cursor-remote-lock.*
```

Retry connecting. If it works, you're done.

**2. If it still fails — manually install the server**

Get your Cursor commit hash from **Help → About** on your local machine:

```
Commit: 0cf8b06883f54e26bb4f0fb8647c9500ccb43310
```

On the remote machine (Linux x64):

```bash
COMMIT=<your commit hash>
PLATFORM=linux-x64

curl -L "https://cursor.blob.core.windows.net/remote-releases/${COMMIT}/vscode-reh-${PLATFORM}.tar.gz" \
  -o /tmp/cursor-server.tar.gz

mkdir -p ~/.cursor-server/bin/${PLATFORM}/${COMMIT}

tar -xzf /tmp/cursor-server.tar.gz \
  -C ~/.cursor-server/bin/${PLATFORM}/${COMMIT} \
  --strip-components=1

~/.cursor-server/bin/${PLATFORM}/${COMMIT}/node --version
```

Then clear the lock and reconnect:

```bash
pkill -f cursor-server
rm -f /run/user/$(id -u)/cursor-remote-lock.*
```

**Remote platform options:** `linux-x64` `linux-arm64` `darwin-x64` `darwin-arm64`

## No internet on the remote machine?

Download on your local machine and scp over:

```bash
scp ~/cursor-server.tar.gz your-remote-host:/tmp/cursor-server.tar.gz
```

## License

MIT
