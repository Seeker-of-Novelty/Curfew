# Curfew

A distraction blocker and bedtime screen lock for Arch-based systems running
KDE Plasma. It is built so that turning a block off in a moment of weakness
takes deliberate effort, while turning one on is instant.

Python and Qt 6 (PySide6), packaged for pacman. No runtime dependency outside
the standard library and PySide6.

## The one rule that shapes everything

**Strengthening a block happens immediately. Weakening one waits.**

| Instant | Queued for the cancel delay |
| --- | --- |
| Add an app, site or blocklist | Remove an app, site or blocklist |
| Extend a session | End a session early |
| Lengthen a bedtime window | Shorten or clear a bedtime window |
| Enable a schedule | Disable a schedule, or drop a day |
| Raise the cancel delay | Lower the cancel delay |

The delay applies whether or not a session is running. That is what stops you
quietly deleting tomorrow's bedtime this afternoon. Queued changes are listed
in the app with a countdown and can be withdrawn, because withdrawing a
weakening is itself a strengthening.

## What it blocks

### Applications

A root service listens to the kernel's process-connector netlink socket, so
every `exec()` is seen and a blocked program is closed in about a tenth of a
second. A sweep of `/proc` every five seconds catches anything missed. Apps
are recognised four ways:

- the systemd unit its cgroup belongs to, which is how the app menu, KRunner
  and Flatpak all start things;
- the Flatpak application id, read from the sandbox;
- the executable path or name;
- the script or bundle path on the command line, which matters because many
  Linux launchers are shell scripts rather than binaries.

Matching through the cgroup kills the whole cgroup, so helper processes and
Proton or Wine children go too. The compositor, shell, lock screen, session
services, anything running as root, and Curfew itself are never touched.

### Websites

Two independent layers:

- **`/etc/hosts`.** systemd-resolved parses this into memory and answers from
  it authoritatively, so a listed name never reaches an upstream server. It
  covers every program, not just browsers.
- **Browser managed policy.** A `URLBlocklist` that cannot be changed from
  the browser's settings and applies to private windows.

The policy also switches **DNS-over-HTTPS off and locks it**
(`DnsOverHttpsMode: off`, `BuiltInDnsClientEnabled: false`, and Firefox's
`DNSOverHTTPS` locked). Without that the hosts layer is decorative: DoH sends
lookups straight to a third party and never consults the system resolver.

### Subscribed blocklists

Point Curfew at any published domain list and it will download, parse and
enforce it. Hosts files, plain domain lists, AdBlock (`||domain^`) and
dnsmasq formats are all detected per line, so mixed files work.

```sh
curfewctl lists --add \
  https://raw.githubusercontent.com/hagezi/dns-blocklists/main/wildcard/nsfw-onlydomains.txt \
  --name "Adult content"
```

Notes on how this works:

- **The daemon never downloads anything.** Its unit sets `IPAddressDeny=any`
  and restricts address families to Unix and netlink. Downloads run in your
  own session, from the app or `curfewctl`, and only the parsed domains are
  pushed over the local socket. A root service that makes outbound HTTPS
  requests is a much bigger thing to trust.
- A user timer refreshes twice a day. Requests are conditional, so an
  unchanged list costs an empty `304` rather than a re-download.
- AdBlock allowlist rules (`@@||domain^`) are ignored. A remote list should
  not be able to punch holes in blocks you set by hand.
- `localhost` and the machine's own hostname can never be blocked, whatever
  a list says.
- Large lists go to the hosts layer only, because a browser policy accepts at
  most 1000 patterns. Locking DoH off is what makes that sufficient.
- Lists can be big. For scale, one adult-content list is ~74,000 domains and
  one gambling list ~529,000; together they make a 21 MB hosts file. Curfew
  warns above 250,000 domains, and if a list is ever cut short it says so
  rather than looking like it worked.

### The launcher

Blocked apps disappear from the application menu and from search, through an
override desktop entry in your own applications directory. Your own entries
are backed up first and restored when the block ends.

### Bedtime

During a bedtime window the screen is locked through logind and re-locked a
few seconds after any unlock. Text consoles are closed, so dropping to a TTY
does not help. Notifications warn you a few minutes beforehand. There is also
a one-off countdown for a single night.

## Suspend, and why a closed lid does not buy you time

Deadlines are absolute wall-clock instants, so suspended time is spent time:
a session with 58 minutes left when the lid closes has 28 minutes left after
half an hour asleep.

This needs care, because `CLOCK_MONOTONIC` — which every asyncio timeout is
measured against — stops advancing while a machine is suspended, and on an
s2idle laptop that can be most of the day. Curfew therefore:

- measures elapsed time against the wall clock, and distinguishes a suspend
  from someone winding the clock forward by comparing against
  `CLOCK_BOOTTIME`, which keeps counting through a suspend;
- subscribes to logind's `PrepareForSleep` so it knows the moment it wakes,
  instead of sitting in a frozen timeout, and immediately re-evaluates every
  block, re-sweeps processes and rebuilds the netlink socket;
- logs how long it was asleep, and reports the running total in
  `curfewctl status`.

If a block ever looks like it over-ran, check the log before blaming the
clock. Every change to a session's end time is logged, extensions included.

## Persistence

The service starts at boot before the display manager, restarts automatically
if killed with no rate limit, and keeps its settings file immutable. None of
this is unbreakable and it is not meant to be: you have root on your own
machine, and `sudo systemctl stop curfewd` always works. The point is that
getting out takes a deliberate sequence of steps rather than one click.

## Install

```sh
git clone https://github.com/YOURNAME/curfew
cd curfew
makepkg -si
sudo systemctl enable --now curfewd
systemctl --user enable --now curfew-lists.timer   # keep blocklists fresh
```

Then open **Curfew** from the app menu. Nothing is blocked until you configure
it. A tray icon starts with your session and shows bedtime warnings.

## Command line

```sh
curfewctl status                       # what is blocked right now
curfewctl apps                         # list installed applications
curfewctl block brave steam --site youtube.com
curfewctl start 2h                     # or: --until 17:00
curfewctl sleep --at 23:00 --until 07:00 --days all
curfewctl sleep --in 45m --for 8h      # one-off sleep timer
curfewctl schedule --add 09:00-12:00 --days weekdays
curfewctl lists --add URL --name NSFW  # subscribe to a blocklist
curfewctl lists --refresh              # re-download all of them
curfewctl pending                      # queued weakenings and their timers
curfewctl cancel                       # request an early end
curfewctl watch                        # stream events
```

## Layout

```
curfew/common/     desktop-entry scanning, time windows, wire protocol,
                   blocklist downloading (user side only)
curfew/daemon/     the root service: process matching and killing, hosts and
                   browser policy, logind, blocklist storage
curfew/gui/        PySide6 app; QML under curfew/gui/qml
data/              systemd units, desktop entries, icon
tests/test_core.py logic tests, no root required
```

The daemon and the app talk over a Unix socket with one JSON object per line.
The daemon holds all state and all authority; the app is only a view. Every
operation goes through one function that returns what to apply now and what to
defer, which is why the rule at the top holds everywhere instead of being
re-implemented per feature.

## Development

```sh
python tests/test_core.py

# a non-root instance with its own state, hosts file and policy tree
CURFEW_SOCKET=/tmp/c.sock CURFEW_HOSTS=/tmp/hosts CURFEW_POLICY_ROOT=/tmp/pol \
  python -m curfew.daemon.main --dev --dry-run --state-dir /tmp/state

CURFEW_SOCKET=/tmp/c.sock python -m curfew.gui.main
```

`--dev` drops the root requirement and the immutable flags. `--dry-run` logs
what would happen instead of doing it. They are independent: `--dev` alone
really does kill processes.

## Removing it

```sh
sudo pacman -R curfew
```

The pre-remove hook runs `curfewd --cleanup`, which strips the hosts block,
deletes the policy files, clears every immutable flag and restores hidden
launcher entries. Settings stay in `/var/lib/curfew` unless you delete them.

## Licence

MIT.
