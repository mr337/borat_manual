
Server
******

The server is the heart of BORAT. It pretty much does everything except convert
AC to DC for the devices. Nuff said here

Deployment
==========
The server is a HP Proliant G7 introduction server. It has performance, and stability
while being on the low side of enterprise servers. The G7 has plenty room for expansion
of memory, hard drives, and extra CPU. For all intents and purposed we nicknamed
the server BORAT.

BORAT is installed with x86_64 Ubuntu 12.04 server. It was chosen because Ubuntu 
provides stable releases and have long term support. RTS2 was also developed on 
Ubuntu, so that is a benefit.

BORAT is deployed with large hard drives for temporary image storage prior
to uploading to SDBV at MSU. The hard drives are build together using software RAID
10. That is the first two hard drives are in RAID 1 and the last two hardrives 
mirror the first two hardrives (in the hard drive bays) making it RAID 10.
This is done with software so there is some maintenance that needs to be done to
ensure proper operation.

Server Hardware
===============

RAID
----

As mentioned the server uses a software RAID to group the hard drives together.
Software RAID is deployed with mdadm. mdadm functions to create,
add, modify, grow, and manage the software RAID. A quick check of how the RAID
is doing is to run the command ``cat /proc/mdstat``::

    lhicks@borat:/home/lhicks# cat /proc/mdstat
    Personalities : [raid10] 
    md0 : active raid10 sda1[0] sdd1[3] sdc1[2] sdb1[1]
        1435542528 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]

or more details::

    lhicks@borat:/home/lhicks# cat /proc/mdstat
    Personalities : [raid10] 
    md0 : active raid10 sda1[0] sdd1[3] sdc1[2] sdb1[1]
    1435542528 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
    
    unused devices: <none>
    root@borat:/home/lhicks# mdadm --detail /dev/md0
    /dev/md0:
    Version : 1.2
    Creation Time : Thu Jan 26 22:36:07 2012
        Raid Level : raid10
        Array Size : 1435542528 (1369.04 GiB 1470.00 GB)
    Used Dev Size : 717771264 (684.52 GiB 735.00 GB)
        Raid Devices : 4
        Total Devices : 4
        Persistence : Superblock is persistent
    
        Update Time : Fri Apr 13 16:58:17 2012
        State : active 
        Active Devices : 4
        Working Devices : 4
        Failed Devices : 0
        Spare Devices : 0
    
        Layout : near=2
        Chunk Size : 512K
    
    Name : borat:0  (local to host borat)
        UUID : 4576b0da:e50a581c:f9dc6a63:d4d3d45c
        Events : 151
    
        Number   Major   Minor   RaidDevice State
        0       8        1        0      active sync   /dev/sda1
        1       8       17        1      active sync   /dev/sdb1
        2       8       33        2      active sync   /dev/sdc1
        3       8       49        3      active sync   /dev/sdd1

All the ``U`` are good things. We don't want to see ``F`` which means, 
failed. If any of them say ``F``, the hard drives will probably have 
to be replaced and introduced into the RAID.

Serial Card
-----------

The HP G7 only comes with a single DB9 serial port. This is common for new 
machines to not even have any or maybe a single one like the G7. Our setup
requires at least two serial ports. 

* LX200GPS Telescope
* AHE Dome Controller
* 240V UPS

In reality we only really care about the first two items. There is a 120V
UPS for other devices that we monitor via USB. In the future we might add
the 240V UPS to NUT but currently it is not needed so we only need two serial
ports. We added a PCIe-> Serial card. This card required drivers to be
build and the driver installed. 

Instructions and source can be found in the git repo under hardware/serial\_card.
Make sure you read them and it will save you loads of time. Especially the parts
written by Lee Hicks.


Software
========

BORAT requires a lot of software in order to run all the connected devices. 
The largest part of software is RTS2 itself but there are many other smaller 
packages that support RTS2.

Drivers
-------
All devices that RTS2 operates excluding the LX200, CCD and filter wheel were 
written here at MSU.

Hardware
********

BORAT obviously requires more hardware than just a server to collect images. BORAT
has around 6 main hardware pieces that are controlled and perform all the operations.

Dome
----
We have been blessed with a 7 foot AstroHaven Enterprice (AHE) Dome. By blessed, I mean been given
a white elephant. The problem with this dome is it is almost 100\% undocumented,
and quite frankly a piece of shit that fails all too often.

The 7 foot AstroHaven dome assembly consists of a fiberglass dome, two motors, two 220v polarity
switches, magnetic switches, and a controller. The only piece one really needs
to care about is communicating with the controller. The rest of the components 
are fairly dumb devices and work very well. 

RTS2 has a driver for the AstroHaven dome. The original controller had a simple
protocol which worked very well. The new controller
has a new feature that is a heartbeat. This is done by the dome controller sending
a status character (values are provided below) every two or three seconds giving the status
of the dome. This works well since the state of the dome doesn't need to be 
guessed or captured since the heartbeat indicates the current state.

This methodology makes writing good serial code a pain since
communicating with the dome has random heartbeat characters in it. So
the RTS2 AHE driver has a lot of code looping
over heartbeat character till a response character is encountered. 

Also, the controller does not buffer commands
or block while a command is processing. For example if the driver issues an ``open
A leaf`` command, the controller will process any open or closing command for 1
second. The result is that the dome leaf would move for 1 second and then stop.
To produce smooth openings/closures, it would be desirable to submit stacked
commands, which would be operated upon one after the other until the dome
reaches the fully open or closed state. Unfortunately, this is not how the controller
works. Rather, instead of queueing commands, the controller halts what 
it is doing and then processes the subsequent command. This results in a
jerky motion that is probably hard on the motors and the fiberglass leaves. 
For this reason the RTS2 AHE dome driver, unfortunately, sleeps for around half
a second before issuing the next command. 

This should be a good warning as to some problems encountered while developing
a simple driver and, more importantly, give answers to the question, "why did
he program it like this?".

Without further ado here is a list of the undocumented protocol that AHE decided
to use 

Commands to open and close dome

+-----+---------+-----+
|Send | Command | Ack |
+=====+=========+=====+
|A    |Open     |a    | 
+-----+---------+-----+
|A    |Close    |A    | 
+-----+---------+-----+
|B    |Open     |b    | 
+-----+---------+-----+
|B    |Close    |B    |  
+-----+---------+-----+

Acknowledgement of open and commands

+-----+---------+-----+
|Send | Command | Ack |
+=====+=========+=====+
|A    |Opened   |x    |
+-----+---------+-----+
|A    |Closed   |X    |
+-----+---------+-----+
|B    |Opened   |y    |
+-----+---------+-----+
|B    |Closed   |Y    |
+-----+---------+-----+

Heartbeat values

+-----+--------------+-----+
|Send |    Command   | Ack |
+=====+==============+=====+
|AB   |Open          |3    |
+-----+--------------+-----+
|A    |Open, B Closed|2    |
+-----+--------------+-----+
|B    |Open, A Closed|1    |
+-----+--------------+-----+
|AB   |Closed        |0    | 
+-----+--------------+-----+

The RTS2 dome driver is located in the borat repo hardware/dome. It is also
pushed publicly to the svn of RTS2 and is located in the RTS2 root directory
in src/dome/. Any changes or fixes, please update our git repo and svn repo.
You will need a sourceforge account and Petr Kubanek would have needed to 
added you as a developer to the project. Simply email him.

One last note: If the dome is giving any problems in the BORAT repo,
under hardware/dome, a python script called ``dome.py`` which is a standalone
program for opening and closing the dome. 

In BORAT's mounted orientation, the north dome leaves are "B" and the south 
dome leaves are "A".

Telescope
=========

The Telescope is a Meade 16'' LX200GPS tube on a fork mount. Everything from
the protocol to the hardware seems to never work right. A lot of love, sweat,
and cursing has been invested into making the thing work halfway correctly.

In the last few months, at the time of writing, the LX200 RTS2 driver has been
copied and modified creating a LX200GPS driver. The LX200 protocol has changed
very little between LX200 and LX200GPS. The main difference between the two is
the LX200GPS driver puts the telescope to sleep within ``standby`` mode. 

Weather Sensor
==============

The weather sensor at BORAT is an all-in-one device. It does cloud detection, 
temperature, wind speed, humidity, and has a rain sensor. Cyanogen, the manufacturer, 
produces an open source driver that we have written an RTS2 interface for.

Cloud detection is the most important function the weather sensor has. Primarily becase
if there are clouds, there is a chance of rain, which would devastate our little setup.
Because of this risk, always check, via skycam, to see if the sky conditions match what RTS2 is 
telling you. If not, you will need to make modifications.

The cloud sensor determines the cloud condiition by taking the sky temperature and 
ambient temperature and subtracting them. From here the weather sensor can make an educated guess 
about the cloud conditions. This is the only part of the weather sensor that requires
calibration.

How does one calibrate the system? Simply using a tool in the cloud driver provided
by Cyanogen. The command line tool is called ``bwcs_test`` and is excellent for
working and testing the cloud sensor. The tool will also dump raw output of all the
sensors and their respected values, such as cloud condition.

Keep in mind that calibration of the weather sensor is done on the sensor ITSELF and NOT
in the RTS2 driver. This means that the weather sensor will actually make the call
if the sky is clear, cloudy, very cloudy. The RTS2 driver simply reads these values
and informs RTS2 about them. 

Here is a small excerpt of calibrations::

    ./bwcs_test
    
    :open /dev/boltwood
    // setT command will set the values for determining condidtion
    // setT <CloudyTemp(C)> <Very Cloudy Temp(C)> <Too Windy (MPH) <Wet Sensor> <Day Sensor>
    :setT -10 0 20 12 100 
    //call readLoop to implement the values
    :readLoop
    
    //press q to quite

Once done, the new values should start being used. Test and make sure. I had problems
with the sensor not taking the new values. These values are tested again with the SkyMinusAmbient
keyword when reading raw values from the sensor.

Security 
********

**THE BELOW IS NOT EVEN CLOSE TO BEING CURRENT, NEEDS UPDATING**

BORAT should have an alarm system to ward off vandals or thieves. An alarm package
to cover our requirements would be expensive. This led us to
assemble an alarm package with multiple components. The dome is surrounded
by a 6' privacy fence, thereby isolating our external environment. 
The alarm system then only needs to monitor within the fence. 
If activity is detected, a loud audible (Darth Vader) warning will be played and
bright flood lamps will be activated to ward off persons, scare away animals,
and provide lighting for our cameras. This setup is broken into three smaller systems.

The primary method for detecting unwanted activity is an array of sensors
placed around the dome. These are passive IR sensors, which sense sudden
changes in temperature via IR radiation, which a person naturally emits.
Small animals do not emit sufficient heat to trip the sensors. 
With a range of ~10 feet, it strikes a balance of detecting
positive matches such as a person near the dome while minimizing false detection
such as wildlife. The motion sensors are wired into series and placed around
the dome. An Arduino board is programmed to detect a voltage drop caused
by a sensor being triggered
and notify the server via USB (see Arduino section). 

Our video surveillance is a combination of CCTV cameras combined with IP cameras.
This serves two purposes. The CCTV cameras are external to the dome and for
security surveillance. Our IP cameras serve a more general role of monitoring 
telescope operations, yet they are also 
connected with the CCTV cameras to aid in
recording inside of the dome during an alarm. The CCTV cameras are driven by 
a capture card inside the server. 

An open source camera suite called ZoneMinder (www.zoneminder.com) 
is the heart of the surveillance system. The 
ZoneMinder suite includes primitive operation such as monitoring and
recording. A bonus to ZoneMinder is that it also detects activity on any of the  CCTV
live feeds. ZoneMinder is keyed to detect small amounts of pixel change in cameras to
determine activity. Any activity will put ZoneMinder in an ALARM state,
starting frame by frame recording and noting activity in logs. Since Nagios is 
already, implemented it will generate a notification via a custom Nagios 
plugin detailing which zone detected activity. 

During nighttime observation, the CCTV camera is independent of the IR sensors.
The procedure of detection follows

#. The Arduino will detect a voltage drop in the motion detectors.
#. The Arduino will send a detection message to the server.
#. The server will respond by notifying RTS2 of the alarm.
#. RTS2 will stop any observations, noting bad exposures in FITS header and close dome immediately.
#. The server will send a ON message to the WebSwitch powering the flood lamps.
#. ZoneMinder will detect the sudden extra light and begin monitoring CCTV feeds for any activity and begin recording.
#. Nagios will detect alarm by monitoring ZoneMinder logs thus generating a notification.

This system provides a complete monitoring solution during observation and idle
night periods. The alarm system will also be active during the day by a much
simpler method. When the dome is closed during daytime a magneto switch, placed
where two leafs meet, will be in a OPEN state. Any opening of a leaf by force 
or unauthorized opening of the dome will set the switch in a CLOSED state 
notifying the server via Android. Nagios notifications will be generated along 
with ZoneMinder recordings.

Zoneminder
==========

Zoneminder is a service running on BORAT. The purpose of Zoneminder is to
capture video for the security cameras and to record the video when an event
has trigger and alarm.

At time of writing, the current version of Zoneminder is 1.24. It is installed
via the package manager in Debian. Zoneminder will probably be fed updates
via the package manager so expect it to be updated over time.

BORAT is equipped with a PCIE 1x Bluecherry BC-H16480A 16 port video and audio
capture card. The settings for Zoneminder to work with the capture card is 
not quite straight forward. Keep in mind this capture card only supports
the following resolutions

Also keep in mind any changes, additions, or removals may cause ZM to complain
in regards to unexpected memory requests. Ignore these messages as kernel.shmall
and kernel.shmmax have been set appropriately. The 
simple solution is do a full reboot. 

Zoneminder Camera/Monitor Settings
----------------------------------

**General Tab**

* Name: name you wish to call it
* Source Type: Ffmpeg
* Function: Monitor
* Enabled: Checked
* Maximum FPS: 5
* Alarm FPS: 5
* Reference Image Blend: 7

**Source**

* Source: /dev/videoX (x is camera number)
* Source Colours: 24 bit colour
* Capture Width: 352 (see note)
* Calture Height: 240 (see note)

**Bluecherry BC-H16480A Specs**

NTSC

* 352x240 @ 480 FPS
* 704x480 @ 150 FPS

PAL

* 352x288 
* 704x576 

Motion Sensors
===============

After failing to find a motion sensor on the market that would fit our needs we
decided to implement our own. We bought several passive infrared (PIR) sensors
off sparkfun.com. These sensors were mounted facing roughly 60 degrees apart
to cover each side of the interior fence. 

The idea behind the PIR sensor is when first powered on they will calibrate to
their surroundings. Any changes in that will drop the voltage on (matt verify)
the signal line signaling something has been detected.

The PIR sensors are read using an Arduino board. The Arduino will talk
to the computer via USB-Serial on the values of the sensor. The Arduino data is
then read by the and Ardunio driver implemented in RTS2. In RTS2 the Arduino is
treated as a block device, such as the weather sensor, and if is triggered will
being shutting down the system and closing the dome protecting the contents from
possible intruders.

The Ardunio and PIR sensors are wired on a loop that runs through each box and each
sensor. They share a ground, power, and signal cable. These cables are conneted to
each sensor by a three pin stereo plug. This allows for each removal of the face-plates
with out having to solder and unsolder connections. 

When removing the faceplates be careful to disconnect the sensors. The PIR sensors
are mounted in the face-plate via silicon to create a water tight seal. They are NOT
secured in any other way and can easily be removed from the plate.

Flood Lights
============

The flood light setup is built to work with the Arduino in preventing possible
vandalism or theft. The flood lights are more of a deterrent and provide
lighting for security cameras to record or hope to capture events.

The flood lights are all wired together and brought into a dome via an electrical
cable. The electrical cable are plugged into the WebSwitch and are turned on through
the switch. A python script located in the repo under hardware/power\_web\_switch/boart\_power.py
will allow via CLI to turn on and off any of the electrical ports on the WebSwitch.
