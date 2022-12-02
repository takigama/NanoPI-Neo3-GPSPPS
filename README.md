# Chrony with GPS and PPS on the NanoPI Neo 3

## Startin Up The Shields...

First, we need to get armbian on the NanoPI, its the best OS for what we're trying
to achieve. I leave this as an excersize for the viewer...


## Modify Armbian

We want to disable a few things before we get started, things we wont need (feel free
to leave them running if you want them though). Lets disable smartmontools, gpsd socket
service and the wpa_supplicant (Neo's dont have wifi anyways).

Run the following commands (we'll also add picocom which will be handy later):
```
systemctl disable smartmontools.service
systemctl disable gpsd.socket
systemctl disable wpa_supplicant.service
apt update; apt install -y picocom
```

Next, we need to tell the OS which gpio pin we want to use for the pps signal, in my
case, im using the gpio labeled "GPIO2_A2/IR-RX" which is pin 7 on the breakout header
or gpio 66 from the perpective of linux

Running the following command will apply this setup to armbian and gpio66 will become
the pps device connected to /dev/pps0

```
armbian-add-overlay ppsoverlay.dts
```

reboot.... and when you log back in, you should find two pps devices:
```
root@nanopineo3:~# ls -al /dev/pps*
crw------- 1 root root 250, 0 Dec  3 04:34 /dev/pps0
crw------- 1 root root 250, 1 Dec  3 04:34 /dev/pps1
```

pps0 SHOULD be the gpio, and pps1 will be connected to the serial port, however there
is no CD line on the NanoPI so that one is fundamentally useless to us.

poweroff the router and connect the gps (see the connections section)

When you boot back up, the gps should now be running at 115200 baud, and gpsd will be 
running:

```
root@nanopineo3:~# ps -ef |grep -i gp[s]
gpsd        1448       1  0 04:34 ?        00:00:00 /usr/sbin/gpsd -N -n -S 2947 /dev/ttyS1
```

Sweet, lets kill gpsd for now, "killall gpsd" as root and lets watch the serial output
from the gps:

```
root@nanopineo3:~# picocom -b 115200 /dev/ttyS1
picocom v3.1
....
initstring     : none
exit_after is  : not set
exit is        : no

Type [C-a] [C-h] to see available commands
Terminal ready
$GPZDA,174749.103,02,12,2022,,*5F
$GPRMC,174749.103,V,,,,,0.00,0.00,021222,,,N*46
$GPGGA,174749.103,,,,,0,0,,,M,,M,,*42
$GPGSA,A,1,,,,,,,,,,,,,,,*1E
$GPTXT,01,01,02,ANTSTATUS=OPEN*2B
$GPZDA,174750.103,02,12,2022,,*57
```

at this point, the gps doesnt have a lock and if you wish to, you can watch the NMEA
sentences until you see it gain a lock (again, I leave this as an exersize for the
viewer to figure out what that looks like).

Once we have a lock, leave picocom with ctrl-a, ctrl-q. Lets see if pps is being asserted

```
root@nanopineo3:~# ppstest /dev/pps0
trying PPS source "/dev/pps0"
found PPS source "/dev/pps0"
ok, found 1 source(s), now start fetching data...
source 0 - assert 1670003529.000226381, sequence: 42 - clear  0.000000000, sequence: 0
source 0 - assert 1670003530.000224961, sequence: 43 - clear  0.000000000, sequence: 0
source 0 - assert 1670003531.000224707, sequence: 44 - clear  0.000000000, sequence: 0
source 0 - assert 1670003532.000224454, sequence: 45 - clear  0.000000000, sequence: 0
```


This means our pps line is being asserted and life is awesome... The "assert" value should
be fairly stable and increment by one for each line (actually translates to the kernel
time when the assert occured) if you see this instead:

```
root@nanopineo3:~# ppstest /dev/pps0
trying PPS source "/dev/pps0"
found PPS source "/dev/pps0"
ok, found 1 source(s), now start fetching data...
time_pps_fetch() error -1 (Connection timed out)
time_pps_fetch() error -1 (Connection timed out)
time_pps_fetch() error -1 (Connection timed out)
time_pps_fetch() error -1 (Connection timed out)
```

It means either the gps isn't connected properly or the gps isnt actually sending a
pps signal yet, lets just wait for it to assert or check connectivity on the gps. Its also
possible that for some reason the pps line came up as pp1 (run ppstest /dev/pps1 to check
and if it does, adjust your settings to use pps1 instead)

Now, lets get the chrony going, copy pps.conf (from the github repo) to /etc/chrony/conf.d
and lets reboot again.

We need to determine the offset between the NMEA sentenses the L80 GPS is sending and when
the pps line is being asserted, so lets watch sourcestats to figure out the offset for pps.conf

```
root@nanopineo3:~# watch chronyc sourcestats
...
Every 2.0s: chronyc sourcestats                                                     nanopineo3: Sat Dec  3 04:55:52 2022

Name/IP Address            NP  NR  Span  Frequency  Freq Skew  Offset  Std Dev
==============================================================================
NMEA                       60  29    71   -172.176    222.115   +823ms    10ms
PPS                        20   8    23     -0.257      0.405   +764ms  3195ns
time.cloudflare.com         2   0    64     +0.000   2000.000   +764ms  4000ms
ntp2.ds.network             2   0    64     +0.000   2000.000   +764ms  4000ms
```

Leave this run for quite a while (30+ minutes) for it to stabilise, it should get to
about 50ms (NMEA line, offset coloumn)...

```
Every 2.0s: chronyc sourcestats                                                     nanopineo3: Sat Dec  3 04:57:05 2022

Name/IP Address            NP  NR  Span  Frequency  Freq Skew  Offset  Std Dev
==============================================================================
NMEA                        7   3     7   -706.007   3111.299    +54ms  2923us
PPS                         6   6     6     -0.526      5.999    +69us  3609ns
time.cloudflare.com         3   3   129     -0.923   2129.202   -126us   236us
ntp2.ds.network             3   3   129     +0.554    722.545   +135us   108us
```

But, that could be different based on the firmware on the L80, in any case, lets
assume its 50ms.

Edit /etc/chrony/conf.d/pps.conf and modify the first line (note 50 == 0.05):

```
refclock SHM 0 refid NMEA noselect poll 0 offset 0.05 noselect
```

Then restart chrony:

```
/etc/init.d/chrony restart
```

Watch the source stats again, and the NMEA line offset should never go above
200ms (chrony requires it to be below 400ms, but lets aim higher), in reality
if you've set the offset close enough, this number should never get above more
than 10ms.

Jump back into editing the /etc/chrony/conf.d/pps.conf and remove the "noselect"
from the end of the PPS line, should look like this:

```
refclock PPS /dev/pps0 refid PPS lock NMEA poll 0
```

And, lets reboot one more time.

adjust pps.conf offset

Once the router has rebooted, you should find it will stabilise out to something
along these lines (the L80 GPS is one of the best units i've ever used in terms
of stability, kicks the Neo6m and mkt units out the door):

```
...

```

The pps line should end with "+/- 290ns" when everything is nice and stable. And
thats where this guide ends, enjoy your super-accurate gps! There are quite a few
other optimisations you can do at this point, including the following:

1) Make the serial line more efficient.
2) hard-set the cpu clock (i.e. remove auto frequency changes)
3) minimise the OS
4) setup permissions for things to connect to the ntp server

Would love to hear from people who give it a go, the Neo3 platform is one of the
best platforms i've seen since the AR9331 based GLInet AR150 (POE) for accurate
time keeping and alot of fun to play with. I wish, however, that friendlyelec would
make another model that has:

1) Decent GPIO output (i.e. nice easily accessible pins)
2) POE Input (put the router on the end of a wire on the roof)
3) Hardware Timestamping (RTL8125BG)

Technically the NanoPI R5C/S have at least the last one, which I find very interesting
though it would have been perfect if the R5C had of had GPIO's and some way of adding
a POE module.


## Connection..

Need a thing here for connections - coming soon(tm)
