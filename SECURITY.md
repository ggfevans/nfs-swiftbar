# Security Policy

## Reporting a vulnerability

Please report security issues privately. Do not open a public issue or pull
request for them.

Use GitHub's private vulnerability reporting:

1. Open the [Security tab](https://github.com/ggfevans/nfs-swiftbar/security).
2. Click **Report a vulnerability**, or use the direct link:
   <https://github.com/ggfevans/nfs-swiftbar/security/advisories/new>.
3. Include the affected version, steps to reproduce, and the impact.

This is a small project maintained by one person, so please allow a few days for
an initial reply. I will confirm receipt, work with you on a fix, and credit you
in the advisory unless you would rather stay anonymous.

## Supported versions

Only the latest release on `main` receives fixes. There are no long-term support
branches.

## Scope and threat model

nfs-swiftbar is a local macOS SwiftBar plugin. It runs no network service and
processes no untrusted remote input. The parts that matter for security are:

- **Privileged actions.** Unmount and remount run `umount -f` through
  `osascript ... with administrator privileges`, which prompts for the admin
  password. Mount paths come from `/etc/auto_direct`, a root-owned file, and are
  quoted with `printf %q` and AppleScript escaping before they reach the shell.
- **No stored secrets.** The plugin saves and transmits nothing.

A report that requires the attacker to already have root or write access to
`/etc/auto_direct` is out of scope, because that access is itself the compromise.

Reports about the argument escaping, the privileged-command construction, or any
way to make the plugin run unintended commands are welcome.
