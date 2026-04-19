> [!WARNING] 
> This is optional! Proceed with caution and at your own risk!

While there is quite some information available how to properly set up PTP on your Raspberry device, e.g.,
 - https://github.com/tiagofreire-pt/rpi_uputronics_stratum1_chrony/blob/main/steps/advanced_system_tuning.md
 - https://gpsd.gitlab.io/gpsd/gpsd-time-service-howto.html#_providing_local_ntp_service_using_ptp
 - https://docs.fedoraproject.org/en-US/fedora/latest/system-administrators-guide/servers/Configuring_PTP_Using_ptp4l/#sec-Serving_NTP_Time_with_PTP
 - https://sourceforge.net/p/linuxptp/mailman/linuxptp-devel/thread/1424738292.9759.53.camel%40intel.com/?page=1

not everything works right out of the box with my particular setup, a system with a highly reliable clock, driven by GNSS with PPS, and a set of clients which should be served by PTP due to its higher precision of the time signal; thus, I've compiled this 'best of'. – Thank you very much to the authors of above documentation for their valuable insights, particularly @tiagofreire-pt and the contributors to the linuxptp-devel mailing list!


# Preparation for both server and client side


## Reducing ethernet coalescence on RX and TX
(Adjusted from https://github.com/tiagofreire-pt/rpi_uputronics_stratum1_chrony/blob/main/steps/advanced_system_tuning.md)

Every ethernet adaptor uses the coalescence method to gather packets and send them in a bulk, for better throughput efficiency.

But this method introduces valuable latency and jitter on NTP or PTP-E2E packets.

Check the defaults:
> sudo ethtool -c eth0

```
Coalesce parameters for eth0:
Adaptive RX: n/a  TX: n/a
stats-block-usecs: n/a
sample-interval: n/a
pkt-rate-low: n/a
pkt-rate-high: n/a

rx-usecs: 49
rx-frames: n/a
rx-usecs-irq: n/a
rx-frames-irq: n/a

tx-usecs: 49
tx-frames: n/a
tx-usecs-irq: n/a
tx-frames-irq: n/a

rx-usecs-low: n/a
rx-frame-low: n/a
tx-usecs-low: n/a
tx-frame-low: n/a

rx-usecs-high: n/a
rx-frame-high: n/a
tx-usecs-high: n/a
tx-frame-high: n/a

CQE mode RX: n/a  TX: n/a
```

Both `rx-usecs` and `tx-usecs` have a value of 49 microseconds.

We'll set both to the minimum accepted by the Raspberry Pi ethernet driver:

> sudo ethtool -C eth0 tx-usecs 4
>
> sudo ethtool -C eth0 rx-usecs 4

To revert to the default values:

> sudo ethtool -C eth0 tx-usecs 49
>
> sudo ethtool -C eth0 rx-usecs 49

This could shave *circa* 40 usecs of response time over the `chrony ntpdata` statistics. Huge improvement on a NTP setup that has hardware timestamping and its higher accuracy.

To make it persistent over restarts, simply create a `systemd` service for `eth0_coalescence`:

> sudo nano /etc/systemd/system/eth0_coalescence.service

Add this:

```
[Unit]
Description=Setting Speed
Requires=network.target
After=network.target

[Service]
ExecStart=/usr/sbin/ethtool -C eth0 tx-usecs 4 rx-usecs 4
Type=oneshot

[Install]
WantedBy=multi-user.target
```

Then, enable and start the `eth0_coalescence` service:

> sudo systemctl enable --now eth0_coalescence.service

# Enable support for the PTP Hardware Clock (PHC) on the Ethernet chip
(Adjusted from https://github.com/tiagofreire-pt/rpi_uputronics_stratum1_chrony/blob/main/steps/advanced_system_tuning.md)

Raspberry CM4 and Pi5 have a PTP hardware clock within their Ethernet chips, so we leverage those to have another high performance reference clock in ntpsec.

Here is our setup and the related time modes (TAI: International Atomic Time (without leap seconds), UTC: Coordinated Universal Time (incl. leap seconds):

```
Server:
  GNSS ──► gpsd ──► ntpsec ──► system clock (UTC)
                                      │
                                      ▼
                                  phc2sys (automagically applies +37s leap offset)
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
                                  phc2sys -E ntpshm (automagically strips +37s)
                                      │
                                      ▼
                                   SHM(2) (UTC) ──► ntpsec
```

As `ntpsec` (or `ntp` for that matter) does not synchronise the NIC clock PHC itself, we need to use phc2sys to sync the system time with PHC and leverage `ptp4l` to provide the right number of leap seconds (currently 37), thus taking care of the conversion UTC-TAI and vice versa. – Please ensure that you have a reasonably current version of `linuxptp`, e.g., https://packages.debian.org/source/testing/linuxptp / https://salsa.debian.org/multimedia-team/linuxptp !

> sudo apt update && sudo apt install linuxptp -y


## Prepare the server side (GNSS/PPS grandmaster)

On the server, which has a highly precise time from its GNSS and PPS connection via ntpsec, we create a new file `/etc/linuxptp/ptp4l.conf` and prepare it to provide the exact time available from the system clock to the PHC, but also make sure that the system clock is not adjusted back from the PTP side:

> sudo nano /etc/linuxptp/ptp4l.conf

```
[global]
summary_interval 10  # syslog only every 1024 seconds

clockClass 255       # means slave-only. Without this,
                     # the client could theoretically advertise itself as a potential
                     # grandmaster to other PTP nodes, which is almost never what you want.
                     # Equivalent: add `-s` to ptp4l's command line.

# use one of the following values for clock accuracy:
# 0x20 25ns    0x24 2.5us    0x28 250us    0x2c 25ms    0x30 10s
# 0x21 100ns   0x25 10us     0x29 1ms      0x2d 100ms   0x31 more than 10s
# 0x22 250ns   0x26 25us     0x2a 2.5ms    0x2e 250ms   0xfe unknown
# 0x23 1us     0x27 100us    0x2b 10ms     0x2f 1s
clockAccuracy           0x23

clock_servo linreg   # linear regression (do _not_ use ntpshm here!)
free_running            1 # do NOT let PTP steer our system clock

# use one of the following for time source:
# 0x10 atomic clock  0x30 terrestrial radio  0x50 NTP       0x90 other
# 0x20 GPS           0x40 PTP                0x60 hand set  0xa0 int. oscillator
timeSource              0x20

[eth0]
delay_mechanism Auto
network_transport       L2
```

Now, we simply follow https://salsa.debian.org/multimedia-team/linuxptp/-/blob/master/debian/README.Debian?ref_type=heads :

Create a `systemd` service for `ptp4l` on interface `eth0`:

> sudo systemctl enable ptp4l@eth0
>
> sudo systemctl start ptp4l@eth0

Leave the `systemd` service for `ptp4l@eth0` as is (if `ptp4l@eth0.service` does not start up correctly after system boot, you might wish to include `After=network-online.target` and `Wants=network-online.target`, see https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1070847)

Create a `systemd` service for `phc2sys`:

> sudo systemctl enable phc2sys@eth0

Adjust the `systemd` service file for `phc2sys` to adjust the PTP hardware clock, based on the system clock, but wait until it is stepped in sync:

> sudo nano /lib/systemd/system/phc2sys@.service

```
[Unit]
Description=Synchronize system clock or PTP hardware clock (PHC)
Documentation=man:phc2sys
After=ntpsec.service
Requires=ptp4l@%i.service
After=ptp4l@%i.service
Before=time-sync.target

[Service]
Type=simple
ExecStart=/usr/sbin/phc2sys -s CLOCK_REALTIME -c eth0 -w

[Install]
WantedBy=multi-user.target
```

Then, enable and start the `phc2sys` service for interface `eth0`:

> sudo systemctl daemon-reload
>
> sudo systemctl enable --now phc2sys@eth0

One might think that `phc2sys@.service` could also leverage the NTPSHM service (i.e., `ExecStart=/usr/sbin/phc2sys -s CLOCK_REALTIME -c eth0 -w -E ntpshm -M 2` like on the clients' side, below), but this would leave the PHC unsynchronized and create an offset by those nasty 37 seconds of time difference between TAI and UTC between the clocks, as NTPSHM is only propagating the current time to the SHM2 memory slot and not synchronizing the PHC incl. the UTC-to-TAI adjustment.

Now check phc2sys is running with the correct direction:

> ps auxww | grep phc2sys

Should show `phc2sys -s CLOCK_REALTIME -c eth0 -w`.

Check the UTC offset is being advertised:

> sudo pmc -u -b 0 "GET TIME_PROPERTIES_DATA_SET"

Should include `currentUtcOffset 37` and `currentUtcOffsetValid 1`.

Check clock identities:

> sudo pmc -u -b 0 "GET PARENT_DATA_SET"

`grandmasterClockClass` should be 6, `grandmasterIdentity` should match your server's MAC-derived ID.

Compare PHC to system clock:

> sudo phc_ctl /dev/ptp0 get; date -u '+%Y-%m-%d %H:%M:%S.%N UTC'

In 2026, PHC should be **exactly 37 seconds ahead** of UTC (TAI time), not 0s and not some µs offset.

**The `phc2sys` log line format `<pair-id> offset ...` is a label pair (e.g., `eth0 sys`), not a direction indicator.**
The actual direction is determined by the command-line flags (`-s` = source, `-c` = target). Don't be fooled by the log line showing the same format before and after a direction fix.


## Prepare the client side

On the client, create a new file `/etc/linuxptp/ptp4l.conf` just with just this:

> sudo nano /etc/linuxptp/ptp4l.conf

```
[global]
summary_interval 10  # syslog only every 1024 seconds
clockClass 255       # means slave-only. Without this,
                     # the client could theoretically advertise itself as a potential
                     # grandmaster to other PTP nodes, which is almost never what you want.
                     # Equivalent: add `-s` to ptp4l's command line.
clock_servo linreg   # linear regression (do _not_ use ntpshm here!)

[eth0]
delay_mechanism Auto # This defaults to E2E unless a P2P peer is detected. If there are
                     # PTP-unaware switches between server and client, E2E suffers from
                     # asymmetric queueing delay, which appears as a fixed offset.
                     # For best results: direct cable between server and clients, or
                     # use a PTP-aware (boundary or transparent clock) switch.
```

Again, follow https://salsa.debian.org/multimedia-team/linuxptp/-/blob/master/debian/README.Debian?ref_type=heads :

Create a `systemd` service for `ptp4l`:

> sudo systemctl enable ptp4l@eth0
>
> sudo systemctl start ptp4l@eth0

Leave the `systemd` service for `ptp4l@eth0` as is.

Create a `systemd` service for `phc2sys`:

> sudo systemctl enable phc2sys@eth0

Adjust the `systemd` service for `phc2sys`:

> sudo nano /lib/systemd/system/phc2sys@.service

```
[Unit]
Description=Synchronize system clock or PTP hardware clock (PHC)
Documentation=man:phc2sys
After=ntpsec.service
Requires=ptp4l@%i.service
After=ptp4l@%i.service
#Before=time-sync.target    # intentionally omitted on clients

[Service]
Type=simple
# Client direction: PHC → SHM2 (not to CLOCK_REALTIME directly).
# -E ntpshm feeds SHM segment 2 for ntpsec to consume as a refclock.
# -M 2 selects SHM segment 2.
# -w waits for the system clock to be sane before starting.
ExecStart=/usr/sbin/phc2sys -a -rr -E ntpshm -M 2 -w

[Install]
WantedBy=multi-user.target
```

Do **NOT** replicate the `-E ntpshm` behavior inside `ptp4l.conf` (via `ntpshm` in `[global]`). `ptp4l`'s SHM writer uses different semantics from `phc2sys`'s and does not apply the UTC offset the same way. Mixing them causes silent time errors — clients receive TAI time labeled as UTC, which is off by 37 seconds. Always route SHM through `phc2sys -E ntpshm` only.

Then, enable and start the `phc2sys` service:

> sudo systemctl daemon-reload
>
> sudo systemctl enable --now phc2sys@eth0


## Finish the client side

First check with `sudo ntpshmmon` if SHM2 is properly propagated.

Now, add this new `refclock` 'ntpsec' configuration with

>  sudo nano /etc/ntpsec/ntp.conf

```
refclock shm unit 2 refid PTP
```

If you observe a persistent fixed offset (tens to hundreds of µs) on the client after convergence, this is typically network path asymmetry between server and client. It can be calibrated out with a `time2` fudge:
```
refclock shm unit 2 refid PTP time2 0.000180   # example: +180us
```
Measure the offset with `ntpq -pn` over an hour of stable operation and use half the mean offset as the `time2` value (PTP splits asymmetry evenly between directions).

Now check with `ntpmon` that your new `refclock` is working properly.


## Verification checklist

On the server:

```
# Direction is system → PHC
ps auxww | grep phc2sys | grep -v grep

# PHC is 37s ahead of UTC (TAI time)
sudo phc_ctl /dev/ptp0 get
date -u '+%Y-%m-%d %H:%M:%S.%N UTC'

# ptp4l advertising correct UTC offset
sudo pmc -u -b 0 "GET TIME_PROPERTIES_DATA_SET"

# ntpsec is the authoritative system clock source
ntpq -c "rv 0 refid,stratum"
```

On the client:

```
# Direction is PHC → SHM
ps auxww | grep phc2sys | grep -v grep

# SHM(2) is being written with small offsets
sudo ntpshmmon -c 10

# ntpsec sees PTP as a low-jitter refclock
ntpq -pn

# Coalescence is tuned (same as server!)
sudo ethtool -c eth0 | grep usecs
```


## Yet unsolved: Enable the reverse correction direction in case GNSS reception fails; this is currently not possible as `phc2sys` can't switch servos on the fly:
`ptp4l` does NOT dynamically demote `clockClass` when the GNSS signal is lost. If your GPS fails, the server continues advertising `clockClass 6` with a free-running crystal, misleading clients into trusting bad time. For production use, consider a watchdog that stops `ptp4l@eth0.service` when gpsd reports loss of lock (via `gpspipe -w` or similar).
