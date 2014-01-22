Miscellaneous
*************

Quick Scripts
============

When a target is observed a script dictates how to observe a target. By using
the ``rts2-scriptexec`` command abritrary observation scripts can be executed.

Take one 3s exposure::

    rts2-scriptexec -d C0 -s 'for 1 { E 3 }' #take 1, 3 second exposure via xmlrpc

Move focus position -1800, in 10 iterations take a 7s exposure and increment focus 
position -75, also while doing 2x2 binning::

    rts2-scriptexec -d C0 -s 'binning=1 F0.FOC_TAR=-1800 for 10 { E 7 F0.FOC_TAR-=75 }' #do quick focus run
