# Building a Secure Raspberry Pi Timeserver #

This project sets out to build a secure stratum 1 timeserver using a raspberry pi and a simple GPS exansion board. The intend is to provide all information needed to build such service for somebody who is reasonably provisient in linux system administration on a Raspberry Pi.

In NTP a stratum 1 clock receives it's time for an original reference clock (Stratum 0). In this case the reference system is the GPS system.


## Hardware ##

This setup uses
* a raspberry pi 3 B+
* a [Uputronics Rasberry Pi+ GPS Expansion board ][] (version 4.1 from 2016)
* an active GPS antenna with an SMA male connector.
* [Uputronics Rasberry Pi+ GPS Expansion board ]: https://store.uputronics.com/index.php?route=product/product&path=60_64&product_id=81 " Uputronics Rasberry Pi+ GPS Expansion board"

Hardware assembly is straightforward: plug the gpio header on top of the raspberry pi pins, then plug the board with the capacitor up on top of the header. Fix the lot using the small metal screws.


## Initial configuration ##

The initial conviguration described here is based on the raspbian release from september 2019. We asume the reader know how to set up a pi system.

This setup boots from an NFS mount, not to wear out the SD.

After clean install, enabling of SSH,  and the midration of the root system to NFS we ran `apt uprade`

There is a bunch of software that needs  to be installed in order to perform all tasks below.

    $ sudo apt-get install pps-tools git scons libncurses-dev python-dev \
    bc libcap-dev nettle-dev libgnutls28-dev libreadline-dev bison \
	asciidoctor cpufrequtils certbot


## Configuring GPS and PPS

Here we mostly follow [Network Time Foundation Stratum-1-Microserver HOWTO][]. An excelent resource if you want to set up a stratum 1 server using NTP.

First we have to make sure so that there is no interference between the different components on the serial bus.

Add the following line to `/boot/config.txt` to disable bluetooth.

    # Disable Bluetooth so serial-tty speed is no longer tied to CPU speed
    dtoverlay=pi3-disable-bt

Now that bluetooth is disabled we can disable the bluetooth deamon as well:

    systemctl disable hciuart


Create a symlink to the GPS device to make it easier to refer to. Put the following in a new file `/etc/udev/rules.d/10-pps.rules` to accomplish this.

    KERNEL=="ttyAMA0", SYMLINK+="gpsd0"


Edit the file `/boot/cmdline.txt`, and remove the following from the single line in the file:

    console=serial0,115200

This prevents it from spawning a login shell on the serial port that your GPS needs to use. (Ensure that the text remain a single line). While you are at it also add `nohz=off` to the end of the line. This option turns of the reduction in scheduling clock ticks that is usually enabled as a power saving feature, but will increas the jitter in our system.


Now edit the file /boot/config.txt, and add this line to the end of the

file:

    enable_uart=1



Also you want to disable any powersaving features of the CPU, anything that makes the system run more predictable helps clock stability.

    $ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
    ondemand
    ondemand
    ondemand
    ondemand
    $ sudo sh -
    # echo 'GOVERNOR="performance"' > /etc/default/cpufrequtils
	# systemctl restart cpufrequtils
	# exit
	$ cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
	performance
    performance
    performance
    performance
    $


### GPS PPS signal ###

When you boot the system and your antenna is in a place where it will 'see satelites' you will notice that the green led on the GPS board starts to flash (at first boot, this may take up to half an hour). This signal, pulse per second (PPS), can be made available to the pi by adding

    dtoverlay=pps-gpio,gpiopin=18

to `/boot/config.txt`

This uses the [Device Trees][] mechanism to signal the kernel to load the PPS-GPIO driver and find the PPS signal on pin 18.

After reboot check if the PPS is seen by the kernel like this:

    $ dmesg | grep pps
    [   11.920478] pps_core: LinuxPPS API ver. 1 registered
    [   11.920492] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
    [   11.942928] pps pps0: new PPS source pps@12.-1
    [   11.943021] pps pps0: Registered IRQ 166 as PPS source


and

    $ sudo ppstest /dev/pps0
    trying PPS source "/dev/pps0"
    found PPS source "/dev/pps0"
    ok, found 1 source(s), now start fetching data...
    source 0 - assert 1576067077.000842158, sequence: 655 - clear  0.000000000, sequence: 0
    source 0 - assert 1576067078.000844631, sequence: 656 - clear  0.000000000, sequence: 0
    source 0 - assert 1576067079.000843680, sequence: 657 - clear  0.000000000, sequence: 0
    source 0 - assert 1576067080.000843888, sequence: 658 - clear  0.000000000, sequence: 0
    [etc]


### GPS signal ###

All the setup above resulted in us being able to read out the GPS data that is delivered every second.. we only need to test that. 

    $ stty -F /dev/gpsd0 raw 9600 cs8 clocal -cstopb
    $ cat /dev/gpsd0
    MLMM?i? [...] ??5)?$GNTXT,01,01,01,More than 100 frame errors, UART RX was disabled*70
    $GNRMC,132750.00,A,5221.38193,N,00457.33136,E,1.164,,111219,,,A*63
    $GNVTG,,T,,M,1.164,N,2.155,K,A*3C
    $GNGGA,132750.00,5221.38193,N,00457.33136,E,1,05,2.82,-37.3,M,45.9,M,,*59
    $GNGSA,A,3,07,09,30,05,28,,,,,,,,4.30,2.82,3.25*15
    $GNGSA,A,3,,,,,,,,,,,,,4.30,2.82,3.25*17
    $GPGSV,3,1,11,02,04,219,,05,65,240,21,07,46,063,23,08,04,059,*75
    $GPGSV,3,2,11,09,07,100,27,13,46,279,,15,15,284,15,21,10,333,16*72
    $GPGSV,3,3,11,27,07,027,,28,24,148,22,30,77,107,16*4D
    $GLGSV,1,1,00*65
    $GNGLL,5221.38193,N,00457.33136,E,132750.00,A,A*73
    $GNRMC,132751.00,A,5221.38222,N,00457.33164,E,0.977,,111219,,,A*67
    $GNVTG,,T,,M,0.977,N,1.810,K,A*3C
    $GNGGA,132751.00,5221.38222,N,00457.33164,E,1,05,2.82,-37.5,M,45.9,M,,*50
    $GNGSA,A,3,07,09,30,05,28,,,,,,,,4.30,2.82,3.25*15
    $GNGSA,A,3,,,,,,,,,,,,,4.30,2.82,3.25*17
    $GPGSV,3,1,11,02,04,219,,05,65,240,21,07,46,063,23,08,04,059,*75
    $GPGSV,3,2,11,09,07,100,27,13,46,279,,15,15,284,14,21,10,333,17*72
    $GPGSV,3,3,11,27,07,027,,28,24,148,23,30,77,107,11*4B
    $GLGSV,1,1,00*65
	[...]

If you get stuck here see [the relevant section in the NTFs howto](https://www.ntpsec.org/white-papers/stratum-1-microserver-howto/#_smoke_test_the_gps_hat_combination)

## Building GPSD

GPSD is needed to get a reasonable wall-clock that in combination with the PPS signal will get us to a milisecond precision server.

The [Network Time Foundation Stratum-1-Microserver HOWTO][] suggests to build the deamon yourself.  So we follow their instructions i.e. clone the git repository and build it.

    $ git clone git://git.savannah.nongnu.org/gpsd.git
    $ cd gpsd
    $ scons timeservice=yes magic_hat=yes nmea0183=yes ublox=yes mtk3301=yes fixed_port_speed=9600 fixed_stop_bits=1
	$ sudo scons udev-install

Once you build everything you can test the lot with `gpsmon` which wil show you a good overview of visible satelites, your location and the time.

    $ sudo ./gpsmon /dev/gpsd0


    timeserv:/dev/gpsd0 9600 8N1  NMEA0183>
    ┌──────────────────────────────────────────────────────────────────────────────┐
    │Time: 2019-12-11T13:51:54.000Z Lat:  52 21.377260' N  Lon:   4 57.376350' E   │
    └───────────────────────────────── Cooked TPV ─────────────────────────────────┘
    ┌──────────────────────────────────────────────────────────────────────────────┐
    │ GNGSA GPGSV GLGSV GNGLL GNTXT GNRMC GNVTG GNGGA                              │
    └───────────────────────────────── Sentences ──────────────────────────────────┘
    ┌──────────────────┐┌────────────────────────────┐┌────────────────────────────┐
    │Ch PRN  Az El S/N ││Time:      135154.00        ││Time:      135154.00        │
    │ 0   5 217 59  18 ││Latitude:    5221.37726 N   ││Latitude:  5221.37726       │
    │ 1   7  62 35  10 ││Longitude:  00457.37635 E   ││Longitude: 00457.37635      │
    │ 2   8  50  9   0 ││Speed:     0.263            ││Altitude:  71.6             │
    │ 3  13 285 57   0 ││Course:                     ││Quality:   1   Sats: 05     │
    │ 4  15 288 24  12 ││Status:    A       FAA: A   ││HDOP:      1.41             │
    │ 5  21 324 13  21 ││MagVar:                     ││Geoid:     45.9             │
    │ 6  27  18  7   0 │└─────────── RMC ────────────┘└─────────── GGA ────────────┘
    │ 7  28 142 35  21 │┌────────────────────────────┐┌────────────────────────────┐
    │ 8  30  79 69  19 ││Mode: A3 Sats: 5 7 15 28 30 ││UTC:           RMS:         │
    │ 9                ││DOP: H=1.41  V=2.45  P=2.82 ││MAJ:           MIN:         │
    │10                ││TOFF:  0.096050537          ││ORI:           LAT:         │
    │11                ││PPS: -0.000102641           ││LON:           ALT:         │
    └────── GSV ───────┘└──────── GSA + PPS ─────────┘└─────────── GST ────────────┘
    (66) $GPGSV,3,1,09,05,59,217,18,07,35,062,12,08,09,050,,13,57,285,*7B
    (68) $GPGSV,3,2,09,15,24,288,10,21,13,324,22,27,07,018,,28,35,142,24*72
    (31) $GPGSV,3,3,09,30,69,079,20*40
    (18) $GLGSV,1,1,00*65
    (52) $GNGLL,5221.37842,N,00457.37869,E,135152.00,A,A*7D
    (69) $GNTXT,01,01,01,More than 100 frame errors, UART RX was disabled*70
    ------------------------------------- PPS -------------------------------------
    (68) $GNRMC,135153.00,A,5221.37838,N,00457.37818,E,0.650,,111219,,,A*66
    (35) $GNVTG,,T,,M,0.650,N,1.203,K,A*3E
    (74) $GNGGA,135153.00,5221.37838,N,00457.37818,E,1,05,1.41,71.8,M,45.9,M,,*75
    (52) $GNGSA,A,3,07,30,05,28,15,,,,,,,,2.82,1.41,2.44*1D
    (42) $GNGSA,A,3,,,,,,,,,,,,,2.82,1.41,2.44*12
    (66) $GPGSV,3,1,09,05,59,217,18,07,35,062,10,08,09,050,,13,57,285,*79
    (68) $GPGSV,3,2,09,15,24,288,09,21,13,324,22,27,07,018,,28,35,142,22*7C
    (31) $GPGSV,3,3,09,30,69,079,19*4A
    (18) $GLGSV,1,1,00*65
    (52) $GNGLL,5221.37838,N,00457.37818,E,135153.00,A,A*77
    ------------------------------------- PPS -------------------------------------
    



## Installing Chrony

Now we will divert from the  [Network Time Foundation Stratum-1-Microserver HOWTO][] as we want to use Chrony as the server - just for code diversity and because the Network Time Foundation NTP server does not yet support Network Time Security.

First download, build and install Chrony with time security.  We will use the [Chrony NTS GIT repository][]

As a non privileged user perform the following:

    $ git clone https://github.com/mlichvar/chrony-nts
	$ cd chrony-nts
	$ ./configure --enable-debug
	$ make
	$ sudo make install
	

You really want to make sure you configure with debug enabled and  that the Features line in the configure output shows: `+NTS`. That line should, if you installed all packages above, look remarkably similar to:

    Features : +CMDMON +NTP +REFCLOCK +RTC +PRIVDROP -SCFILTER -SIGND +ASYNCDNS +NTS +READLINE +SECHASH +IPV6 +DEBUG

Note: If you do not see +NTS then make sure that you have actually installed `nettle-dev` and `libgnutls28-dev` in the above.

Note: The Chrony INSTALL instructs us to install a `timepps.h` file to work with the PPS. Fortunatelly, our raspbian installation comes with one in `/usr/include/sys/timepps.h` which the configure script finds by default.

After installing Chrony (default install location `/usr/local`) you should edit a config file.


    driftfile /var/lib/chrony/drift
    
    allow
    
    # set larger delay to allow the NMEA source to overlap with
    # the other sources and avoid the falseticker status
    refclock SHM 0 refid GPS precision 1e-1  delay 0.2
    refclock SOCK /var/run/chrony.gpsd0.sock refid PPS


TODO: a proper starting script


    # killall -9 gpsd chronyd
    # chronyd -f /etc/chrony.conf
    # sleep 2
    # gpsd -n /dev/gpsd0
    # sleep 2
    # cgps

## Turning on NTS security.

First test the setup in client mode.

In order to do so add a known timeserver. Cloudflare runs one on  `time.cloudflare.com:1234` and NetNod runs three at `nts.ntp.se:4443`, `nts.sth1.ntp.se: 4443`, and `nts.sth2.ntp.se: 4443`

In order to secure our server with NTS we will need a certificate. We will use Let's Encrypt as the certificate provider. Since we already installed `certbot` above we can go straight ahead.

Since there is no active webserver running on our system we can use `certbot` in its stand-alone mode.  I am using the name 'timemaster.example.com' as an example. You have to make sure that your system is named appropriatly and that it resolves through the DNS.

     sudo certbot certonly --standalone --preferred-challenges http -d  timemaster.example.com

The certificate and private key  you generated are stored in

    /etc/letsencrypt/live/timemaster.example.com/


In order to make sure the certificates are renewed timely you want to have add the following line to your root's crontab:

    0 1 * * * perl -le 'sleep rand 9000' && /usr/bin/certbot renew

TODO: POST RENEW HOOK (that calls the start script that we also have
to do above)

After you configured the certificates load them in the `chrony.conf`:

    ntsserverkey /etc/letsencrypt/live/timemaster.evangineer.net/privkey.pem
    ntsservercert /etc/letsencrypt/live/timemaster.evangineer.net/fullchain.pem

You have to load the fullchain, otherwise clients cannot find their
way back to the root certificates.

Use openssl in client mode to test if your server speaks TLS1.3:

    openssl s_client -tls1_3 -connect timemaster.example.com:11443





## Trouble Shooting

Capture useful data.

    tshark -i eth0 -b filesize:42000 -b duration:86400 -b files:7 -w  tshark.pcap -P -f "port 123 or port 11443"

You really want to write these files to an NFS disk or a attached USB
drive - wearing out your SD card this way is not a good idea.

Load the capture file into wireshark.



## Background reading ##

There are a great many resources that will help you to get up to speed on NTP, NTP security, and the use of GPS for time keeping on a raspberry pi. Your favorite search engine will be able to find all you need, but if your search skills fail, here are some of the sources we used.

#### NTP ####

For setting up accurate time with GPS and PPS information there were a few resources I used:

* [Network Time Foundation Stratum-1-Microserver HOWTO][]
* [GDPS Howto][]
* This [5 minute guide][] is the authoritiative source to get the [Uputronics Rasberry Pi+ GPS Expansion board ][]  working with NTP. 
* The [Rasberry Pi as a stratum 1 NTP server][] page on a blog by David and Cecilia Tailor describes how to set up the Pi with a GPS to deliver PPS syncing.




#### NTP Security ####

* The [APNIC blog by Martin Langer][] provides a technical overview of how NTP security works. It provides a good background on how a TLS based key exchange is used to provides the authentication cookies that are further used for exchange of messages over UDP.

* This [The Netnod NTS Howto][] describes how to set up an NTPS client to get time from their stratum 0 cesium clock in a secure way.


#### Raspberry Pi and GPS boards ####
The [Ostfalia Raspberry Pi NTP/NTS][] is a seriously cool interface to
a running PI. The "View filesystem" buttonm exposes some of the configuration.



 



#### Chrony and PPS ####

* [The RobotsForRobotics blog on chrony with GPS][] describes how to get PPS working with Chrony




[5 minute guide]: https://ava.upuaut.net/?p=951 "5 minute guide to making a GPS Locked Stratum 1 NTP Server with a Raspberry Pi"
[APNIC blog by Martin Langer]: https://blog.apnic.net/2019/11/08/network-time-security-new-ntp-authentication-mechanism/ "Network Time Security: new NTP authentication mechanism"
[Chrony NTS GIT repository]: https://github.com/mlichvar/chrony-nts "Chrony NTS repository"
[Device Trees]: https://www.raspberrypi.org/documentation/configuration/device-tree.md "Device Trees"
[Network Time Foundation Stratum-1-Microserver HOWTO]]: https://www.ntpsec.org/white-papers/stratum-1-microserver-howto "Stratum-1-Microserver HOWTO"
[Rasberry Pi as a stratum 1 NTP server]: https://www.satsignal.eu/ntp/Raspberry-Pi-NTP.html "The Raspberry Pi as a Stratum-1 NTP Server"
[The Netnod NTS Howto]: https://www.netnod.se/time-and-frequency/how-to-use-nts "How to use NTS"
[The RobotsForRobotics blog on chrony with GPS]: http://robotsforroboticists.com/chrony-gps-for-time-synchronization/  "chrony with GPS for Time Synchronization "

[GDPS Howto]: https://gpsd.gitlab.io/gpsd/gpsd-time-service-howto.html "GPSD Time Service HOWTO"

[Ostfalia Raspberry Pi NTP/NTS]: http://nts3-e.ostfalia.de/ "Ostfalia's PI"

