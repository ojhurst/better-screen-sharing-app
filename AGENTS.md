# Better Screen Sharing — agent operating notes

You are an AI agent (Claude Code or similar) on a Mac that has **Better Screen
Sharing** installed. This file is everything you need to drive it. You do **not**
have, and do not need, the application source.

Read this before you try to show a human a remote machine's screen.

---

## What it is

A config-driven VNC viewer (Electron + noVNC) with a self-managed SSH tunnel. It
replaces Apple Screen Sharing for watching machines that an agent is driving —
especially headless Linux boxes running a browser under Xvfb.

**Use this instead of `open vnc://…`.** Apple's client cannot manage the tunnel,
fails opaquely on a stale port, and gives you no way to re-point it from a script.
This app makes the view a scriptable, first-class thing.

---

## The one thing to understand: the config IS the API

There is no REST API and no GUI automation. **You control the app by editing one
JSON file.** The running app watches it (~1 second poll) and live-refreshes its
Connections panel, its menu, and — if `active` changed — the picture on screen.

```
~/Library/Application Support/better-screen-sharing/targets.json
```

That is the live, writable config that both the app and the `bss` CLI read. Edit
that path. A copy inside the `.app` bundle is only the seed used on first run;
the bundle is read-only and editing it does nothing.

```json
{
  "active": "headless-browser",
  "wsPort": 6080,
  "targets": {
    "<key>": { "...connection..." }
  }
}
```

- `active` — key of the connection currently shown in the live view.
- `wsPort` — local noVNC websocket port. Leave it alone.
- `targets` — map of stable slug → connection object.

Change `active`, and within about a second the human's window is looking at that
machine. That is the whole trick.

---

## Connection shapes

There are two `type`s. Always write `type` explicitly, even though it is inferred
when omitted (`ssh` if an `ssh` key is present, else `direct`).

### `direct` — straight VNC to a reachable host

```json
"studio": {
  "label": "Mac Studio",
  "type": "direct",
  "host": "studio.local",
  "port": 5900,
  "username": "",
  "password": ""
}
```

- `host` — hostname, `.local` mDNS name, IP, or an SSH alias that resolves.
- `password` — plaintext. LAN convenience only. See "Passwords" below before
  you write a real one into this file.

### `ssh` — tunnel to a remote display, optionally starting the VNC server first

```json
"headless-browser": {
  "label": "Headless browser (Xvfb :99)",
  "type": "ssh",
  "ssh": "my-server",
  "remotePort": 5900,
  "localPort": 5901,
  "display": 99,
  "ensureX11vnc": true,
  "ensureDesktop": false
}
```

- `ssh` — an alias from `~/.ssh/config`. **Prefer an alias over a raw IP**; LAN
  IPs rotate on DHCP and a hardcoded one will rot.
- `remotePort` — where the VNC server listens on the remote.
- `localPort` — **must be unique across all `ssh` targets.** Each gets its own
  tunnel. When adding one, use `max(existing localPorts) + 1`.
- `display` — X display number. Only consulted when `ensureX11vnc` is true.
- `ensureX11vnc` — start `x11vnc` on the remote display before tunneling. Set
  true for headless Linux; false for a Mac that already runs a VNC server.
- `ensureDesktop` — headless Linux only. Ensures a window manager, taskbar, and
  wallpaper exist on the remote display, so there is something to actually drive.

### Rules when you write the JSON by hand

1. `localPort` unique per `ssh` target. Collisions produce "Address already in use".
2. SSH alias over raw IP.
3. Key is a stable slug — lowercase, hyphens. It is what you pass to `bss view`.
4. Write valid JSON. A malformed file means the app cannot re-point at all.

---

### Same-Mac, different-user virtual sessions

You can view and control **another macOS account on the machine you are already
sitting at**, while your own session keeps the physical console. macOS gives a
non-console user its own virtual desktop — separate cursor, keyboard, and browser
profile — so an agent can drive a browser with this machine's network identity
without stealing your mouse.

```json
"agent-session": {
  "label": "Agent (virtual session)",
  "type": "direct",
  "host": "localhost",
  "port": 5900,
  "username": "agentuser",
  "password": "...",
  "nativeViewer": false,
  "mode": "local-user-session"
}
```

`mode: "local-user-session"` is optional but recommended. It declares intent, and
the app then checks the two preconditions below **at connect time** and tells you
immediately if they are not met — instead of connecting successfully and showing
you a black rectangle. Leaving it off changes nothing else.

Two details do the work:

- **`host: "localhost"`.** No relay, no LAN address, no tunnel. Connect straight
  to the local Screen Sharing port.
- **`nativeViewer: false`.** This is the important one. Apple's Screen Sharing
  client refuses same-host connections with *"You cannot control your own
  screen"* — that check lives in **Apple's client, not in this app**. Setting
  `nativeViewer: false` keeps you on the embedded viewer, which has no such
  check. Routing through an external relay to defeat the message is unnecessary
  and changes what macOS decides to serve you.

**Prerequisites, both required:**

1. **The target account must already be logged in** via Fast User Switching.
   Screen Sharing will answer for a user with no session, but there is no
   desktop behind it.
2. **The target account must NOT be the one holding the physical console.**
   macOS will not build a virtual desktop for a user who already has the real
   one, and will not mirror the console over this connection either.

Check both:

```bash
stat -f "%Su" /dev/console   # who holds the console
who                          # which accounts have live sessions
```

If the app connects and stays black, it now names which of these it is — see
Troubleshooting.

## CLI — `bss`

Thin, safe wrappers over the same file. Prefer these over hand-editing JSON; they
handle unique-port allocation and key collisions for you.

```bash
bss list                                    # targets + which is active
bss view <key>                              # make <key> active, launch if needed, raise to front
bss view <key> --no-focus                   # same, but do not steal focus
bss add "<Label>" <address> [direct|ssh] [port]
bss rm <key>                                # delete a connection
bss start                                   # launch the app
bss stop                                    # quit it
bss logs [-f|N]                             # tail the log
bss update [--check|--latest]               # update to the newest published build
```

`bss view` deliberately raises the window to the front. When an agent says "look
at this," the window has to actually come to the human — a re-pointed window
buried behind three others helps nobody. Use `--no-focus` for scripted or batch
re-pointing where you are not asking anyone to look.

If `bss` is not on your `PATH`, it lives inside the bundle:

```bash
"/Applications/Better Screen Sharing.app/Contents/Resources/app.asar.unpacked/bin/bss"
```

---

## The workflow you will actually run

Human asks to see what your automation is doing on a remote machine:

```bash
bss list                  # find the key for that machine
bss view headless-browser       # point the window there and raise it
bss logs 20               # confirm it connected
```

A healthy connect logs a line like:

```
rfb connect headless-browser fb=1366x900
```

`fb=WxH` is the remote framebuffer. If you see it, the human is looking at the
remote screen right now. Say so out loud — do not make them guess whether the
window updated.

If the target does not exist yet:

```bash
bss add "Headless Browser" my-server ssh 5900     # creates an ssh target with a free localPort
bss view headless-browser
```

Then set `ensureX11vnc: true` and `display` in the JSON if it is a headless
Linux display.

---

## Logs — always check these before guessing

```
~/Library/Logs/better-screen-sharing.log
```

Timestamped, and the packaged app has no terminal, so this file is the only
window into what happened. `bss logs`, `bss logs -f`, `bss logs 200`. The app's
File menu → **Open Log** opens it in the GUI.

Captured: start banner, connect lifecycle, the exact `ssh … -L` command, orphaned
tunnel kills, tunnel stderr and exit codes, errors.

---

## Troubleshooting

**Connected, authenticated, and the screen stays black**
For a local target with a `username`, the app asks macOS directly and tells you
which case it is instead of guessing:

- *"<user> is signed in at the physical console right now"* — that account holds
  the real desktop, so there is no virtual one to send. Switch it out at the
  login window, or target a different user.
- *"<user> is not logged in on this Mac"* — log in once via Fast User Switching,
  then reconnect.
- *"session looks right"* — the account is logged in and is not the console
  user, so this really is a sleeping display or an encoding problem.

Older builds reported every one of these as *"not sending video, or an encoding
mismatch"*, which pointed at the wrong problem. If you see that wording, update.

**`tunnel ssh exited (code 255) before establishing`**
A stale tunnel is holding the `localPort`. The app clears these automatically
before each connect — look for `freed orphaned tunnel(s)` in the log. If it
persists, kill them yourself:

```bash
pkill -f "ssh -N.*-L 590"
```

**Black screen, no `rfb connect` line**
No VNC server on the remote. For a headless Linux display, confirm x11vnc is up
on the port your target expects:

```bash
ssh <alias> 'pgrep -a x11vnc'
ssh <alias> 'export DISPLAY=:99; x11vnc -display :99 -forever -shared -nopw -rfbport 5900 -bg'
```

**Do not relocate a VNC server off the port a target expects.** If a target says
`remotePort: 5900`, moving the server to another port silently kills the human's
live view. Change one or the other, never just one.

**The window opens but shows nothing / no window appears at all**
`ELECTRON_RUN_AS_NODE` leaked in from an Electron-hosted shell — VS Code and
Claude Code both set it. It makes the GUI app run as headless Node: it logs a
little, opens no window, and exits. The `bss` CLI already clears it. If you are
launching the app some other way, clear it yourself:

```bash
env -u ELECTRON_RUN_AS_NODE open -a "Better Screen Sharing"
```

**Connections panel did not update after you edited the JSON**
You almost certainly edited the seed inside the `.app` bundle instead of the live
config in `~/Library/Application Support/`. Check which file you wrote. Also
verify the JSON parses — a syntax error stops the watcher cold.

---

## Passwords

Two options on a connection:

- `password` — plaintext in the config file. Fine for a throwaway LAN box, bad
  for anything else. Never commit it anywhere.
- `passwordSecret` — an opaque string resolved at connect time by an external
  helper, so the real credential never sits in the config file. The helper is
  whatever the **top-level `passwordCommand`** setting names, invoked as:

  ```
  <passwordCommand> <passwordSecret>
  ```

  Whatever it prints on stdout becomes the password. `passwordCommand` defaults
  to `bss-resolve-secret` on your `PATH`. Point it at your own secrets tool —
  a password manager CLI, a cloud parameter-store lookup, anything that prints
  a secret for a given key:

  ```json
  {
    "wsPort": 6080,
    "passwordCommand": "/usr/local/bin/my-secret-lookup",
    "targets": {
      "some-box": {
        "label": "Some box",
        "type": "direct",
        "host": "some-mac.local",
        "username": "admin",
        "passwordSecret": "vnc/some-box"
      }
    }
  }
  ```

  If the helper is missing or errors, the lookup fails and the password is
  empty — check the log for `password secret error`.

**As an agent: do not write a human's password into this file on your own
initiative.** Ask first, or use `passwordSecret`.

---

## Updating

```bash
bss update --check      # installed=<n> latest=<n> behind=<0|1>
bss update              # download and install the newest published build, relaunch
```

Updates download a signed build from the GitHub releases below. The app also
offers an update at launch when it is behind, and via File → **Check for Updates**.

---

## Reporting bugs and requesting features

**https://github.com/ojhurst/better-screen-sharing-app/issues**

That repository is documentation, issue tracking, and releases only — it contains
no source. If you hit a defect while driving this app, file it there rather than
working around it silently. Use the issue forms; they ask for the four things
that make a report actionable:

1. Build number — `bss update --check`, or the app's About box.
2. macOS version and chip — `sw_vers` and `uname -m`.
3. Connection type — `direct` or `ssh`, and whether `ensureX11vnc` is set.
4. A log excerpt — `bss logs 60`.

Scrub before you file. Log lines and config snippets can carry hostnames, SSH
aliases, LAN IPs, usernames, and secret paths. Replace them with placeholders.
A public issue tracker is a public place.
