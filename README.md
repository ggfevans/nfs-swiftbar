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
- **Force remount (sudo)** — `umount -f` the stale handle then re-access to let
  autofs re-trigger a fresh mount. Prompts once for admin rights via `osascript`.
  This is the genuinely useful action under autofs: it recovers a stale `soft`
  mount after a network blip, which a plain toggle can't.

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
