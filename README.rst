===========================
Roland FP-30 Remote Control
===========================

Remote contol the Roland FP-30 (dashboard) settings using WebMIDI.

----

The Roland FP-30 has an accompanying App that allows to change the settings via
a mobile device. Unfortunately the App is extremely flaky and its communication
is undocumented.

  .. note:: The following has only been tested with the Roland FP-30.
            Similar instruments (like the FP-10) might work.


tl;dr
=====

The App uses undocumented MIDI SysEx DT1 and RQ1 to communicate with the
instruments settings using 4 byte model ID ``00 00 00 28`` and 4 Byte addresses.

The instrument itself also exposes a GS module. This is a separate module,
modifying its parameters only affects MIDI data received via MIDI, not data
received from the instrument itself (Local Control). By disabling Local Control
and looping back the MIDI events this can be used to extend the capabilities of
the instrument. This has been done by other projects.


Communication
=============

Using the App
-------------

The App connects via BLE or USB and supports MIDI over those connections.
Unfortunately the MIDI manual doesn't document anything related to remote
controlling the instrument.

Connecting the mobile device with the App running to a PC makes it send data
via MIDI.

Thus in a (larger) setup using both the App and a PC to interface with MIDI is
possible if the App MIDI is routed to the instrument (and back.)


Snooping MIDI traffic
---------------------

Using a PC to route the MIDI messages allows to dump the data transmitted.
Various MIDI tools allow creating such a setup (f.e. ``aseqdump`` in
combination with ``aconnect`` to set up the routing.)

There's some initial output when connecting. After that changing values on the
instrument causes output like::

  24:0  SysEx  F0 41 10 00 00 00 28 12 01 00 02 13 41 29 F7
  24:0  SysEx  F0 41 10 00 00 00 28 12 01 00 02 13 40 2A F7
  24:0  SysEx  F0 41 10 00 00 00 28 12 01 00 02 13 3F 2B F7
  24:0  SysEx  F0 41 10 00 00 00 28 12 01 00 02 13 3E 2C F7

Changing the same value from the App gives pretty much the same::

  20:0  SysEx  F0 41 10 00 00 00 28 12 01 00 02 13 3F 2B F7


Understanding the data
----------------------

There are various resources on SysEx on Roland instruments. `This page
<http://chromakinetics.com/handsonic/rolSysEx.htm>`_ explains DT1 and RQ1.
Looking at our communication the ``12`` stands out, so this could be a DT1, but
the size of the model ID doesn't match. The `FP-30 MIDI manual
<https://static.roland.com/assets/media/pdf/FP-30_MIDI_Imple_e01_W.pdf>`_
documents DT1 with one byte Model ID and 3 bytes for address.

Roland uses a common scheme for SysEx messages, though there are small
differences between instruments (size of model ID, number of address bytes).
Those DT1 and RQ1 are not described in the FP-30 MIDI manual.

Checking the MIDI manual for other devices shows the `TD-50
<https://static.roland.com/assets/media/pdf/TD-50_MIDI_Imple_e03_W.pdf>`_
using 4 bytes for model ID, so this is an existing variant. Using a model ID of
``00 00 00 28`` makes this fit. Calculating the checksum also gives the same
value as in the log.

Decoding the 1st message from above like that yields

- Model ID: ``00 00 00 28``
- Command ID: ``12`` (DT1)
- Address: ``01 00 02 13``
- Value: ``41`` (65 decimal)

In the example the volume has been decreased. The value sent is 1 byte, which
can (given that we're actually dealing with 7bit values here) hold the full
range 0 ~ 100.

We can now watch at the data sent by the App, and try to send other values
based on the above.


App startup handshake
---------------------

Trying to send a DT1 constructed like this makes the instrument react if the
App has already made a connection, otherwise it doesn't give the expected
result. Furthermore, the instrument doesn't send data unless the App is
connected. Let's look at what's happening during App startup.

::

  20:0  SysEx  F0 41 10 00 00 00 28 11 01 00 07 00 00 00 00 08 70 F7
  24:0  SysEx  F0 41 10 00 00 00 28 12 01 00 07 00 00 00 00 00 00 00
               00 00 78 F7

The App sends a RQ1 for address 01000700 for 8 Bytes. The instrument returns
with a DT1 and 8 Bytes of data, which are all ``00`` here. So the App queries
the instrument for data (read request), and the instrument transfers the
requested data back to the App.

This happens a couple of times with different addresses.

::

  20:0  SysEx  F0 7E 10 06 01 F7
  24:0  SysEx  F0 7E 10 06 02 41 42 00 00 20 00 01 00 00 F7

Now the App sends the "Identity Request Message". This is documented in the
FP-30 MIDI manual, and the instrument sends the response as documented. So
there is a check for the actual type of the instrument connected. This makes
sense since the App supports other instruments that differ in features (like
tone list.)

::

  20:0  SysEx  F0 41 10 00 00 00 28 12 01 00 03 00 00 00 7C F7
  20:0  SysEx  F0 41 10 00 00 00 28 12 01 00 03 06 01 75 F7
  20:0  SysEx  F0 41 10 00 00 00 28 12 01 00 03 00 00 01 7B F7

Here things get interesting. Now the App sends DT1, i.e. it transfers
("writes") to the instrument. That's the same thing as when changing the
volume, but this is during startup.

Manually trying these addresses and the values sent by the App reveals that
those two are the ones needed to enable the App features:

- ``01000306``  set to ``01`` enables the instrument to accept DT1 from the
  App. Set to ``00`` and the instrument will ignore DT1 for settings.
- ``01000300``  set to ``00 01`` enables the instrument to send DT1 SysReqs on
  changes made via the dashboard. Set to ``00 00`` to disable.


Parameter Address Map
---------------------

With this knowledge getting the Parameter Address Map used is trivial by simply
observing the data sent. Some addresses are read-write, some read-only and some
are read-only and require writing to a different write-only address for
changing.  Some values are signed and thus use an offset (for 1 Byte value the
offset is 0x40).

======== ====== ==== =========================================================
Adress   Bytes  Type Description
======== ====== ==== =========================================================
01000000    32  r-   (appears to be some kind of identification)
01000101     1  r-   transpose (-6 ~ 5)
01000103     1  rw   recorder play state (stop = 0, start = 1)
01000105     2  r-   playback current bar?
01000108     2  r-   metronome speed
0100010f     1  r-   metronome status
01000110     1  r-   speaker on (0: on, 1: off)
01000200     1  rw   | keyboad mode
                     | (0: single, 1: split, 2: dual, 3: twin)
01000201     1  rw   split point: MIDI note B1 ~ B6
01000202     1  rw   split: left shift (-2 ~ 2)
01000203     1  rw   split: balance (-9 ~ 9)
01000204     1  rw   dual: tone 2 shift (-2 ~ 2)
01000205     1  rw   dual: balance (-9 ~ 9)
01000206     1  rw   twin: mode (0: pair, 1: individual)
01000207     3  rw   | tones / tone 1
                     | byte 0: piano / e-piano / other (0 / 1 / 2),
                     | byte 1, 2: index in list.
0100020a     3  rw   tones left (split mode)
0100020d     3  rw   tone 2 (dual mode)
01000213     1  rw   master volume (0 ~ 100)
01000218     2  rw   master tuning (+- 0.1Hz, 0200 = 440Hz)
0100021a     1  rw   ambience (0 ~ 4)
0100021c     1  rw   brilliance (-1 ~ 1)
0100021d     1  rw   | key touch
                     | (0: fix, 1: superlight, 2: light, 3: medium, 4: heavy, 5: superheavy)
0100021f     1  rw   | beats type
                     | (0: 0/4, 1: 2/4, 2: 3/4, 3: 4/4, 4: 5/4, 5: 6/4)
01000300     2  -w   | Enable sending dashboard changes to MIDI
                     | (0x00, 0x01: on, 0x00, 0x00: off)
01000306     1  -w   | Enable Remote control
                     | (0x00: off, 0x01: on)
01000307     1  -w   transpose (-6 ~ 5)
01000309     2  -w   metronome tempo (10 ~ 500)
01000509     1  -w   metronome toggle (value always 0)
01000700     8  r-   (unknown. Always zeros.)
01000800     1  r-   (unknown. Seems to be some kind of identification.)
======== ====== ==== =========================================================


Instrument Bugs
===============

There is a bug in the instrument software: when changing the transpose value to
0 (i.e. no transpose) no message is sent to the App. This can be observed with
the official App too. Reading the transpose location gives the correct result,
and changing transpose to anything else makes the instrument send a DT1 as
expected.


Related Projects
================

* https://github.com/evanraalte/RolandPiano
* https://github.com/JJulio/FP30playground
* https://github.com/arachsys/webmidi

