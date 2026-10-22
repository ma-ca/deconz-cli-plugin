Introduction
============

The deconz-cli-plugin provides an API to access devices with ZigBee Home Automation (HA) and ZigBee Light Link (ZLL). It can be used as command line interface (cli) to send and receive raw ZigBee commands with deCONZ software.

As hardware the [RaspBee](http://www.dresden-elektronik.de/funktechnik/solutions/wireless-light-control/raspbee?L=1) ZigBee Shield for Raspberry Pi is used to directly communicate with the ZigBee devices.

The deconz-cli-plugin requires the [deCONZ software](http://www.dresden-elektronik.de/funktechnik/products/software/pc/deconz?L=1).

The deconz-cli-plugin opens a socket on port TCP 5008 and listens for incoming connections. The plugin allows to send commands and receive responses to and from ZigBee devices like Philips Hue lights. The plugin translates the incoming commands to APS-Data and passes the data to the RaspBee firmware.

## Preparation and configuration steps

In the standard Raspbian Linux distribution the serial interface /dev/ttyAMA0 is set up as a serial console, i.e. for boot message output. Since the RaspBee ZigBee firmware uses the same interface to communicate with the control software, the following changes must be done to “free” the UART.

In the file /boot/cmdline.txt:

If present remove the text `console=serial0,115200` or `console=ttyAMA0,115200`

Raspbian wheezy: comment out the following line in `/etc/inittab`

     #Spawn a getty on Raspberry Pi serial line
    T0:23:respawn:/sbin/getty -L ttyAMA0 115200 vt100

Raspian jessie:

    sudo systemctl disable serial-getty@ttyAMA0.service

Raspberry Pi3 jessie:
in the file /boot/config.txt add the line

    enable_uart=1

Reboot

## Installation and compilation steps

The deCONZ development package is not needed anymore. The deCONZ library headers are fetched
automatically from [deconz-lib](https://github.com/dresden-elektronik/deconz-lib) during the CMake configure step.

##### Compile the plugin

1. Compile the plugin

    ```
    git clone https://github.com/ma-ca/deconz-cli-plugin.git
    cd deconz-cli-plugin
    cmake -S . -B build
    cmake --build build
    ```

   Requires CMake >= 3.11, Qt5 or Qt6 (Core, Network, SerialPort) development packages, OpenSSL development
   package, git and a C/C++ compiler. Use `-DDECONZ_LIB_GIT_TAG=<tag>` to select a specific deconz-lib version.

2. Copy the plugin to the deConz plugins folder

    ```
    sudo cp build/libdeconz_cli_plugin.so /usr/share/deCONZ/plugins
    ```

Start deConz (parameter --dbg-info=1 only needed for debug messages). It might be necessary to start with UI when starting and forming the ZigBee network for the first time. Thereafter the UI is not needed anymore.

with UI (requires X-server)

    sudo systemctl enable deconz-gui
    sudo systemctl start deconz

-or-

    deCONZ --http-port=80

    deCONZ --http-port=80 --dbg-info=1 --dbg-aps=2 --dbg-zcl=1


without UI

    sudo systemctl enable deconz
    sudo systemctl start deconz

-or-

    deCONZ -platform minimal --http-port=80

Hardware requirements
---------------------

* Raspberry Pi
* [RaspBee](http://www.dresden-elektronik.de/funktechnik/solutions/wireless-light-control/raspbee?L=1) ZigBee Shield for Raspberry Pi
* or a [ConBee](https://www.dresden-elektronik.de/funktechnik/solutions/wireless-light-control/conbee/?L=1) USB dongle

Usage
=====

The plugin is started when the deConz software is started. The deConz software must be running before starting pilight.


Run netcat to connect to port 5008. Type `help` to show usage.

    nc localhost 5008


Command | Description
------- | -----------
`r <shortaddr> <ep> <cluster> <attrid>` | read attributes
`b <shortaddr> <ep>` | read basic attributes
`b <shortaddr> <ep> <cluster>` | read basic attributes on cluster
`m <profile> <cluster>` | send match descriptor request (discover cluster)
`p <shortaddr>` | permit Joining on device (coordinator = 0)
`zclattr <shortaddr> <ep> <cluster> <command>` | send ZCL attribute request
`zclcmd <shortaddr> <ep> <cluster> <command>`  | send ZCL command request
`zclcmdgrp <groupaddr> <ep> <cluster> <command>` | send ZCL command request to group
`zdpcmd <shortaddr> <cluster> <command>` | send ZDP command request
`zclattrmanu <shortaddr> <ep> <cluster> <manufacturer id> <command>` | send ZCL attribute request manufacturer specific
`zclcmdmanu <shortaddr> <ep> <cluster> <manufacturer id> <command>`  | send ZCL command request manufacturer specific
`sendtime <shortaddr> <ep>` | send ZCL time attributes to Time_Cluster 0x000A


Response | Description
--------| -----------
<-LQI 0x84182600xxxxxxxx   06 0 1 0x001FEE00xxxxxxxx 0x4157 1 1 2 01 02 56 | LQI neighbor reponse
<-ZCL attribute report 0x001FEE00xxxxxxxx 0x0006 1 00 00 10 00 | ZCL attribute report (from cluster 0x0006)
<-ZCL serverToClient 0x00124B00xxxxxxxx 1 for cluster 0x0500 10 00 00 00 00 00 | ZCL attribute (with extaddr)
<-ZCL serverToClient 0x3B58 1 for cluster 0x0500 10 00 00 00 00 00 | ZCL attribute (with shortaddr)
<-APS attr 0x001FEE00xxxxxxxx 5 0x0702 0x0000 0x25 0E DA 76 57 00 00 00 04 00 2A 00 00 00 | APS data

    <-LQI [source addr] [neighborTableEntries] [start] [count] [extaddr] [shortaddr] [device type] [rxOnWhenIdle] [relationship] [permitJoin] [depth] [lqi]
    <-APS attr [source extaddr] [endpoint] [cluster] [attributeid] [typeid] [attr value] [...more attributes]



LQI | Description
--- | -----------
source addr | LQI neighbor table from <source addr>
neighborTableEntries | total table entries
start | current table entry start index
count | number of table entries in this frame
extaddr | table entry extended address
shortaddr | table entry short address
device type | 0 = coordinator, 1 = router, 2 = end device, 3 = unknown
rxOnWhenIdle | receiver on when idle 0 or 1
relationship | relationship 0 = neighbor is the parent, 1 = child, 2 = sibling, 3 = None of the above, 4 = previous child
permitJoin | permit Join 0 = neighbor is not accepting join requests, 1 = accepting join requests
depth | The tree depth of the neighbor device
lqi | The estimated link quality (range 0x00 - 0xff)


Supported ZigBee devices
========================

Successfully tested with follwing ZigBee devices.

Device | Vendor
------ | ------
Light Philps Hue White | Philips, LWB006
Light | OSRAM, Classic B40 TW - LIGHTIFY
Smart Plug | OSRAM, Plug 01
Movement Sensor | Bitron Home, 902010/22
Motion Sensor | Philips Hue motion sensor
Smoke Detektor with siren | Bitron Home, 902010/24
Smart Plug with Metering | Bitron Home, 902010/25
Thermostat | Bitron Home, 902010/32
Switch | ubisys, S2 (5502)
Button | Philips Hue dimmer switch
Button | [Xiaomi Smart Wireless Switch](https://www.banggood.com/Original-Xiaomi-Smart-Wireless-Switch-p-1045081.html)
Temperature | Xiaomi Temperature and Humidity Smart Sensor

Reference Manuals
-----------------
[Bitron](http://www.bitronvideo.eu/index.php/service/bitron-home/downloads/)

[Ubisys](http://www.ubisys.de/en/smarthome/products-s2.html)

Reset ZigBee device to factory default
--------------------------------------

[Philips Hue Light](http://www2.meethue.com/de-de/support/faq-detail/?productCategoryId=131288)
Can I factory reset a Hue light with the Hue dimmer switch?
Yes you can, press and hold the ON and OFF button simultaneously until the LED light on the dimmer switch turns green (Note: the lamp is blinking during this process. Hue Beyond and Hue Phoenix are excluded).

Osram Lightify Light turn the light on and off five times and wait 5 seconds inbetween.


[Bitron Video](http://www.bitronvideo.eu/index.php/service/bitron-home/zb-registrierung/)
Press and hold the button for 10 seconds.


Troubleshooting
---------------

After a while (rarely and randomly) a device keeps responding with an error status 0xD0 when reading attributes or sending other commands. Sometimes this just gets resolved without doing anything but waiting. However, sometimes it turns out that the device shortaddress has changed for some unknown reason. Oberserving the LQI neibortable entries shows the new shortaddress.

<pre>
zclattr <b>0x4157</b> 5 0x0702 0000000004 --> send OK
<-APS-DATA.confirm FAILED status 0xD0, id = 0x25, srcEp = 0x01, dstcEp = 0x05, dstAddr = <b>0x4157</b>
<-LQI 0x00178801xxxxxxxx   07 0 1 0x001FEE00xxxxxxxx <b>0xA855</b> 1 1 3 01 01 F7
zclattr <b>0xA855</b> 5 0x0702 0000000004 --> send OK
</pre>

Restart RaspBee
---------------


    sudo pkill deCONZ; sudo GCFFlasher -r; sudo systemctl start deconz

Example Commands
----------------

Send match descriptor to Profile 0x0104 (ZHA) and Cluster 0x0102 (Window Covering Cluster)

    m 0x0104 0x0102

    --> send OK
    <-ZDP match 0x001FEE000000253B 0xD1F3   1 0x0102 0x0104
    b 0xD1F3 1 0x0102
     --> send OK
    <-ZCL attr 0xD1F3 1 0x0102 0x0004 ubisys
    <-ZCL attr 0xD1F3 1 0x0102 0x0005 J1 (5502)
    <-ZCL attr 0xD1F3 1 0x0102 0x0006 20170712-DE-FB0
    <-ZCL attr 0xD1F3 1 0x0102 0x4000 unknown

send match descriptor to Profile 0xC05E (ZLL) and Cluster 0x0006 (OnOff Cluster)

    m 0xC05E 0x0006

send match descriptor request to Profile 0x0104 (ZHA) and Cluster 0x0006 (OnOff Cluster)

    m 0x0104 0x0006

get group membership

    zclcmd 0xC05E 11 0x0004 0200

switch off / on (cluster `0x0006`, endpoint 2, commands off/on/toggle `00`/`01`/`02`)

    zclcmd 0xE471 2 0x0006 00  # turn off
    zclcmd 0xE471 2 0x0006 01  # turn on
    zclcmd 0xE471 2 0x0006 02  # toggle

read attribute OnOff (id `0x0000`) on cluster `0x0006` endpoint 2, response `01` = bool true

    zclattr 0xE471 2 0x0006 000000
     --> send OK
    <-APS attr 0xE471 2 0x0006 0x0000 0x10 01

read S2 ubisys config

    zclattr 0x6C25 232 0xFC00 000100

    --> send OK
    <-APS attr 0x001FEE0000001739 232 0xFC00 0x0001 0x48 41 04 00 06 00 0D 03 06 00 02 06 01 0D 04 06 00 02 06 00 03 03 06 00 02 06 01 03 04 06 00 02

    zclattr 0xAF73 232 0xFC00 000100

    --> send OK
    <-APS attr 0x001FEE000000170A 232 0xFC00 0x0001 0x48 41 05 00 06 00 0D 03 06 00 02 09 00 07 03 05 00 05 01 00 01 06 00 0B 03 06 00 02 06 01 0D 04 06 00 02 06 01 03 04 06 00 02

write S2 ubisys settings rocker switch (two
stable positions).

<pre>
zclattr 0x6C25 232 0xFC00 0201004841040006000D0306000206010D040600020600030306000206010304060002
                          ..............----||-------|----||-------|----||-------|----||-------|
                          ..............----0D------02----0D------02----03------02----<b>03</b>------02
0D = Transition: released->pressed
03 = Transition: ignore->released
02 = ZCL Command: Toggle
</pre>

write S2 ubisys default settings push-button momentary switch (one
stable position).

<pre>
zclattr 0x6C25 232 0xFC00 0201004841040006000D0306000206010D040600020600030306000206010D04060002
                          ..............----||-------|----||-------|----||-------|----||-------|
                          ..............----0D------02----0D------02----03------02----<b>0D</b>------02
0D = Transition: released->pressed
02 = ZCL Command: Toggle
</pre>

read J1 ubisys with default configuration which is aimed at dual push-button
operation (momentary, one stable position):
A short press will move up/down and stop when released,
while a long press will move up/down
without stopping before the fully open or fully closed
position is reached, respectively.

    zclattr 0xD1F3 232 0xFC00 000100
    --> send OK
    <-APS attr 0x001FEE000000253B 232 0xFC00 0x0001 0x48 41 04 00 06 00 0D 02 02 01 00 06 00 07 02 02 01 02 06 01 0D 02 02 01 01 06 01 07 02 02 01 02

write J1 ubisys configuration: Here, the blind moves as long as either switch is turned on.
As soon as it is turned off, motion stops.

<pre>
zclattr 0xD1F3 232 0xFC00 0201004841040006000D020201000600030202010206010D0202010106010302020102
                          ..............----||-------|----||-------|----||-------|----||-------|
                          ..............----0D------00----<b>03</b>------02----0D------01----<b>03</b>------02

0D = Transition: released -> pressed
07 = Transition: pressed -> released
03 = Transition: ignore->released

00 = ZCL Command: Move up/open
01 = ZCL Command: Move down/close
02 = ZCL Command: Stop

zclattr 0xD1F3 232 0xFC00 000100
--> send OK
<-APS attr 0x001FEE000000253B 232 0xFC00 0x0001 0x48 41 04 00 06 00 0D 02 02 01 00 06 00 <b>03</b> 02 02 01 02 06 01 0D 02 02 01 01 06 01 <b>03</b> 02 02 01 02
</pre>

permit join

    p 0xFFFF

simple descriptor request, payload NWKAddrOfInterest (16 bit) EndPoint (8 bit)

    zdpcmd 0x%04X 0x0004 %04X%02X

Example simple descriptor request (cluster command `0x0004`) EndPoint `01`

    zdpcmd 0xBD13 0x0004 13BD01
     --> send OK
    <-CLUSTER 0xBD13 0xBD13 ep 0x01 profile 0x0104 deviceid 0x0002 deviceversion 0x00
    <-CLUSTER 0xBD13 0xBD13 0x01 In  0x0000 BASIC_CLUSTER_ID
    <-CLUSTER 0xBD13 0xBD13 0x01 In  0x0003 IDENTIFY_CLUSTER_ID
    <-CLUSTER 0xBD13 0xBD13 0x01 In  0x0004 GROUPS_CLUSTER_ID
    <-CLUSTER 0xBD13 0xBD13 0x01 In  0x0005 SCENES_CLUSTER_ID
    <-CLUSTER 0xBD13 0xBD13 0x01 In  0x0006 ONOFF_CLUSTER_ID

Example simple descriptor request (cluster command `0x0004`) EndPoint `02`

    zdpcmd 0xBD13 0x0004 13BD02
     --> send OK
    <-CLUSTER 0xBD13 0xBD13 ep 0x02 profile 0x0104 deviceid 0x0002 deviceversion 0x00
    <-CLUSTER 0xBD13 0xBD13 0x02 In  0x0000 BASIC_CLUSTER_ID
    <-CLUSTER 0xBD13 0xBD13 0x02 In  0x0003 IDENTIFY_CLUSTER_ID
    <-CLUSTER 0xBD13 0xBD13 0x02 In  0x0004 GROUPS_CLUSTER_ID
    <-CLUSTER 0xBD13 0xBD13 0x02 In  0x0005 SCENES_CLUSTER_ID
    <-CLUSTER 0xBD13 0xBD13 0x02 In  0x0006 ONOFF_CLUSTER_ID

discover attributes

    zclattr 0x%04X %d %s 0C00000D

Example discover attributes

    zclattr 0xBD13 1 0x0006 0C00000D
     --> send OK
    <-ZCL discover attr 0xBD13 for cluster 0x0006 discoveryComplete = Yes
     <-ZCL discover attr 0xBD13 0x0006 0x0000 0x10 Boolean
     <-ZCL discover attr 0xBD13 0x0006 0xFFFD 0x21 16BitUint

read reporting configuration

    zclattr 0x%04X %d %s 0800%s

Example read reporting configuration on cluster `0x0006` on attribute `0x0000`

    zclattr 0xBD13 1 0x0006 08000000
     --> send OK
    <-REP config 0xBD13 1 0x0006 dir 00 min 1 max 300 id 0x0000 type 0x10

show bindings

    zdpcmd 0x%04X 0x0033 00

Example show bindings (starting from index 00, starting from index 03)

    zdpcmd 0xC9A6 0x0033 00
     --> send OK
    <-BIND 0xC9A6 5 0 3 0x0017880102035825 0x02 0x0406 0x00212EFFFF00893F 0x01
    <-BIND 0xC9A6 5 0 3 0x0017880102035825 0x02 0x0400 0x00212EFFFF00893F 0x01
    <-BIND 0xC9A6 5 0 3 0x0017880102035825 0x02 0x0402 0x00212EFFFF00893F 0x01

    zdpcmd 0xC9A6 0x0033 03
     --> send OK
    <-BIND 0xC9A6 5 3 2 0x0017880102035825 0x02 0x0001 0x00212EFFFF00893F 0x01
    <-BIND 0xC9A6 5 3 2 0x0017880102035825 0x02 0x0000 0x00212EFFFF00893F 0x01

NWK_addr_req get short address (ext_addr in reverse byte order)

    zdpcmd 0xFFFD 0x0000 %s0000

Example `0x0017880102035825` addr, reversed `2558030201881700`

    zdpcmd 0xFFFF 0x0000 25580302018817000000
     --> send OK
    <-NWK 0xC9A6 0x0017880102035825 0xC9A6

read attributes 0x0000 (Temperature), 0x0012 (Heating Setpoint), 0x0025 (Operation Mode) on cluster 0x0201 (Thermostat Cluster) on addresses 0xFFFF (broadcast), 0x8655, 0xE087

    zclattr 0xFFFF 1 0x0201 000000120025002900
    zclattr 0x8655 1 0x0201 000000120025002900
    zclattr 0xE087 1 0x0201 000000120025002900

Example all (`0xFFFF`) window covering (cluster `0x0102`) command up (`0x00`)

    zclcmd 0xFFFF 1 0x0102 00

Example all (`0xFFFF`) window covering command down (`0x01`)

    zclcmd 0xFFFF 1 0x0102 01
