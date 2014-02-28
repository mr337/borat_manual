RTS2 Implementation
*******************

Primer
======

The software that is the puppet master for all the devices is RTS2. RTS2 was
written by Peter Kubanek.

A slight introduction into RTS2 is necessary. RTS2 handles all the high and low
level working of an autonomous telescope. The high functions include scheduling,
observations, and more interesting stuff such as automatic focusing. It also plays
the low level role by talking with the hardware directly or via a prebuilt driver.

While RTS2 may seem complex, it is actually pretty intuitive. Though it is a large
system, do not be afraid to take a peak at the code. The sourceforge website
also contains a doxogen documentation that will help to understand the source code.

It is important to make note that BORAT is running on svn revision 11782.
Hopefully a stable branch will be cut soon, so far this revision seems stable.

Big Picture
===========

RTS2 works by having several, actually many, programs together, more than 
can talk to each other. Each program/service talks through a central controller known as
centrald.

Centrald is the first program to be ran when the RTS2 service is started.
Centrald is the hub for communication. It takes commands sent from other
programs, like devices, and passses them along.

The programs that communicate with RTS2 can be device drivers or high function
programs like scheduler and target selection. Each of these programs have to
inherit an interface provided from rts2. These interfaces can be telescope,
sensor, camera, dome, focuser, and more. BORAT focuses mainly on the essential
devices just listed.

In short, that is the big picture for RTS2. As with many things the devil is in
the details.


Devices
=======
All the BORAT drivers, except the LX200, filter wheel, and CCD, were
written at MSU. We tried to comment the shit out of them, so someone else can
use/improve/recycle them if needed.

On startup of the RTS2 service, certain drivers are called. Which drivers and
options are located at ``/etc/rts2/devices.ini``.

Dome
-----

The dome driver is pretty self explanatory. Shoot some code over the serial
connection and
wait for replies. Look in the chapter ``Hardware`` for details on the 
protocol. The driver is called ``rts2-dome-ahe``.

Weather Sensor
--------------

The weather sensor is built on the open source driver provided by 
the manufacturer,
Cyanogen. The RTS2 weather sensor driver connects the driver to RTS2. Pretty
simple stuff. More details on calibration in ``Hardware``. The weather sensor
driver is called ``rts2-sensord-boltwood``.

Focuser
-------

This is by far the most pain-in-the-ass driver to date. Quite frankly I hate
it with the passion of 1,000 burning suns. It works, that is all that matters.
The focus driver is called ``rts2-focusd-ez4axis``.
