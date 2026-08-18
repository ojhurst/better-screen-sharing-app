# Better Screen Sharing

A config-driven VNC viewer for macOS that replaces Apple Screen Sharing when you
need to *watch a machine an automation is driving* — including headless Linux
boxes running a browser under Xvfb.

Embedded noVNC live view, plus a self-managed SSH tunnel that re-dials itself.
Point it at a target and it shows up.

> **This repository contains documentation, issue tracking, and releases only.**
> The application source is not published here. Downloads are on the
> [Releases](https://github.com/ojhurst/bss/releases) page.

---

## Why it exists

Apple Screen Sharing cannot manage an SSH tunnel. Point it at a headless display
over a forwarded port and it fails with "Connection failed to localhost" the
moment the saved port goes stale — with nothing to re-dial the tunnel and no way
to re-point it from a script.

Better Screen Sharing owns the whole path: it brings up the tunnel, optionally
starts `x11vnc` on the remote display, proxies the VNC stream into an embedded
noVNC client, and reconnects when the tunnel drops. Every target lives in one
JSON file, so switching machines is a one-line change — or one command.

## Install

1. Download the latest `Better-Screen-Sharing-<build>-arm64.zip` from
   [Releases](https://github.com/ojhurst/bss/releases).
2. Unzip and drag **Better Screen Sharing.app** to `/Applications`.
3. First launch: **right-click the app → Open**, then confirm.

Step 3 is required once. The app is signed with a Developer ID certificate but
is not yet notarized, so Gatekeeper asks for explicit consent on first launch.
Right-click → Open is the supported path; after that it opens normally. If you
would rather clear the quarantine flag directly:

```bash
xattr -dr com.apple.quarantine "/Applications/Better Screen Sharing.app"
```

Apple Silicon only (arm64).

### The `bss` CLI

The command-line tool ships inside the bundle. Put it on your `PATH`:

```bash
ln -sf "/Applications/Better Screen Sharing.app/Contents/Resources/app.asar.unpacked/bin/bss" \
  /usr/local/bin/bss
```

## Use

```bash
bss list                  # show targets and which one is active
bss view studio           # point the window at "studio" and bring it to the front
bss add "My VPS" vps ssh  # add an SSH-tunneled target
bss logs 40               # what just happened
bss update --check        # installed vs. latest build
```

## Configure

Everything lives in one file:

```
~/Library/Application Support/better-screen-sharing/targets.json
```

The running app watches it and refreshes within about a second, so editing the
file *is* the control interface — change `active` and the live view follows.

```json
{
  "active": "headless-browser",
  "wsPort": 6080,
  "targets": {
    "studio": {
      "label": "Mac Studio",
      "type": "direct",
      "host": "studio.local",
      "port": 5900
    },
    "headless-browser": {
      "label": "Headless browser (Xvfb :99)",
      "type": "ssh",
      "ssh": "my-server",
      "remotePort": 5900,
      "localPort": 5901,
      "display": 99,
      "ensureX11vnc": true
    }
  }
}
```

- **`direct`** — straight VNC to a reachable host.
- **`ssh`** — tunnel to a remote display via an alias from `~/.ssh/config`.
  `localPort` must be unique per target. Set `ensureX11vnc` to start the VNC
  server on the remote display before connecting.

Full schema, troubleshooting, and the complete CLI surface are in
[AGENTS.md](AGENTS.md).

## Driving it from an AI agent

This tool was built to be operated by an AI coding agent, not just a person.
[AGENTS.md](AGENTS.md) is written for exactly that: point Claude Code (or any
agent that reads `AGENTS.md`) at this repository, or at the copy shipped inside
the app bundle at `Contents/Resources/AGENTS.md`, and it will know how to add a
target, re-point the view, read the logs, and diagnose a failed connection —
without any access to the source.

The short version: there is no API to learn. The agent edits one JSON file, and
the window follows.

## Interface

- **Connections panel** — left sidebar, one row per target. The top bar's left
  button collapses it to give the remote more room; the top bar itself never
  hides, so that toggle is always available.
- **Full screen** — ⌃⌘F, the View menu, or the green traffic light. A thin blue
  frame marks the edge and a slide-down bar appears at the top edge for status
  and exit.
- **Display menu** — rearranges the *remote* Mac's displays over SSH (mirror,
  extend, single). Mirroring collapses multiple monitors into one, which is much
  easier to drive through a viewer.
- **File → Open Log / Check for Updates**.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `ssh exited (code 255) before establishing` | Stale tunnel holding the port. Usually auto-cleared; else `pkill -f "ssh -N.*-L 590"`. |
| Black screen, no `rfb connect` in the log | No VNC server on the remote. Start `x11vnc` on the expected port. |
| No window opens at all | `ELECTRON_RUN_AS_NODE` leaked from an Electron-hosted shell. Launch with `env -u ELECTRON_RUN_AS_NODE open -a "Better Screen Sharing"`. |
| Config edits do nothing | You edited the seed inside the `.app` instead of the live file in `~/Library/Application Support/`, or the JSON no longer parses. |

Log file: `~/Library/Logs/better-screen-sharing.log`.

## Bugs and feature requests

[Open an issue](https://github.com/ojhurst/bss/issues/new/choose). Please include
the build number (`bss update --check`), your macOS version, the connection type,
and a `bss logs 60` excerpt — **scrubbed of hostnames, IPs, usernames, and any
credential paths.**

## License

The documentation in this repository is MIT licensed — see [LICENSE](LICENSE).
The application binary is distributed free of charge, as-is and without warranty;
its source is not published.
