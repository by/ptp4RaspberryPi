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
  feeding it to `ntpsec` as a refclock — or, with an `ntpsec` that has the
  PHC refclock driver, reading the PHC directly (see
  [Option B](#option-b-ntpsec-phc-refclock)).
- **Service names**: Debian's `ntpsec` package runs `ntpsec.service`; an
  `ntpsec` built from source and installed with `./waf install` runs
  `ntpd.service` and ships `ntp-wait.service`. The units below list both
  names where it matters — ordering against a unit that doesn't exist is
  silently ignored, so listing both is harmless.


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
and the PHC, and it applies the UTC↔TAI offset when it crosses that
boundary. Doing this inside `ptp4l.conf` (`ntpshm` as a servo there) uses
subtly different semantics and silently produces 37‑second errors on
clients. (On the client side, an `ntpsec` with the PHC refclock driver can
replace `phc2sys -E ntpshm` and the SHM hop; see
[Option B](#option-b-ntpsec-phc-refclock).)

Where does the 37 s come from? `phc2sys -w` takes it from `ptp4l`'s
`currentUtcOffset` — on the grandmaster that is `ptp4l`'s own setting, not
the kernel or `ntpsec`: `ptp4l`'s `utc_offset` setting (default 37). Out of
the box `ptp4l` announces it with `currentUtcOffsetValid 0`, never changes
it after a leap second, and never announces one (see
[Known limitations](#known-limitations)).


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

`ntpsec` needs the leap‑seconds file to handle leap seconds itself and to
set the kernel's TAI offset; the PHC refclock (`flag1`) reads the same
file. (The server's `phc2sys -w` does *not* use it: it takes the
offset from `ptp4l`, see [Architecture](#architecture).)

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

# --- Raspberry Pi CM4/CM5/Pi5 quirks (same as on the clients) --------------
# The grandmaster hardware-timestamps every Sync it sends, so the slow TX
# timestamp path bites here at least as much as on a client.
tx_timestamp_timeout    100
ptp_minor_version       0
```

Note: no `ntpshm_segment` here — the server does not feed SHM, only clients do.

`clockClass`, `clockAccuracy`, `timeSource` and `utc_offset` (default 37)
are fixed at startup; see [Known limitations](#known-limitations).

### ptp4l systemd unit

Debian's `linuxptp` 4.x ships the template unit
`/usr/lib/systemd/system/ptp4l@.service`. The vendor unit's one rough edge is
that it doesn't wait for the network to be fully up, which on Pis can race
against PHY initialization and produce
`ioctl SIOCSHWTSTAMP failed: Invalid argument` at boot
(Debian bug [#1070847](https://bugs.debian.org/1070847)).

On the grandmaster there is a second, more important rule: **`ptp4l` must
not start before the system clock is right.** Otherwise it announces
clockClass 6 for a clock that is still being set, and `phc2sys` copies that
clock into the PHC. So `ptp4l` waits for `ntp-wait` (ntpd synchronized) and
then, up to 30 minutes, until ntpd has selected PPS.

Fix both via a drop‑in, **not** by editing the vendor file:

```bash
sudo systemctl enable ntp-wait.service   # ships with ntpsec; see note below
sudo systemctl edit ptp4l@eth0.service
```

Add only:

```ini
[Unit]
After=network-online.target ntp-wait.service
Wants=network-online.target ntp-wait.service

[Service]
Restart=on-failure
RestartSec=2s
# Wait (max 30 min) until ntpd has selected PPS, then start anyway.
TimeoutStartSec=35min
ExecStartPre=-/usr/bin/timeout 30m /bin/sh -c 'until /usr/local/bin/ntpq -c "rv 0 refid" | grep -q "refid=kPPS"; do sleep 5; done'
```

- Check the refid string once PPS is selected: `ntpq -c "rv 0 refid"` must
  print exactly what the `grep` looks for (here `refid=kPPS`, from
  `refid kPPS` on the PPS refclock line). Adjust the `ntpq` path
  (`/usr/bin/ntpq` for Debian's package).
- `ExecStartPre` counts against the start timeout, hence `TimeoutStartSec`.
  The leading `-` lets `ptp4l` start anyway after the 30 minutes.
- `ntp-wait.service` is the unit shipped by an `ntpsec` built from source.
  Check `systemctl list-unit-files | grep -i wait` for the name on your
  system.
- With `minpoll 10` on the PPS refclock, the first PPS sample can take ~17
  minutes after ntpd synchronizes. `minpoll 4 maxpoll 4` on the PPS line
  shortens the wait to about a minute.

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
After=ntp-wait.service ntpd.service ntpsec.service ptp4l@%i.service
Before=time-sync.target

[Service]
Type=simple
ExecStart=/usr/sbin/phc2sys -s CLOCK_REALTIME -c %I -w -q

[Install]
WantedBy=multi-user.target
```

Key changes from the vendor: the bare `ptp4l.service` references are replaced
with the templated `ptp4l@%i.service`; `ntp-wait.service` (and the ntpd unit
under either name) is added to `After=` so that the system clock is
disciplined before we start copying it to the PHC; and `ExecStart=` switches
to server direction (`-s CLOCK_REALTIME -c %I`). Note that
`After=ntpsec.service` alone only waits for ntpd to *start*, not to
synchronize — and does nothing at all where the unit is called `ntpd.service`.

Make sure there is no per-instance copy
`/etc/systemd/system/phc2sys@eth0.service`: it would silently override this
template for eth0 (`systemctl cat phc2sys@eth0.service | head -1` must show
`phc2sys@.service`).

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

### Boot ordering pitfalls

With `ptp4l` waiting for ntpd to synchronize, **nothing that ntpd needs in
order to synchronize may be ordered after `ptp4l` or `ntp-wait`.** Leftover
`After=ptp4l@eth0.service` lines from an older setup create loops:

- in an ntpd drop-in → systemd reports an *ordering cycle* and silently
  drops the `ntp-wait` job (nothing waits any more);
- in gpsd's drop-in, or in a unit gpsd `Requires=`/`BindsTo=` (an NTRIP
  relay, for instance) → no cycle is reported, but boot **hangs** until
  `ntp-wait` times out (`ntpwait -n 30000 -s 1`, about 8 hours), because
  ntpd cannot synchronize without gpsd.

The intended order is: GNSS chain (gpsd and what it needs) and ntpd →
`ntp-wait` → `ptp4l` (waits for PPS) → `phc2sys`.
Check before rebooting:

```bash
# must print nothing:
systemctl show ntpd.service ntpsec.service gpsd.service -p After \
    | tr ' ' '\n' | grep -E "ntp-wait|ptp4l"
# no "ordering cycle" lines:
systemd-analyze verify ntpd.service gpsd.service ptp4l@eth0.service phc2sys@eth0.service
```

and after a reboot:

```bash
journalctl -b -o short-precise \
    | grep -E "Started (gpsd|ptp4l|phc2sys)|ntpd\[.*Starting|Finished ntp-wait"
```

If a service that gpsd depends on needs correct time (e.g. TLS to an NTRIP
caster), give it `Restart=on-failure` instead of ordering it after
`ntp-wait`.

### Optional: watch the PHC from ntpsec

With an `ntpsec` that has the PHC refclock driver, ntpd can continuously
measure the whole system clock → `phc2sys` → PHC (TAI) chain. On the
grandmaster this must be **`noselect`** — the PHC is fed *from* the system
clock, so selecting it would make ntpd and `phc2sys` steer each other:

```
refclock phc path eth0 flag1 1 flag4 1 minpoll 4 maxpoll 4 refid PTP noselect
```

`ntpq -p` then shows the PHC's offset from the system clock (expect well
under a microsecond once `phc2sys` has locked; ±37 s means a UTC/TAI
mismatch). Until `phc2sys` runs after boot, the PHC is still on UTC and
reads −37 s; ntpd's jitter for this line stays large for a few polls after
`phc2sys` steps the PHC. Both are expected and harmless with `noselect`.

Likewise, if any of your PTP clients also serve NTP (refid `.PTP.`), mark
them `noselect` in the grandmaster's `ntp.conf`: they get their time from
this grandmaster, and ntpd's loop detection doesn't catch that.

### Server verification

```bash
# Exactly one ptp4l and one phc2sys, and phc2sys is in system→PHC direction:
ps auxww | grep -E '[p]tp4l|[p]hc2sys'

# PHC should be ~37 s ahead of UTC (i.e. TAI):
sudo phc_ctl /dev/ptp0 cmp

# ptp4l is advertising the current UTC offset and the right clockClass
# (currentUtcOffsetValid is 0 out of the box, see Known limitations):
sudo pmc -u -b 0 "GET TIME_PROPERTIES_DATA_SET"      # currentUtcOffset 37, ptpTimescale 1
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

### Option A: phc2sys → SHM → ntpsec

This works with any `ntpsec`. If yours has the PHC refclock driver, see
[Option B](#option-b-ntpsec-phc-refclock), which needs neither `phc2sys`
nor SHM on the client.

#### phc2sys systemd unit (client direction)

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
After=ntpd.service ntpsec.service ptp4l@%i.service

[Service]
Type=simple
ExecStart=/usr/sbin/phc2sys -s %I -E ntpshm -M 2 -w -q

[Install]
WantedBy=multi-user.target
```

- `-s %I` — source is the interface's PHC.
- `-E ntpshm` — servo writes to an NTP SHM segment instead of steering a local
  clock. (The `-h` help text lists only `pi` and `linreg`, but `ntpshm` and
  `nullf` are also valid.)
- `-M 2` — use SHM segment 2. Must match the `unit 2` in `ntp.conf`.
- `-w` — wait for `ptp4l` to lock, then inherit the UTC↔TAI offset from it.
  Without this (or `-O <offset>`), phc2sys refuses to start with *time offset
  must be specified using -w or -O*. `-w` is the better choice because the
  offset tracks any future leap-second changes automatically.
- No `Before=time-sync.target` here — on the client, `ntpsec` is the authority;
  we don't want to signal "time is synced" before `ntpsec` has actually locked
  onto the SHM refclock.

Why not `-a` (automatic mode)? `-a` has phc2sys query ptp4l's management
interface and dynamically follow whichever port ptp4l has selected as slave —
useful for boundary-clock hosts with multiple PTP ports that can fail over.
For a single-NIC client feeding ntpsec, there's nothing dynamic to follow, so
the simpler `-s %I -E ntpshm -M N -w` form is canonical and matches the
examples in the Red Hat, Fedora, and linuxptp documentation.

```bash
sudo systemctl enable --now phc2sys@eth0.service
```

#### ntpsec refclock

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

### Option B: ntpsec PHC refclock

An `ntpsec` built with the PHC refclock driver (`refclock phc`, driver 47)
reads the client's PHC directly: no `phc2sys`, no SHM segment, no
37‑second handling outside ntpd. Keep `ptp4l` exactly as above — it is what
locks the client PHC to the grandmaster — and:

```bash
sudo systemctl disable --now phc2sys@eth0.service
```

In `ntp.conf`:

```
leapfile /usr/share/zoneinfo/leap-seconds.list
refclock phc path eth0 flag1 1 refid PTP prefer
```

- `path eth0` picks the PHC of that interface (or give `/dev/ptp0`).
- `flag1 1`: the PHC runs on TAI; the driver subtracts the TAI−UTC offset
  from the leap‑seconds file (required — without it the samples are
  discarded). It uses the client's own file, not the grandmaster's
  announced offset, so keep `tzdata` current.
- Each second the driver takes a burst of up to 25 PHC readings and
  averages the ones with the shortest read latency; it uses hardware
  cross‑timestamping when the NIC offers it. The startup log line
  `PHC(0): using /dev/ptpN via …` shows which method was found.
- The `ntpshm_segment` line in the client's `ptp4l.conf` is no longer needed.

The driver cannot tell whether `ptp4l` is locked; neither can the SHM
path. Watch `ptp4l`'s log, or `pmc -u -b 0 "GET TIME_STATUS_NP"`.

### Calibrating path asymmetry

A stable residual offset (typically tens to hundreds of µs) between client and
server after full convergence usually indicates an asymmetric network path —
different physical delays in each direction. Unmanaged switches are the
common cause. PTP assumes the two directions are symmetric and splits the
round‑trip in half, so an asymmetric path appears as a fixed bias.

Measure the mean offset of the PTP refclock against an independent
reference (a GNSS/PPS source on the client, or the grandmaster itself) over
an hour of quiet operation. The bias you see *is* the error PTP made —
half of the path asymmetry — so correct it in full, not by half.

Preferably correct it in `ptp4l` on the client, so the PHC itself (and
everything downstream) is right. `delayAsymmetry` is in nanoseconds and,
per `ptp4l(8)`, positive when the master‑to‑slave path is the slower one.
Sign conventions are easy to get backwards, so set it to the measured bias,
restart `ptp4l`, and check: if the bias doubled, flip the sign; if it only
halved, double the value:

```ini
# client ptp4l.conf, [global] or the port section
delayAsymmetry          180000      # example: 180 µs
```

Alternatively correct it in ntpsec with `time1` — **not** `time2`: for the
SHM driver `time2` is a sanity limit, and values below 1 s are ignored, so
`time2 0.000180` has no effect. `time1` is added to every sample, so use the
opposite sign of the measured offset:

```
refclock shm unit 2 refid PTP time1 -0.000180    # offset was +180 µs
```

(Same for `refclock phc`.) Better still: replace the unmanaged switch with
a PTP transparent clock or run a direct cable.


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

- The grandmaster's `ptp4l` announces the wrong `currentUtcOffset` (check
  `GET TIME_PROPERTIES_DATA_SET` below). `phc2sys -w` on the server applies
  whatever `ptp4l` says, and the clients' `phc2sys -w` takes it from the
  announce messages. It comes from `utc_offset` in the server's `ptp4l.conf`.
- Client with the PHC refclock (Option B): `flag1 1` missing. (A missing
  leapfile doesn't cause 37 s — the driver then discards the samples and
  logs a warning.)
- Someone put `ntpshm` as a servo inside `ptp4l.conf` instead of routing
  SHM through `phc2sys -E ntpshm`. The two don't apply the offset the same way.
- Only right after the grandmaster boots: its PHC is still on UTC until
  `phc2sys` runs. With the boot ordering above, `ptp4l` isn't announcing
  yet at that point.

Check the server:

```bash
sudo phc_ctl /dev/ptp0 cmp       # should be ~+37 s
sudo pmc -u -b 0 "GET TIME_PROPERTIES_DATA_SET"   # currentUtcOffset 37
grep leapfile /etc/ntpsec/ntp.conf /etc/ntp.conf 2>/dev/null
```


## Known limitations

**No automatic clockClass demotion or leap-second announcement.** `ptp4l`'s
`clockClass`, `timeSource` and `utc_offset` are fixed at startup, and it
announces `currentUtcOffsetValid 0`. If the server loses GPS lock, it keeps
advertising clockClass 6 while its PHC runs free off the crystal, and it
never announces a leap second (after one, `utc_offset` must be raised by
hand). The only runtime way to change these is
`pmc -u -b 0 "SET GRANDMASTER_SETTINGS_NP ..."`, driven by an external
watchdog that follows ntpd (e.g. `ntpq -c "rv 0 refid"`) and the
leap‑seconds file. On the clients, `clock_class_threshold` in `ptp4l.conf`
limits which clockClass they accept as locked.

`phc2sys` can't hot‑swap direction either, so a single‑box bidirectional
fallback (become a client when GPS is lost) isn't possible with the current
tooling — it needs orchestration on top.


## License

This documentation: CC0 / public domain. Use, modify, and redistribute
without restriction.
