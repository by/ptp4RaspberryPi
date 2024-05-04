> [!WARNING] 
> This is optional! Proceed with caution and at your own risk!

While there is quite some information available how to properly set up PTP on your Raspberry device, e.g.,
 - https://github.com/tiagofreire-pt/rpi_uputronics_stratum1_chrony/blob/main/steps/advanced_system_tuning.md
 - https://gpsd.gitlab.io/gpsd/gpsd-time-service-howto.html#_providing_local_ntp_service_using_ptp
 - https://docs.fedoraproject.org/en-US/fedora/latest/system-administrators-guide/servers/Configuring_PTP_Using_ptp4l/#sec-Serving_NTP_Time_with_PTP
 - https://sourceforge.net/p/linuxptp/mailman/linuxptp-devel/thread/1424738292.9759.53.camel%40intel.com/?page=1

not everything works right out of the box with my particular setup, a system with a highly relaibale clock, driven by GNSS with PPS, and a set of clients which should be serverd by PTP due to its higher precision of the time signal; thus, I've compiled this 'best of'. – Thank you very much to the authors of above documentation for their valuable insights, particularly @tiagofreire-pt and the contributors to the linuxptp-devel mailing list!


# Preparation


## Reducing ethernet coalescence on RX and TX
(Adjusted from https://github.com/tiagofreire-pt/rpi_uputronics_stratum1_chrony/blob/main/steps/advanced_system_tuning.md)

Every ethernet adaptor uses the coalescence method to gather packets and sent them on a bulk, for better thoughput efficiecy. 

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

We'll set both to the mininum accepted by the Raspberry Pi ethernet driver:

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

Here is our setup and the related time modes (TAI: International Atonic Time (without leap seconds), UTC: Coordinated Universal Time (incl. leap seconds):

```
GNSS --> gpsd --> chrony/ntpsec --> NTP --> phc2sys --> ptp4l --> PHC --> [network] --> PHC --> ptp4l --> phc2sys --> SHM --> ntpsec
time
type:TAI      UTC               UTC     UTC         UTC       TAI     TAI           TAI     TAI       UTC         UTC     UTC
```

As `ntpsec`(or `ntp`for that matter) does not synchronise the NIC	clock PHC itself, we need to use phc2sys to sync the system time with PHC and leverage `ptp4l` to provide the right number of leap seconds (currently 37), thus taking care of the conversion UTC-TAI and vice versa. –  Please ensure that you have a reasonaly current version of `linuxptp`, e.g., https://packages.debian.org/source/testing/linuxptp / https://salsa.debian.org/multimedia-team/linuxptp !

> sudo apt update && sudo apt install linuxptp -y


## Prepare the server side

On the server, which has a highly precise time from its GNSS and PPS connection via ntpsec, we create a new file `/etc/linuxptp/ptp4l.conf` and prepare it to provide the exact time available from the system clock to the PHC, but also make sure that the system clock is not adjusted back from the PTP side:

> sudo nano /etc/linuxptp/ptp4l.conf

```
[global]
# only syslog every 1024 seconds
summary_interval        10

# use one of the following values for clock accuracy:
# 6   The clock node synchronizes its time to the master reference time source.
#     PTP assigns a time table to the clock node. A clock node with time class 6
#     cannot become a member clock of any other clocks in the domain.
# 7   The former time class is 6. The clock node cannot synchronize its time to
#     a time source. It enters the reappointment mode and meets the reappointment
#     conditions. PTP assigns a time table to the clock node. A clock node with
#     time class 7 cannot become a member clock of any other clocks in the domain.
# 13  The clock node synchronizes its time to a time source. ARB assigns a time
#     table to the clock node. A clock node with time class 13 cannot become a
#     member clock of any other clocks in the domain.
# 14  The former time class is 13. The clock node cannot synchronize its time to
#     a time source. It enters the reappointment mode and meets the reappointment
#     conditions. ARB assigns a time table to the clock node. A clock node with
#     time class 14 cannot become a member clock of any other clocks in the domain.
# 52  The clock node with time class 7 becomes optional clock A because it does
#     not meet the reappointment conditions. A clock node with time class 52 cannot
#     become a member clock of any other clocks in the domain.
# 58  The clock node with time class 14 becomes optional clock A because it does
#     not meet the reappointment conditions. A clock node with time class 58 cannot
#     become a member clock of any other clocks in the domain.
# 187 The clock node with time class 7 becomes optional clock B because it does not
#     meet the reappointment conditions. A clock node with time class 187 can
#     become a member clock of another clock in the domain.
# 193 The clock node with time class 14 becomes optional clock B because it does
#     not meet the reappointment conditions. A clock node with time class 193 can
#     become a member clock of another clock in the domain.
# 248 Default time class value.
# 255 Clock node operating in slave-only mode.
clockClass              6

# use one of the following values for clock accuracy:
# 0x20 25ns    0x24 2.5us    0x28 250us    0x2c 25ms    0x30 10s
# 0x21 100ns   0x25 10us     0x29 1ms      0x2d 100ms   0x31 more than 10s
# 0x22 250ns   0x26 25us     0x2a 2.5ms    0x2e 250ms   0xfe unknown
# 0x23 1us     0x27 100us    0x2b 10ms     0x2f 1s
clockAccuracy           0x22
clock_servo             linreg
free_running            1

# use one of the following for time source:
# 0x10 atomic clock  0x30 terrestrial radio  0x50 NTP       0x90 other
# 0x20 GPS           0x40 PTP                0x60 hand set  0xa0 int. oscillator
timeSource              0x20

[eth0]
#delay_mechanism Auto
network_transport       L2
```

Now, we simply follow https://salsa.debian.org/multimedia-team/linuxptp/-/blob/master/debian/README.Debian?ref_type=heads :

Create a `systemd` service for `ptp4l` on intercafe `eth0`:

> sudo systemctl enable ptp4l@eth0
> sudo systemctl start ptp4l@eth0

Leave the `systemd` service for `ptp4l@eth0` as is.

Create a `systemd` service for `phc2sys`:

> sudo systemctl enable phc2sys@eth0

Adjust the `systemd` service file for `phc2sys` to adjust the PTP hardware clock, based on the system clock:

> sudo nano /lib/systemd/system/phc2sys@.service

```
[Unit]
Description=Synchronize system clock or PTP hardware clock (PHC)
Documentation=man:phc2sys
After=ntpd.service
Requires=ptp4l@I.service
After=ptp4l@I.service
Before=time-sync.target

[Service]
Type=simple
ExecStart=/usr/sbin/phc2sys -a -rr

[Install]
WantedBy=multi-user.target
```

Then, enable and start the `phc2sys` service for interface `eth0`:

> sudo systemctl start phc2sys@eth0

One might think that `phc2sys@.service` could also leverage the NTPSHM service (i.e., `ExecStart=/usr/sbin/phc2sys -a -rr  -E ntpshm -M 2` like on the clients' side, below), but this would leave the PHC unsynchronized and create an offset by those nasty 37 seconds of time difference between TAI and UTC between the clocks, as NTPSHM is only propagating the current time to the SHM2 memory slot and not synchronizing the PHC incl. the UTC-to-TAI adjsutment.


## Prepare the client side

On the client, create a new file `/etc/linuxptp/ptp4l.conf` just with just this:

> sudo nano /etc/linuxptp/ptp4l.conf

```
[global]
summary_interval 10  # syslog only every 1024 seconds
clock_servo linreg   # linear regression (do _not_ use ntpshm here!)

[eth0]
delay_mechanism Auto
```

Again, follow https://salsa.debian.org/multimedia-team/linuxptp/-/blob/master/debian/README.Debian?ref_type=heads :

Create a `systemd` service for `ptp4l`:

> sudo systemctl enable ptp4l@eth0
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
After=ntpd.service
Requires=ptp4l.service
After=ptp4l.service
#Before=time-sync.target

[Service]
Type=simple
ExecStart=/usr/sbin/phc2sys -a -rr -E ntpshm -M 2

[Install]
WantedBy=multi-user.target
```

The configuration of phc2sys is not quite straightforward here: with the option `-E ntpshm -M 2`, it does not change the systemclock directly but provides the time from PHC via phc2sys with correct leap secands thanks to ptp4l via SHM2 for chrony/ntpd/ntpsec to mix&match. – Do _not_ do this in ptp4l.conf!

Then, enable and start the `phc2sys` service:

> sudo systemctl start phc2sys@eth0


## Finish the client side

First check with `sudo ntpshmmon` if SHM2 is properly propagated.

Now, add this new `refclock` 'ntpsec' configuration with

>  sudo nano /etc/ntpsec/ntp.conf

```
refclock shm unit 2 refid PTP
```

Now check with `ntpmon` that your new `refclock` is working properly.


## Yet unsolved: Enable the reverse correction direction in case GNSS reception fails; this is currently not possible as `phc2sys` can't switch servos on the fly.
