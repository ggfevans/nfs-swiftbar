# Changelog

All notable changes to this project are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
adheres loosely to [Semantic Versioning](https://semver.org/).

## [1.4] - 2026-05-25

### Added
- **Unmount (sudo)** per-share action and **Unmount all (sudo)** root action.
  Force-unmount (`umount -f`) without re-triggering, so shares drop to idle
  immediately instead of waiting out the autofs idle timeout.

### Changed
- Refactored the shared map-parse, chained-unmount and re-trigger logic into
  `read_mountpoints`, `sudo_umount_all` and `retrigger` helpers, used by all
  four mount/unmount actions.

## [1.3] - 2026-05-24

### Added
- **Force remount all (sudo)** root action. Chains every share's `umount -f`
  into a single privileged `osascript` call (one admin prompt), then re-triggers
  each and reports a summary.

## [1.2] - 2026-05-24

Initial release.

### Added
- Per-share NFS autofs status (**active / idle / unreachable**), read from
  `/etc/auto_direct` as the single source of truth.
- Reachability determined by a TCP probe of the nfsd port (2049), so "reachable"
  means the NFS service is actually answering rather than just that the host
  responds to ping.
- Per-share actions: **Reveal in Finder**, **Show free space** (click-only and
  active-only, so it never automounts an idle share), **Force remount (sudo)**.
- `--selftest` flag to render all three states from fixture data without
  touching real mounts or the network.

### Security
- All `osascript` arguments are AppleScript-escaped and `printf %q`-quoted for
  the inner `/bin/sh`; the re-trigger access is bounded by a `perl` alarm so an
  unreachable server can't hang the action.
