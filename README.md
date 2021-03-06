<b>FAPI - Farm Assistant (API)

Firmware Specification  (v0.1)

Andrew McClure
AgSense Ltd.

Last Updated 29 Apr. 2017

Status: Draft

BACKGROUND</b>

The document describes the firmware used on AgSense's Farm Assistant's End Nodes.

These nodes can be used for rural/industrial processes; and communicate via an XBee SX 900Mhz network.

The Nodes are generic, low power, remote telemetry units that can be used for range of I/O functions including:

- Acctuator
- Analog voltge sensors
- 1-Wire sensors with software support SHTx1 Temperature/Humidity sensors 
- Digital inputs, including flow sensor
- Exended TTL I/O including software support for TTL Camera, Electric Fence and GPS
- Signal strength/Battery voltage functions

<b>MAIN PROCESSOR and HARDWARE FEATURES</b>

The firmware runs on the ATMEL 1284p processor:
http://www.atmel.com/images/doc8059.pdf

4MBit WinBond SPI Flash provides suppot for OTA firmware updates

Power management is via our integrated 12v charge circuits and can be from either AC/DC transformer or solar panel.

<b>I/O Features</b>

- Console output via Digital Ports 14/15 (RX/TX)
- UART0 connected to XBee radios / OR FTDI programming
- UART1 multiplexed 4 ways to provide support for multiple devices
- 6pin Headers exposed for  Serial programmers
- Bootloader supports boot from Flash memory
- 4x Analog input
- Digital Interrupt/Flow Meter
- SPI
- 1-Wire

<b>WIRELESS COMMUNICATIONS and DIGIMESH</b>

At the core of our firmware is a network protocol for exchanging data seamlessly, reliably and securely with the gateway.

Our network stack features the following:

- End node interface is sub classed from Arduino Streams library providing a set of core routines for beginning; reading; writing; checking availability (and ready states), and flushing the communications.
- On top of these routines are additional methods for specifying BAUD rates, and querying the XBee Modem's state.
- Subclassed further are wrappers that provide a framework for handling packetised data.  
- Packets can be of arbritary length, and are defined by a simple begin/end byte.  
- A constrained (buffered) wrapper is also provided that queues packets of a finite size.
- beginPacked and endPacket routines are provided, and also implied when using simple "print" statements from a wrapper class.
- The underlying network protocol is an implementation of Selective Repeat ARQ https://en.wikipedia.org/wiki/Selective_Repeat_ARQ with 7-frame windows and a 6 byte header.
- On the gateway side, a dedicated nodejs process listens to all network traffic and exposes each End Node as a UNIX socket. If data is to be sent in packets, this is handled at the network layer with a flag set via the End Node and Communicated in the network header. (All encapsulation is therefore seamless to the gateway client.)

<b>APPICATION LAYER (SenML)</b>

Services that run in our firmware expose an Application Layer protocol based on SenML: https://datatracker.ietf.org/doc/draft-ietf-core-senml/

To conform with a Serial based framework, this protocol has been extended to incorporate SenML/JSON oriented requests (as oppossed to pureley CoAP based)

For example, to activate an acctuator we send:

<pre>
[ { bn: 'urn:dev:xbee:0013a20040af7346' },
  { n: 'D0', vb: true } ]
</pre>

To query the state of an accuator we use:

<pre>
[ { bn: 'urn:dev:xbee:0013a20040af7346' },
  { n: 'D0' } ]    (or leave vb:undefined)
</pre>

The n: flag correlates to the Service name.  A full API is being developed.  Here are some additional examples:

- Voltage sensor from End Node
<pre>
[ { bn: 'urn:dev:xbee:0013a20040af7346' },
  { n: 'A0', v: 10.1 } ]
</pre>

- Positional data in header
<pre>
[ { bn: 'urn:dev:xbee:0013a20040af7346', pos:[&lt;lon&gt;,&lt;lat&gt;,&lt;alt&gt;], bt:&lt;ms since midnight&gt; }, {...} ],
</pre>

<i>(Note: as our End Nodes do not incorporate a RTC, time is synced via the gateway)</i>

- Camera image
<pre>
[ { bn: 'urn:dev:xbee:0013a20040af7346'},
  {n:'CAM', vd:&lt;Buffer 0x....&gt;}]
</pre>

(note, all SenML data is encoded/decoded via Msgpack http://msgpack.org, providing both type and size metadata)

- Firmware information

Request:
<pre>
[ { bn: 'urn:dev:xbee:0013A20000000000'},{ n:VERS }]
</pre>

Response:
<pre>
[ { bn: 'urn:dev:xbee:0013A20000000000', bt: 28890433 },
  { n: 'config', vs: '0x10B' },  ( See CONFIG MANAGEMENT below )
  { n: 'repo', vs: 'git@gitlab.comxxxxxxx.git' },
  { n: 'branch', vs: 'OTASWork' },
  { n: 'revision', vs: '145c72141082ef85284babd72bcd575fb8533100' },
  { n: 'EEPROM', vs: "1.0" },
  { n: 'XARQ', vs: "1.0" },
  { n: 'RESET', vs: "1.0" },
  { n: 'VERS', vs: "1.0" } ]
</pre>

Signal Strength
<pre>
[ { bn: 'urn:dev:xbee:0013a20040af7346'},
  {n:'dB', v:15}]
  </pre>


<b>OTA - Over the Air Programming</b>

Our bootloader supports OTA progamming by checking for a flash image in the 4MBit SPI Flash.

We provide two modes for updating OTA. Fast and Slow.

- FAST: This temporarily disables our Application Layer providing a fast dedicated byte stream upload  (4-10kbps).
- SLOW: This encapsulates 250byte segments within SenML with a simple API for starting, sending, and resetting the device.

<i>Note: Slow is beneficial for performing background updates without affecting operations either on the node itself, or on other network nodes served by the same gateway.</i>


<b>CONFIGURATION MANAGEMENT</b>

The FAPI Software Library combines a set of core services for managing the available services on the end Node.

These services are either stand alone, or utilise a set of slightly adapted existing, Arduino conformant, libraries.

All services provide a core loop() and setup() method.  They also provide a name() and version() method.

Some services inherit from a Sleep service that provides optional PUB/SUB notifications with low power sleep.

(PUB/SUB settings are controlled via SenML using the ps:x tag where 0 means no sleep; and x provides the sleep window in milliseconds)

To enable a service, a configuration DEFINE is passed to the build process; used to include/exclude module. A build script runs the Makefile and checks for changes in parameters. For example, to build a version of the firmware with support for a TANK Module, OTAS updates, RESET,  EEPROM and SLEEP we can use:

./build XCONFIG=0x111F XDEBUG=0x1 FAST=0x0.

Each Service is responsible for checking dependendencies via a static_assert method.

Example services:

- FAPI_EEPROM_ 0x01
- FAPI_XARQ_   0x02 
- FAPI_SLEEP_  0x04
- FAPI_RSET_   0x08
- FAPI_OTAS_   0x10
- FAPI_BATT_   0x40
- FAPI_SIGNAL_ 0x80
- FAPI_REVS_   0x100
- FAPI_GPS_    0x200
- FAPI_CAM_    0x400
- FAPI_ACC_    0x8000
- FAPI_TANK_   0x1000
- FAPI_SHT1x_  0x200
