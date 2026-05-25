# nfs-swiftbar

A [SwiftBar](https://github.com/swiftbar/SwiftBar) plugin that shows the status of
NFS shares managed by macOS **autofs**, with a one-click sudo recovery action.

## What it shows

Reads the shares straight from `/etc/auto_direct` (single source of truth — no
duplicated config), then for each share renders:

| Indicator | Meaning |
|-----------|---------|
| 🟢 green dot · active | mounted and active right now |
| ⚪️ grey dot · idle | reachable, just not mounted (healthy — past the 1h autofs idle timeout) |
| 🔴 red dot · unreachable | nfsd (TCP 2049) not answering — server down, NFS stopped, or off-VPN |

Menu-bar icon: green if any share is active, red if any host is unreachable,
grey if everything's merely idle. Header shows `N/total active`.

## Actions (per share)

- **Reveal in Finder** — opens the path, which triggers the automount.
- **Show free space** — `df` the share and report `Size / Used / Free` in a
  notification. Click-only by design, and shown only on **active** shares so
  checking capacity never accidentally automounts an idle one.
- **Unmount (sudo)** — `umount -f` the share *without* re-triggering it, so it
  drops to idle immediately instead of waiting out the 1h timeout. autofs still
  re-mounts on next access. Shown only on active shares (nothing to unmount
  otherwise). Forced (`-f`) so a dead server can't hang the unmount.
- **Force remount (sudo)** — `umount -f` the stale handle then re-access to let
  autofs re-trigger a fresh mount. Prompts once for admin rights via `osascript`.
  This is the genuinely useful action under autofs: it recovers a stale `soft`
  mount after a network blip, which a plain toggle can't.

## Root menu

- **Force remount all (sudo)** — runs the force-remount across every share in
  the map. All the `umount -f`s are chained into a single privileged
  `osascript` call, so you're prompted for admin **once**, not per share; each
  share is then re-triggered and a `Force-remounted N/total` summary is shown.
  Note this brings idle shares up too (they'll idle-unmount again after 1h).
- **Unmount all (sudo)** — force-unmounts every share in one prompt, *without*
  re-triggering. Everything drops to idle until next access; reports an
  `Unmounted N/total` summary.

## Design note — don't defeat the idle timeout

The 30s poll never `ls`/`stat`/`df`s the mount paths; status comes only from the
`mount` table and a TCP probe of the nfsd port (2049). Touching a path would
re-trigger the automounter every cycle and keep shares mounted forever.

`df` *does* access a share, so "Show free space" is a deliberate, click-only
action rather than part of the poll — active shares still idle-unmount after the
1h autofs timeout as designed.

## Install

```bash
ln -sf "$PWD/nfs.30s.sh" "$HOME/Documents/SwiftBar/nfs.30s.sh"
```

(`~/Documents/SwiftBar` is the configured SwiftBar plugin directory:
`defaults read com.ameba.SwiftBar PluginDirectory`.)

## Development

Render all three states (active / idle / unreachable) from fixture data, without
touching real mounts or the network — handy for eyeballing icons and colours:

```bash
./nfs.30s.sh --selftest
```

The live poll and `--selftest` share one rendering function (`emit_share`), so
the fixture output is faithful to the real thing.

## Dependencies

`bash`, `mount`, `nc`, `df`, `osascript` — all stock macOS. No runtime to install.
