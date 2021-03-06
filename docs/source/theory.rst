.. index:: !goals

.. _theory:

==============================================
Project Goals
==============================================

Main Goals
-------------
The main goal of this project is to present
the **Arduino as an I/O controller** using a 
command-response communications language
over the USB port.  The code should run on
any Arduino with a USB port.  The code
should provide access to the basic I/O 
capabilities of the Arduino.

conserve SRAM
+++++++++++++++++++

To fit the code into the smaller SRAM space of 
Arduino models such as the Uno R3 (a 2k space), 
all strings were coded as ``char *`` rather
than use ``String`` [#]_ objects.

.. [#] http://arduino.cc/en/Reference/StringObject

short commands
+++++++++++++++

One objective of **cmd_response**
was to make basic read and write commands
very short, to simplify typing at some interactive
terminal.  
(For a similar project using SCPI [#SCPI]_ commands, see :ref:`epicsduino`.)
The USB commands have this form::

  //  USB command interface of form: baseCmd [arg1 [arg2]]
  //
  // char* baseCmd (lower case)
  // long arg1, arg2 (no octal or hex interpreted)

additional rules for the interface
++++++++++++++++++++++++++++++++++++++++

Additionally, these rules were used to 
build the interface:

* Commands either **get** (start with ``?``) 
  or **set** (start with ``!``) a value.
* All :ref:`commands <commands>` received should return a response.
* No automatic command echo is provided.
* All **set** commands return "Ok" or an error message.
* All **get** commands return a value or an error message.
* All :ref:`errors` start with the text: ``ERROR_``
* Most error messages will return the command text as a diagnostic.
* The device provides a useful message on :ref:`startup <is.our.sketch.running>`.
* A software :ref:`identifier <get_id>` can be queried.
* A software :ref:`version <get_version>` can be queried.
* The version should be a string rather than an integer.


.. _basics:

Basic Operations
-----------------

The basic capabilities common to all Arduino boards
consist of:

* a number of digital (binary) input/output (configurable) channels
* some of the digital channels have pulse-width modulation output capability
* a number of analog input channels (10-bit analog-to-digital converter)

The commands which support these basic operations are:
:ref:`num_ai`, :ref:`num_bi`, :ref:`get_ai`, 
:ref:`get_bi`, :ref:`set_pin`, :ref:`set_bo`,
and :ref:`set_pwm`.


.. index:: !signal averaging, averaging

.. _averaging:

Signal Averaging
----------------

It takes a few dozen [#]_ microseconds to read each
analog input channel, much faster than typical 
communications over USB to request and retrieve 
a value.  Averaging the analog values over a 
fixed period makes sense.  

.. [#] The actual time to convert the ADC signal
   is dependent on several factors, including
   the signal level itself.  For more information,
   see: http://arduino.cc/en/Reference/AnalogRead
   
   On the Mega2560, the rate with no watched channels
   was reported as 25841/s.  Watching just one 
   channel, the reported rate dropped to 6567/s.  
   For two channels, the reported rate was 3781/s.

The signal averages are available from the
command interface with :ref:`ai_mean`, 
in addition to the 
instantaneous values available with the 
:ref:`get_ai` command.  However, since it takes
valuable time to read these analog signals, 
only the desired channels should be averaged, 
to optimize for speed.  

.. note:: A channel must be 
   configured (with :ref:`watch_ai`) 
   before calling :ref:`ai_mean`.

.. index:: multiplier

The average for each channel is obtained by
accumulating measurements of the analog input,
:math:`a_i` until the update period has elapsed,

:math:`(k/n) \sum_i^n a_i`.

The total number of values accumulated during
each period, :math:`n`, is combined with the
period length, :math:`t` to compute the update
rate (returned by :ref:`get_rate`),

:math:`n / t`.

The average value has higher precision than the
instantaneous value.  Exactly how much higher
is empirical.  Some external control systems
expect *raw* (integer) values from I/O controllers.
With such raw numbers, the external control system
will apply automatically pre-configured scaling 
factors to place the number into engineering units, 
such as *VDC*.

.. index:: !multiplier

Here, a positive multiplier, :math:`k`, is used
to scale the average value so that its limit 
of precision can be expressed as a *long* integer.
To request the value of :math:`k`, use :ref:`get_k`.
To request the value of :math:`k`, use :ref:`set_k`.
Limits of :math:`k` are found by requesting
:ref:`get_k_min` and :ref:`get_k_max`.

.. index:: watched channels, averages

All **watched channels** are accumulated during each
averaging period.  The averages (for just the watched channels)
and update rate are recomputed *only* at the end of each period.

How well does Signal Averaging work?
++++++++++++++++++++++++++++++++++++++

In the next figure [#]_ are plotted (vs. time) the instantaneous 
signal (green), sampled regularly at twice per second and the
time-averaged signal (red) as described.  Here, the averaging period
is 0.5 seconds.  The sensor is a light-dependent resistor 
(photoresistor) sensing a slowly decreasing signal (the 
afternoon sun light intensity, also known as *ambient 
solar intensity*).

..	figure:: signal-averaging.png
	:alt: fig.signal-averaging
	:width: 70%
	:align: center

	Ambient solar intensity vs. time.  
	Green: instantaneous signal sampled at 2 Hz.
	Red: time-average signal with 0.5 second averaging time.
	(:download:`signal-averaging.png`)

Observe how the time-averaged signal does not vary smoothly
with the slowly decreasing light level.  Instead, it follows 
more closely a smooth step-wise descent.  The steps correlate
with the discrete values of the 10-bit ADC channel (see the 
green curve).

About 15:43, the averaging period was changed to 1.0 seconds.
The chart is shown again, with both older and newer data.
Note the effect of passing clouds on the signal at 15:50 and 16:02.

..	figure:: signal-averaging-longer.png
	:alt: fig.signal-averaging-longer
	:width: 70%
	:align: center

	Ambient solar intensity vs. time.  
	Green: instantaneous signal sampled at 2 Hz.
	Red: time-average signal with 0.5 second averaging time
	(1.0 second averaging after 15:43).
	(:download:`signal-averaging-longer.png`)

We conclude that the signal averaging method used here smooths
the signal quite a bit but favors values close to the native 
precision of the ADC.  
The general trend of decreasing ambient solar intensity with time
is apparent but we have gained an artefact that suggests the
intensity is slowly oscillating.  This is not a natural phenomenon.
For all our effort, we improved 
our immunity to the noise and variation of single ADC measurements,
with only a modest increase in the precision of our measurement.

.. [#] These charts are created using the EPICS system 
   we describe later.  See: :ref:`cssboy`.

Startup
-----------------

On startup, the device will output a message such
as this to the USB port::

  cmd_response started: 7113

This message (from an Arduino Mega2560) signifies 
the device is *running* and the **cmd_response**
interface code is ready to receive a 
command and return a response.
The number ``7113`` is the amount of free 
SRAM available on startup.  On the Uno R3, the number
is ca. 1700, meaning there is still plenty of
SRAM available.

If this startup message does not appear.
the device is not ready for communications
and something is wrong.  The logical list
to check starts with the values in the 
Communication Parameters (given
in :ref:`comm_parms`).  If the problem persists,
check the external host's USB port,
the FTDI driver on the external host, 
read/write permissions to the USB port, and
the USB cable.  If the problem still persists,
question that the Arduino is running 
the **cmd_response** code.


Configuration Check
--------------------

It can be useful for an external control system 
to verify which controller is communicating.
Two commands are provided to identify
this software (:ref:`get_id`) and 
version (:ref:`get_version`).


.. index:: epicsduino

.. _epicsduino:

epicsduino
--------------

.. epicsduino is behind the APS firewall.
   It could be posted publicly for better access.

The **epicsduino** project of Keenan Lang
at the Advanced Photon Source [#]_
is *very* similar to this project as it was developed in parallel to 
this project.
In the **epicsduino** project, an SCPI command interface [#SCPI]_ is 
constructed using String objects.  This interface consumes significant 
SRAM and thus limits the features which can be implemented in the interface 
language and also limits the Arduino models on which the sketch can be used.
Also, the SCPI commands are longer, and take more time to parse.

.. [#] **epicsduino**: https://subversion.xray.aps.anl.gov/bcdaioc/projects/epicsduino
.. [#SCPI] SCPI: http://www.ivifoundation.org/scpi/default.aspx
