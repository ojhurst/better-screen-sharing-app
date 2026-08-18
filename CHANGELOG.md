# Changelog

Build numbers are a single incrementing integer, shown in the window title bar
and reported by `bss update --check`.

## Build 56

- **Notarized by Apple**, with the ticket stapled into the bundle. First launch
  is now clean: no right-click workaround, and no network connection needed for
  Gatekeeper to clear the app.
- Fixed code signing for two executables hidden inside Electron's frameworks,
  `chrome_crashpad_handler` and `ShipIt`. Signing a framework does not sign bare
  binaries nested within it, so both had been shipping unsigned, with no
  hardened runtime and no secure timestamp. Apple rejected the first submission
  on exactly these.

## Build 55

- Better Screen Sharing is a paid product. Downloads are no longer served from
  this repository; it is documentation and issue tracking now.

## Build 54

- This repository moved from `ojhurst/bss` to `ojhurst/better-screen-sharing-app`.
  The old address still redirects. Build 54 points here directly.
- The updater now follows redirects when it asks GitHub for the latest release,
  so an installed copy keeps updating even if this repository is renamed again.

## Build 53

- Fixed an update check that could never fire on a bundle installed before
  Build 52. Those bundles report a marketing version (`1.0.0`) where the build
  number now lives; reading it as digits produced "100", so every real release
  looked older and the updater stayed shut. A version that is not a plain
  integer is now ignored, and an installed build that cannot be identified
  counts as out of date rather than current — so an old install repairs itself
  on the next check instead of silently never updating again.

## Build 52

- First public release. The app is now distributed as a signed download rather
  than something you build from source.
- `bss update` downloads and installs the newest published build directly,
  verifying its signature before it replaces anything. No source checkout, no
  build toolchain, and no git required — which is what made updates impossible
  on any machine that only had the app.
- The updater now installs to a staging path and swaps it into place, so a
  failed update can no longer leave a machine with no app at all.
- `AGENTS.md` ships inside the bundle at `Contents/Resources/AGENTS.md`, so an
  AI agent can learn to drive the app with no network and no source.
- **New setting: `passwordCommand`.** `passwordSecret` is now resolved by a
  helper you name in the config rather than a hardcoded path, so you can point
  it at your own password manager or parameter store. Defaults to
  `bss-resolve-secret` on `PATH`.
- The app icon is applied correctly. Every previous build silently shipped
  Electron's default icon.

## Build 51

- The Connections panel collapser is now the primary, always-visible control for
  reclaiming screen space. The top bar never hides, so the toggle is always
  reachable, and collapsing re-letterboxes the remote to fill the new width.

## Build 50

- An unreadable build number can no longer masquerade as "up to date." The
  updater fails loudly instead of silently skipping an available update.

## Build 49

- Immersive full-screen mode: a thin frame in the letterbox gutter marks the
  screen edge, and a slide-down bar at the top edge carries the live indicator
  and an exit button. Enter and exit via ⌃⌘F, the View menu, or the green
  traffic light — all three land in the same mode.

## Build 47

- Draw a cursor over the remote canvas even when the remote hides its own. A
  headless display with no cursor made the view feel dead.

## Build 46

- Support a target on the same machine under a different user account, for
  viewing a second local desktop session.

## Builds 44–45

- Headless Linux targets can now be given a real desktop: `ensureDesktop` starts
  a window manager, a taskbar, and a wallpaper on the remote display, and every
  window is sized to 95% of the work area so its edges stay grabbable through
  the viewer.

## Build 43

- `bss view <key>` now launches the app if it is not running and raises the
  window to the front. Re-pointing a window that stays buried behind others
  helps nobody. `--no-focus` opts out for scripted re-pointing.

## Build 42

- Keep watching for content on a blank remote until it actually arrives, rather
  than giving up on a fixed timer.

## Builds 39–40

- Clipboard paste into the remote, including typing the local clipboard into a
  focused remote field.

## Builds 37–38

- Resilient reconnect: exponential backoff with a circuit breaker, and a
  wait-for-bind tunnel that survives high-latency links.
- Per-connection refresh button, keyboard-focus hardening, and a one-click
  copy-error button.

## Build 35

- Self-update: **File → Check for Updates**, a silent check at launch, and the
  `bss update` CLI.

## Build 34

- Mac targets connect only on an explicit click, so launching the app no longer
  triggers a Screen Sharing prompt. Fixed fit mode clipping the top and bottom.

## Builds 19–22

- Remote mouse and keyboard control, with an input probe.
- Zoom in/out/fit with keyboard shortcuts and ⌘-scroll.
- Display menu: mirror or extend the remote Mac's monitors over SSH. Mirroring
  collapses multiple displays into one, which is far easier to drive remotely.

## Builds 13–18

- Warm session pool — switching connections is instant, like browser tabs. Only
  the visible connection auto-redials; a dropped background one revives when you
  switch to it.
- Renderer-side VNC event logging: connect, disconnect, auth, security failure.
- Fixed the Display controls in the packaged app by unpacking `bin/` from the
  asar archive.

## Builds 7–12

- Packaged as a standalone `.app`, with the config relocated to a shared,
  user-writable path so the read-only bundle and the CLI agree on one file.
- Connections CRUD panel and File menu, `direct` and `ssh` connection types, and
  live config reload.
- File logging with instrumentation, automatic freeing of an orphaned local
  port, and the build number in the native title bar.

## Builds 1–2

- Config-driven noVNC viewer with a self-managed SSH tunnel.
- The CLI clears `ELECTRON_RUN_AS_NODE`, so launching from an Electron-hosted
  shell opens a real window instead of a headless Node process.
