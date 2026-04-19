# ptp4RaspberryPi

PTP (Precision Time Protocol) on Raspberry Pi, served from a GNSS/PPS‑disciplined
grandmaster to Pi clients, and fed back into `ntpsec` on each client as a
low‑jitter refclock.

> **Warning.** This is optional and experimental. Proceed at your own risk.

Useful references that informed this document:

- <https://github.com/tiagofreire-pt/rpi_uputronics_stratum1_chrony/blob/main/steps/advanced_system_tuning.md>
- <https://gpsd.gitlab.io/gpsd/gpsd-time-service-howto.html#_providing_local_ntp_service_using_ptp>
- <https://docs.fedoraproject.org/en-US/fedora/latest/system-administrators-guide/servers/Configuring_PTP_Using_ptp4l/#sec-Serving_NTP_Time_with_PTP>
- <https://sourceforge.net/p/linuxptp/mailman/linuxptp-devel/thread/1424738292.9759.53.camel%40intel.com/>
- Debian packaging README: <https://salsa.debian.org/multimedia-team/linuxptp/-/blob/master/debian/README.Debian>

Thanks to `@tiagofreire-pt` and the `linuxptp-devel` contributors.


## Scope and assumptions

- **Hardware**: Raspberry Pi CM4, CM5, or Pi 5 on both sides. These have a PTP
  Hardware Clock (PHC) in the Ethernet MAC.
- **OS**: Debian 12 (bookworm) or 13 (trixie); Raspberry Pi OS based on either.
  `linuxptp` 4.0+ is assumed — older versions use different option names
  (`slaveOnly` instead of `clientOnly`, numeric `fault_reset_interval`, etc.).
- **Topology**: server and clients on the same L2 segment, ideally direct or
  through a PTP‑aware switch. Unmanaged switches in the path will cause
  path‑asymmetry offsets; see the troubleshooting section.
- **Time source on server**: GNSS + PPS, disciplining `ntpsec`.
- **Server role**: PTP grandmaster, clockClass 6.
- **Client role**: PTP client, consuming time via `phc2sys -E ntpshm` and
  feeding it to `ntpsec` as a refclock.


## Architecture

```
Server:
  GNSS ──► gpsd ──► ntpsec ──► system clock (UTC)
                                      │
                                      ▼
                                  phc2sys -w  (applies UTC→TAI offset, +37s)
                                      │
                                      ▼
                                   PHC (TAI) ──► ptp4l ──► network
                                                              │
                                                              ▼
Client:                                                    network
                                                              │
                                                              ▼
                                   PHC (TAI)  ◄──  ptp4l  ◄───┘
                                      │
                                      ▼
                                  phc2sys -E ntpshm  (strips TAI→UTC, -37s)
                                      │
                                      ▼
                                   SHM(2) (UTC) ──► ntpsec
```

Why `phc2sys` in both directions? `ntpsec` (and `ntpd`) do not discipline the
NIC's PHC. `phc2sys` is the bridge that moves time between the system clock
and the PHC, and it's also the only component that correctly applies the
UTC↔TAI offset when it crosses that boundary. Doing this inside `ptp4l.conf`
(`ntpshm` as a servo there) uses subtly different semantics and silently
produces 37‑second errors on clients.


## Prerequisites

### Install linuxptp on both sides

```bash
sudo apt update && sudo apt install linuxptp
```

Confirm version 4.x:

```bash
ptp4l --version
```

If you're on 3.x, several options in this document (`clientOnly`,
`fault_reset_interval ASAP`) need to be written in their 3.x form; consider
upgrading to trixie or backports instead.

### Verify hardware timestamping support

```bash
ethtool -T eth0
```

You need to see entries like:

```
PTP Hardware Clock: <N>
Hardware Transmit Timestamp Modes: ... on
Hardware Receive Filter Modes:    ... ptp-v2-l2-event  (or similar)
```

If `PTP Hardware Clock: none`, this NIC cannot do hardware timestamping and the
rest of the guide does not apply as written; you'd need `time_stamping software`
and accept ~tens of µs accuracy. On CM4, CM5, and Pi 5 the on‑board Ethernet
has a PHC; on older Pis it does not.

### Identify the PHC device

```bash
ethtool -T eth0 | grep 'PTP Hardware Clock'
```

If it says `PTP Hardware Clock: 0`, that's `/dev/ptp0`; `1` → `/dev/ptp1`, etc.
On a Pi with only one PHC this is always `/dev/ptp0`, but the explicit check is
good hygiene — commands later in the document use this path.

### Configure the leap‑seconds file for ntpsec (server)

The server's `phc2sys` needs to know the current UTC↔TAI offset (37 s) in order
to feed the PHC in TAI. It picks this up from the kernel's TAI offset, which
`ntpsec` sets — but only if `ntpsec` has been given a leap‑seconds file.

The file ships with the base `tzdata` package at
`/usr/share/zoneinfo/leap-seconds.list` — no extra package install needed. Just
point `ntpsec` at it. In `/etc/ntpsec/ntp.conf`, add:

```
leapfile /usr/share/zoneinfo/leap-seconds.list
```

Then:

```bash
sudo systemctl restart ntpsec
```

Without this, `phc2sys -w` on the server will happily copy UTC time into the
PHC without offset, and every client will be 37 seconds off.

Note: `leap-seconds.list` has an expiration date (re‑issued semi‑annually by
IERS). Debian ships updated `tzdata` packages as new IERS bulletins are
published, so routine `apt upgrade` keeps it fresh. `ntpsec` logs a warning as
the expiration approaches.


## Preparation common to both sides

### Reduce Ethernet coalescence

Ethernet drivers coalesce packet interrupts for throughput, at the cost of
latency. For PTP we want the opposite tradeoff. Current values:

```bash
sudo ethtool -c eth0
```

Set to the minimum the Pi's driver accepts:

```bash
sudo ethtool -C eth0 rx-usecs 4 tx-usecs 4
```

Make it persistent with a small service:

```bash
sudo systemctl edit --force --full eth0-coalescence.service
```

Paste:

```ini
[Unit]
Description=Reduce eth0 coalescence for PTP
Wants=network.target
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -C eth0 rx-usecs 4 tx-usecs 4
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now eth0-coalescence.service
```

This shaves roughly 40 µs off chrony/ntp response jitter. Apply it on both
server and clients.


## Server side (GNSS/PPS grandmaster)

### ptp4l configuration

The server's PHC is driven by `phc2sys` from the system clock (which `ntpsec`
disciplines from GNSS). `ptp4l` only publishes that PHC to the network — it
does not discipline it.

Create `/etc/linuxptp/ptp4l.conf`:

```ini
[global]

# Grandmaster attributes — clockClass 6 = "synchronized to primary reference".
clockClass              6

# Clock accuracy. Conservative for a GPSDO-fed Pi:
#   0x20 25ns    0x23 1us     0x26 25us    0x29 1ms
#   0x21 100ns   0x24 2.5us   0x27 100us   0x2a 2.5ms
#   0x22 250ns   0x25 10us    0x28 250us   0xfe unknown
clockAccuracy           0x23

# Make this node unambiguously preferred over any accidental second grandmaster.
priority1               64
priority2               128

# ptp4l uses the PHC as its clock. It does NOT discipline the PHC here;
# phc2sys does that, driven by ntpsec. "free_running" prevents ptp4l from
# trying to steer the PHC if it ever receives PTP messages itself.
free_running            1

# Clear port faults immediately (helps during driver or link hiccups).
fault_reset_interval    ASAP

# One summary log line per 2^10 = 1024 s.
summary_interval        10

# Linear-regression servo. Irrelevant on a free-running master, but set anyway.
clock_servo             linreg

# Time source: 0x10 atomic, 0x20 GPS, 0x30 terrestrial radio, 0x40 PTP,
#              0x50 NTP, 0x60 hand-set, 0x90 other, 0xa0 internal osc.
timeSource              0x20

# Layer-2 transport with auto delay mechanism (E2E by default; P2P if peer advertises).
network_transport       L2
delay_mechanism         Auto

# Hardware timestamping (default, but explicit is better).
time_stamping           hardware
```

Note: no `ntpshm_segment` here — the server does not feed SHM, only clients do.

### ptp4l systemd unit

Debian's `linuxptp` 4.x ships the template unit
`/usr/lib/systemd/system/ptp4l@.service`. The vendor unit's one rough edge is
that it doesn't wait for the network to be fully up, which on Pis can race
against PHY initialization and produce
`ioctl SIOCSHWTSTAMP failed: Invalid argument` at boot
(Debian bug [#1070847](https://bugs.debian.org/1070847)).

Fix via a drop‑in, **not** by editing the vendor file:

```bash
sudo systemctl edit ptp4l@eth0.service
```

Add only:

```ini
[Unit]
After=network-online.target
Wants=network-online.target

[Service]
Restart=on-failure
RestartSec=2s
```

Enable and start:

```bash
sudo systemctl enable --now ptp4l@eth0.service
systemctl status ptp4l@eth0.service
```

You should see `port 1 (eth0): INITIALIZING to LISTENING` and, shortly after,
`LISTENING to MASTER on ANNOUNCE_RECEIPT_TIMEOUT_EXPIRES`.

### phc2sys systemd unit (server direction)

The vendor `phc2sys@.service` has two problems we need to work around:

1. Its `ExecStart=` runs `phc2sys -w -s %I`, which is the client direction
   (PHC as source). We need the server direction (system clock as source).
2. It declares `Requires=ptp4l.service` and `After=ptp4l.service` — a
   non-templated unit name that doesn't exist on modern Debian, which makes
   `systemctl start` fail with `Unit ptp4l.service not found`.

Problem 1 alone could be fixed with a drop-in that overrides `ExecStart=`.
Problem 2 can't: systemd's drop-in mechanism does not allow resetting
dependency directives (`Requires=`, `After=`, `Wants=`, `Before=`) to an empty
list — only adding to them. So we have to override the whole unit file.

```bash
sudo systemctl edit --full phc2sys@.service
```

Replace the entire content with:

```ini
[Unit]
Description=Synchronize system clock or PTP hardware clock (PHC) for %I
Documentation=man:phc2sys
Requires=ptp4l@%i.service
After=ntpsec.service ptp4l@%i.service
Before=time-sync.target

[Service]
Type=simple
ExecStart=/usr/sbin/phc2sys -s CLOCK_REALTIME -c %I -w -q

[Install]
WantedBy=multi-user.target
```

Key changes from the vendor: the bare `ptp4l.service` references are replaced
with the templated `ptp4l@%i.service`; `ntpsec.service` is added to `After=` so
that the system clock is disciplined before we start copying it to the PHC;
and `ExecStart=` switches to server direction (`-s CLOCK_REALTIME -c %I`).

The cost of `--full` is that the override is a full local copy of the unit
rather than a patch — future upstream improvements won't flow in
automatically. If Debian eventually fixes the vendor unit
(worth a bug report), you can `systemctl revert phc2sys@.service` to adopt
the fix.

Enable and start:

```bash
sudo systemctl enable --now phc2sys@eth0.service
systemctl status phc2sys@eth0.service
```

### Server verification

```bash
# Exactly one ptp4l and one phc2sys, and phc2sys is in system→PHC direction:
ps auxww | grep -E '[p]tp4l|[p]hc2sys'

# PHC should be ~37 s ahead of UTC (i.e. TAI):
sudo phc_ctl /dev/ptp0 cmp

# ptp4l is advertising the current UTC offset and the right clockClass:
sudo pmc -u -b 0 "GET TIME_PROPERTIES_DATA_SET"      # currentUtcOffset 37, currentUtcOffsetValid 1
sudo pmc -u -b 0 "GET PARENT_DATA_SET"               # grandmasterClockClass 6

# ntpsec is in charge of system time:
ntpq -c "rv 0 refid,stratum"
```

`phc_ctl ... cmp` prints the offset between the PHC and `CLOCK_REALTIME`
atomically; don't compare `phc_ctl get` against `date` by eye, the shell latency
swamps the real error.


## Client side

### ptp4l configuration

Create `/etc/linuxptp/ptp4l.conf`:

```ini
[global]

# This node only ever acts as a PTP client.
clientOnly              1

# Client reports clockClass 255 (not a potential master).
clockClass              255

# Same defensive settings as server.
fault_reset_interval    ASAP
summary_interval        10
clock_servo             linreg

# SHM segment for phc2sys -E ntpshm to write to.
# Must match the "unit N" in ntp.conf's refclock shm.
ntpshm_segment          2

network_transport       L2
delay_mechanism         Auto
time_stamping           hardware

# Time source: 0x40 PTP (we receive time from a PTP grandmaster).
timeSource              0x40

# --- Raspberry Pi CM4/CM5/Pi5 quirks ---------------------------------------
# The macb/bcmgenet driver is slow to return TX timestamps. The default
# tx_timestamp_timeout of 1 ms causes "timed out while polling for tx timestamp"
# followed by FAULT_DETECTED, which in this configuration loops forever.
# 100 ms is plenty of margin on this hardware.
tx_timestamp_timeout    100

# The Pi Ethernet IP doesn't handle PTP 2.1 correctly. Pin the on-wire minor
# version to 0 (PTP 2.0) for compatibility. Required on linuxptp 4.x with Pi
# hardware; harmless to leave on linuxptp 3.x.
ptp_minor_version       0
```

### ptp4l systemd unit

```bash
sudo systemctl edit ptp4l@eth0.service
```

```ini
[Unit]
After=network-online.target
Wants=network-online.target

[Service]
Restart=on-failure
RestartSec=2s
```

```bash
sudo systemctl enable --now ptp4l@eth0.service
```

### phc2sys config for SHM feed

`phc2sys` needs to know which SHM segment to write. There's no command‑line
flag for this; it comes from a config file. Create
`/etc/linuxptp/phc2sys.conf`:

```ini
[global]
ntpshm_segment          2
```

(Why a separate file? Passing `-f /etc/linuxptp/ptp4l.conf` also works —
`phc2sys` ignores `ptp4l`‑specific options — but a dedicated file is clearer
and avoids surprises if you tighten `ptp4l.conf` later.)

### phc2sys systemd unit (client direction)

Same vendor-unit problem as on the server: the shipped `phc2sys@.service` has
non-removable `Requires=ptp4l.service` / `After=ptp4l.service` that won't
resolve, and an `ExecStart=` that doesn't match what we want. We override the
whole unit.

```bash
sudo systemctl edit --full phc2sys@.service
```

```ini
[Unit]
Description=Synchronize system clock or PTP hardware clock (PHC) for %I
Documentation=man:phc2sys
Requires=ptp4l@%i.service
After=ntpsec.service ptp4l@%i.service

[Service]
Type=simple
ExecStart=/usr/sbin/phc2sys -s %I -E ntpshm -f /etc/linuxptp/phc2sys.conf -q

[Install]
WantedBy=multi-user.target
```

- `-s %I` — source is the interface's PHC.
- `-E ntpshm` — servo writes to SHM instead of steering a local clock. Target
  selected by the config file's `ntpshm_segment`.
- No `Before=time-sync.target` here — on the client, `ntpsec` is the authority;
  we don't want to signal "time is synced" before `ntpsec` has actually locked
  onto the SHM refclock.

```bash
sudo systemctl enable --now phc2sys@eth0.service
```

### ntpsec refclock

In `/etc/ntpsec/ntp.conf`, add:

```
refclock shm unit 2 refid PTP
```

`unit 2` must match the `ntpshm_segment` set above. Restart:

```bash
sudo systemctl restart ntpsec
```

Verify SHM is actually being written and consumed:

```bash
sudo ntpshmmon -c 10       # should show 10 samples on unit 2 with small offsets
ntpq -pn                   # refid=PTP should eventually get '*' (system peer)
```

`ntpq -pn` can take a few minutes to promote the PTP refclock to system peer
as `ntpsec` collects enough samples to trust it.

### Calibrating path asymmetry

A stable residual offset (typically tens to hundreds of µs) between client and
server after full convergence usually indicates an asymmetric network path —
different physical delays in each direction. Unmanaged switches are the
common cause. PTP assumes the two directions are symmetric and splits the
round‑trip in half, so an asymmetric path appears as a fixed bias.

Measure the mean offset with `ntpq -pn` or `chronyc sources` over an hour of
quiet operation, then calibrate with `time2` in the refclock line:

```
refclock shm unit 2 refid PTP time2 0.000180    # +180 µs example
```

Use half the observed one‑way bias as the `time2` value. Better: replace the
unmanaged switch with a PTP transparent clock or run a direct cable.


## Client verification

```bash
# Exactly one ptp4l and one phc2sys:
ps auxww | grep -E '[p]tp4l|[p]hc2sys'

# ptp4l should be SLAVE with small master offsets:
journalctl -u ptp4l@eth0.service -n 30 --no-pager

# Client PHC should also be ~37 s ahead of UTC (it's slaved to the GM's PHC):
sudo phc_ctl /dev/ptp0 cmp

# SHM is ticking:
sudo ntpshmmon -c 10

# ntpsec has adopted PTP as its system peer:
ntpq -pn
```

Healthy `ptp4l` log lines look like:

```
port 1 (eth0): UNCALIBRATED to SLAVE on MASTER_CLOCK_SELECTED
master offset          -38 s2 freq   -2374 path delay       8123
master offset          -12 s2 freq   -2369 path delay       8120
```

`s2` = servo state locked. Offsets of a few tens of ns to low µs on a direct
cable, tens of µs through a switch, are normal.


## Troubleshooting

### `ioctl SIOCSHWTSTAMP failed: Invalid argument` at boot

Four common causes, roughly in order:

1. **Interface not up yet.** The ptp4l drop‑in above
   (`After=network-online.target`, `Restart=on-failure`, `RestartSec=2s`)
   addresses this.
2. **NIC can't do HW timestamping at all.** `ethtool -T eth0` will show no
   filters or `PTP Hardware Clock: none`. Switch to `time_stamping software`.
3. **Transport/filter mismatch.** `network_transport L2` but the driver only
   advertises UDP filters, or vice versa. Read `ethtool -T` carefully and
   match.
4. **Second `ptp4l` running.** See next item — this is the surprise.

### TX timestamp timeouts on hardware that used to work

Symptoms: `timed out while polling for tx timestamp`, port flaps between
LISTENING and UNCALIBRATED, never converges. If `tx_timestamp_timeout 100` is
already set and the issue is new on previously‑working hardware, check for a
**duplicate daemon**:

```bash
ps auxww | grep '[p]tp4l'
```

Two processes = two daemons racing for the same PHC and socket. The usual
cause is a leftover non‑templated unit from an older `linuxptp` version
(pre‑4.0 shipped `/usr/lib/systemd/system/ptp4l.service`, modern only ships
`ptp4l@.service`). To find stray references across the whole systemd tree:

```bash
sudo grep -rH 'ptp4l\.service\|phc2sys\.service' \
    /etc/systemd /run/systemd /usr/lib/systemd 2>/dev/null | grep -v '@'
```

On a clean modern Debian system the **only** expected hit is
`timemaster.service: Conflicts=... ptp4l.service ...` in the vendor unit file —
a harmless no‑op because the referenced unit doesn't exist. Anything else is a
leftover to investigate:

- A non‑templated `ptp4l.service` or `phc2sys.service` under
  `/usr/lib/systemd/system/` that `dpkg -S` reports as unowned — delete it.
- Full replacement units under `/etc/systemd/system/` that shadow vendor
  packages (`ntpd.service`, `gpsd.service` are common victims) —
  `systemctl revert <unit>` then recreate as drop‑ins.

Confirm package‑owned files haven't been hand‑edited:

```bash
dpkg -V linuxptp
```

The only expected flag is `??5?????? c /etc/linuxptp/ptp4l.conf` (your config,
marked as a conffile). Anything else in the output means a package file has
been modified in place.

### `Unit ptp4l.service not found` when starting phc2sys

The vendor `phc2sys@.service` has `Requires=ptp4l.service` / `After=ptp4l.service`
(non-templated) which on modern Debian doesn't exist — only `ptp4l@.service`
does. A drop-in *cannot* fix this: systemd does not allow dependency
directives (`Requires=`, `After=`, etc.) to be reset or subtracted from in a
drop-in, only added to. The vendor's broken references stay in effect no
matter what you put in a drop-in.

The fix is to override the whole unit with `systemctl edit --full phc2sys@.service`
(shown above in the server and client sections). If you previously tried a
drop-in approach with empty `Requires=` / `After=` lines expecting them to
reset, remove that drop-in first:

```bash
sudo rm -rf /etc/systemd/system/phc2sys@.service.d
sudo systemctl daemon-reload
```

Then do the full override.

To verify dependencies resolve correctly after the override:

```bash
systemctl show phc2sys@eth0.service -p Requires -p After
```

`Requires=` should contain only `ptp4l@eth0.service` plus systemd-auto-injected
targets (`sysinit.target`, `system-phc2sys.slice`). If bare `ptp4l.service`
still appears, the override didn't take — check with `systemctl cat
phc2sys@eth0.service` that it's reading your `/etc/systemd/system/phc2sys@.service`
and not the vendor file.

### Drop‑in file rules cheat sheet

- A drop‑in is a **patch**, not a replacement. Include only what differs from
  the vendor unit.
- **Exec‑type list directives** (`ExecStart=`, `ExecStartPre=`, `ExecStop=`,
  etc.) and other list directives like `Environment=`, `EnvironmentFile=`,
  `ConditionPathExists=` can be reset by an empty assignment, then reassigned:

  ```ini
  [Service]
  ExecStart=
  ExecStart=/new/command
  ```

- **Dependency directives cannot be reset in a drop‑in.** `Requires=`,
  `After=`, `Wants=`, `Before=`, `Conflicts=`, `Requisite=`, `BindsTo=`,
  `PartOf=` — these can only be *added to* via drop‑in. To remove a dependency
  declared in the vendor unit, you have to override the whole unit with
  `systemctl edit --full <unit>` and edit the offending line out of the local
  copy.
- `[Install]` sections in drop‑ins are ignored. Enable/disable is done via
  `systemctl enable/disable`.
- Always verify with `systemctl cat <unit>` (merged view) after editing, and
  `systemd-analyze verify <unit>` to catch syntax problems before trying to
  start. `systemctl show <unit> -p Requires -p After` shows what the parser
  actually resolved, which is the authoritative view.
- Never edit files under `/usr/lib/systemd/system/` or `/lib/systemd/system/`
  directly — they get overwritten on package upgrades. Use
  `systemctl edit <unit>` (instance drop‑in), `systemctl edit <template>@.service`
  (template‑level drop‑in), or `systemctl edit --full <unit>` (full override).

### Clients off by exactly 37 seconds

Almost always a UTC↔TAI handling bug:

- Server's `ntpsec` has no leapfile configured → kernel TAI offset is 0 →
  `phc2sys -w` copies UTC to PHC without offset → `ptp4l` advertises a PHC
  that's TAI‑labeled but UTC‑valued.
- Or someone put `ntpshm` as a servo inside `ptp4l.conf` instead of routing
  SHM through `phc2sys -E ntpshm`. The two don't apply the offset the same way.

Check the server:

```bash
sudo phc_ctl /dev/ptp0 cmp       # should be ~+37 s
sudo pmc -u -b 0 "GET TIME_PROPERTIES_DATA_SET"   # currentUtcOffset 37
cat /etc/ntpsec/ntp.conf | grep leapfile
```


## Known limitations

**No automatic clockClass demotion on GNSS loss.** If the server loses GPS
lock, `ptp4l` continues advertising `clockClass 6` while its PHC runs free off
the crystal. Clients keep trusting it. A production setup needs an external
watchdog that monitors `gpsd` (e.g. via `gpspipe -w`) and, on sustained loss
of fix, either stops `ptp4l@eth0.service` or rewrites `ptp4l.conf` with
`clockClass 7` (holdover) or `clockClass 52` (degraded) and reloads.

`phc2sys` can't hot‑swap direction either, so a single‑box bidirectional
fallback (become a client when GPS is lost) isn't possible with the current
tooling — it needs orchestration on top.


## License

This documentation: CC0 / public domain. Use, modify, and redistribute
without restriction.
