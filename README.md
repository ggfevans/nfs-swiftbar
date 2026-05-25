# nfs-swiftbar

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Platform: macOS](https://img.shields.io/badge/platform-macOS-000000?logo=apple&logoColor=white)

A [SwiftBar](https://github.com/swiftbar/SwiftBar) plugin that puts the status of
your NFS shares in the macOS menu bar — and gives you one-click `sudo` recovery
when a mount goes stale. Built for shares mounted on demand by **autofs**.

![nfs-swiftbar showing three active shares and the per-share action submenu](assets/nfs-swiftbar-screenshot.png)

## Why

Under autofs, shares mount on access and unmount themselves after an idle period,
so there's no Finder window telling you what's currently attached or whether a
server is reachable. This plugin surfaces that at a glance, without interfering
with the automounter — the background poll never touches the mount paths, so it
won't keep idle shares alive (see [Design notes](#design-notes)).

## Features

- **At-a-glance status** per share: active, idle, or unreachable.
- **Honest reachability** — probes the nfsd port (TCP 2049), so a share is only
  "reachable" when NFS is actually answering, not merely when the host pings.
- **Recovery that fits autofs** — force-remount a stale handle, or unmount on
  demand, individually or across all shares in a single admin prompt.
- **Stays out of the way** — the 30s poll reads the mount table and a port
  probe only; it never accesses the mount paths, preserving the idle timeout.
- **Zero runtime** — pure `bash` plus stock macOS tools.

## Status indicators

The plugin reads your shares from `/etc/auto_direct` (single source of truth) and
renders each one:

| Indicator | Meaning |
|-----------|---------|
| 🟢 green · active | mounted and active right now |
| ⚪️ grey · idle | reachable, just not mounted (healthy — past the autofs idle timeout) |
| 🔴 red · unreachable | nfsd (TCP 2049) not answering — server down, NFS stopped, or off-VPN |

The menu-bar icon is green if any share is active, red if any host is
unreachable, otherwise grey; the header shows `N/total active`.

## Menu actions

**Per share** (sudo actions prompt once via a standard macOS admin dialog):

| Action | What it does |
|--------|--------------|
| Reveal in Finder | Opens the path, triggering the automount. |
| Show free space | `df` the share and report `Size / Used / Free`. *Active shares only* — so checking capacity never automounts an idle one. |
| Unmount (sudo) | `umount -f` **without** re-triggering, so the share drops to idle now instead of waiting out the timeout. *Active shares only.* |
| Force remount (sudo) | `umount -f` then re-access so autofs mounts it fresh. The useful recovery move when a `soft` mount goes stale after a network blip. |

**Root menu:**

| Action | What it does |
|--------|--------------|
| Force remount all (sudo) | Force-remount every share. One admin prompt; reports `Force-remounted N/total`. (Brings idle shares up too; they idle out again later.) |
| Unmount all (sudo) | Force-unmount every share, no re-trigger. One prompt; reports `Unmounted N/total`. |

## Requirements

- **macOS** with [SwiftBar](https://github.com/swiftbar/SwiftBar) installed.
- NFS shares mounted via **autofs direct maps** declared in `/etc/auto_direct`
  (see [Configuration](#configuration)).
- Stock tools only: `bash`, `mount`, `nc`, `df`, `osascript`, `perl` — all
  present on a default macOS install.

## Configuration

The plugin parses `/etc/auto_direct`, the autofs **direct map** that lists your
mounts. Each non-comment line is `mountpoint  options  host:export`, e.g.:

```
/Volumes/media      -fstype=nfs,resvport,rw,soft,intr,tcp   nas.example.lan:/export/media
/Volumes/backups    -fstype=nfs,resvport,rw,soft,intr,tcp   nas.example.lan:/export/backups
```

The map is wired into the automounter via `/etc/auto_master`:

```
/-    auto_direct
```

…then `sudo automount -vc` to load it. The plugin only needs the **mountpoint**
and the **host** (the part before `:` in the third column) from each line.

> Using a different map file? Edit the `MAP="/etc/auto_direct"` line near the top
> of `nfs.30s.sh`.

## Install

```bash
git clone https://github.com/ggfevans/nfs-swiftbar.git
ln -sf "$PWD/nfs-swiftbar/nfs.30s.sh" "$HOME/Documents/SwiftBar/nfs.30s.sh"
```

`~/Documents/SwiftBar` should be your SwiftBar plugin directory — confirm with:

```bash
defaults read com.ameba.SwiftBar PluginDirectory
```

Then **SwiftBar → Refresh All**. The `30s` in the filename sets the refresh
interval; rename (e.g. `nfs.1m.sh`) to change it.

## Design notes

**Don't defeat the idle timeout.** The 30s poll never `ls`/`stat`/`df`s the
mount paths — status comes only from the `mount` table and the TCP port probe.
Accessing a path would re-trigger the automounter every cycle and keep shares
mounted forever. For the same reason, **Show free space** (which must `df`, and
so touches the share) is a deliberate, click-only action on active shares rather
than part of the poll.

**Forced unmounts.** Unmount and force-remount use `umount -f` so an unreachable
server can't hang the operation; the re-trigger access is additionally bounded
by a `perl` alarm. Best-effort: a hard mount wedged in uninterruptible I/O may
still block.

## Development

Render all three states from fixture data — no real mounts or network touched —
to eyeball icons and colours:

```bash
./nfs.30s.sh --selftest
```

The live poll and `--selftest` share one rendering function (`emit_share`), so
the fixture output stays faithful to the real thing.

## License

[MIT](LICENSE) © Gareth Evans
