# herdr-notifications

Native OS desktop notifications for [herdr](https://herdr.dev) agent status
changes — get pinged by your system's real notification center when an
agent needs you or finishes a task, instead of having to keep an eye on the
terminal.

- **Cross-platform**: Linux, macOS, and Windows via [`notify-rust`](https://github.com/hoodie/notify-rust)
  (also builds and runs on the BSDs, though herdr's own plugin manifest
  schema doesn't have a platform value for them yet).
- **Only notifies when it matters**: fires on `blocked` (agent needs input)
  and `done` (agent finished), and dedupes so an unchanged status never
  re-notifies — but a `blocked → working → blocked` cycle correctly notifies
  again, since it's not a repeat.
- **Click to focus**: clicking a status-change notification focuses the
  originating pane back in herdr. On Linux, set
  `HERDR_NOTIFICATIONS_RAISE_HOST=1` to also raise the terminal window that
  hosts the Herdr UI (KDE, GNOME, XFCE/X11, Hyprland, Sway — best-effort).
- **Location in the toast**: body shows `workspace · tab` (and the agent
  title when present) so you can tell which project fired, not only which
  agent binary.
- **Zero runtime config required**: works out of the box; the dedup state
  lives under herdr's own per-plugin state directory (or a per-user local
  data directory as a fallback), never a shared/world-writable location.

## Install

```sh
herdr plugin install quinnjr/herdr-notifications
```

herdr builds the plugin (`cargo build --release`) on install, then wires up
the `pane.agent_status_changed` event automatically.

## Usage

Nothing to configure — once installed and enabled, notifications just
happen. To confirm your OS notification permissions/backend are working
without waiting for a real agent-status change, run the bundled smoke-test
action from herdr's command palette or:

```sh
herdr plugin action invoke test-notification --plugin quinnjr.herdr-notifications
```

## How it works

herdr fires a `pane.agent_status_changed` event (idle / working / blocked /
done / unknown) for every agent pane whenever its status changes. This
plugin's binary is invoked once per event:

1. Every status transition is recorded to a small on-disk dedup table (one
   entry per pane), written atomically (temp file + rename) and guarded by
   a short-lived exclusive lock, so concurrent status changes across
   multiple panes can't corrupt or race on it.
2. Only `blocked` and `done` are surfaced as notifications — `idle` /
   `working` / `unknown` are recorded (so the next `blocked`/`done` is
   correctly recognized as new) but never notify on their own.
3. The notification is shown on a background thread with a bounded wait, so
   a stuck notification daemon can never hang the process indefinitely.
   Summary is `{agent} is done` / `{agent} needs you`; body is
   `location · tab`, taken from `HERDR_PLUGIN_CONTEXT_JSON` when it describes
   the pane that changed. Otherwise it falls back to `herdr pane list`, where
   the first label is the pane's cwd basename (or its workspace id) and the
   second is its tab id — close enough to place the pane, but not the
   workspace and tab *labels* the context path gives you.
4. On Linux/BSD the toast carries an **Open in Herdr** button. Clicking it
   runs `herdr agent focus <pane_id>` to bring that pane back into view.
   On Linux, opt in with `HERDR_NOTIFICATIONS_RAISE_HOST=1` to also raise the
   host terminal window via desktop-specific APIs (KWin on KDE, GNOME Shell
   Eval on GNOME, `hyprctl`/`swaymsg` on those compositors, `wmctrl`/`xdotool`
   on X11). The toast stays up for 60 seconds, and the plugin process exits
   with it.
5. Closing a notification is not a click. The one exception is
   xfce4-notifyd, where a body click emits *only*
   `NotificationClosed(Dismissed)` and never `ActionInvoked`; the plugin
   detects that daemon via `GetServerInformation` and reads a dismissal as a
   click there alone. Set `HERDR_NOTIFICATIONS_CLICK_ON_DISMISS=1` to force
   that reading on another daemon that behaves the same way, or `=0` to turn
   it off.

Herdr 0.8+ delivers `HERDR_PLUGIN_EVENT_JSON` as
`{"event":"…","data":{…}}`; this plugin unwraps `data` and still accepts
older bare payloads (Herdr 0.7.x), so `min_herdr_version` stays `0.7.0`.
`agent`, `display_agent` and `title` are typed `["string", "null"]` by herdr
and are accepted missing, `null`, or empty; `workspace_id` is accepted
missing. An unrecognized `agent_status` is ignored rather than treated as a
parse failure, so a future herdr status will not break the plugin.

## Requirements

- herdr ≥ 0.7.0
- A working OS notification backend: a D-Bus session + notification daemon
  on Linux/BSD (present on virtually every desktop environment), or the
  native notification center on macOS/Windows.
- Optional (Linux host-window raise): one or more of `wmctrl`, `xdotool`,
  `qdbus`/`qdbus6` (KDE), `gdbus` (GNOME), `hyprctl` (Hyprland), or
  `swaymsg` (Sway), depending on your desktop.

## Development

```sh
cargo build --release
cargo test
herdr plugin link .   # develop against a local checkout instead of installing
```

## License

MIT — see [LICENSE](LICENSE).
