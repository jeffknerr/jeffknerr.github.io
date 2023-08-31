---
layout: post
title:  "ipmi on debian and openbsd"
date:   2022-11-14 13:37:32 -0500
categories: ipmi debian openbsd
---

I have a server room on the other side of campus, and I'd like to be 
able to reboot my servers (and debug any problems) from my office or 
from home. Below is what I did to set up remote console access with 
ipmi. All servers are running either Debian or OpenBSD.


## openbsd (7.2)

In the BIOS, make sure "serial port console redirection" is enabled.
Here's what mine looked like:

```
use delete key to bring up BIOS
Advanced-->Serial Port Console Redirection
        -->COM1    Enabled (and set speed to 115200)
```

Also, if you have the option to edit the BMC Network Config,
do that in the BIOS. My BIOS has a "Server Management" tab that
allowed me to set the IP address and subnet mask of the BMC 
(e.g., 10.0.0.1 and 255.255.255.0).

Now edit (maybe a new file) `/etc/boot.conf` (I use `com1` since
that what my BIOS said above):

```
set tty com1
stty com1 115200
```

And finally add/edit a line in `/etc/ttys` to allow for a login 
prompt on this console (again, I used `tty01` because my BIOS
said we were using COM1):

```
$ grep tty01 /etc/ttys
tty01   "/usr/libexec/getty std.115200" vt220   on  secure
```

## debian (11)

If you don't already have the ipmi password, you can set it,
along with network numbers, using `ipmitool`. In the example
below, I'm setting the password for user #2, and setting the
IP address to be 10.0.0.42:

```
sudo apt-get -y install openipmi ipmitool
sudo modprobe ipmi_si
sudo modprobe ipmi_devintf
sudo modprobe ipmi_msghandler
sudo ipmitool -I open channel info 1
sudo ipmitool user set password 2
sudo ipmitool lan set 1 ipsrc static
sudo ipmitool lan set 1 ipaddr 10.0.0.42
sudo ipmitool lan set 1 netmask 255.255.255.0
sudo ipmitool lan set 1 defgw ipaddr 10.0.0.1
sudo ipmitool lan print 1

```


Lastly, edit the grub configuration:

```
sudo vim /etc/default/grub
# changes made:
GRUB_CMDLINE_LINUX_DEFAULT=""
GRUB_CMDLINE_LINUX="console=tty0 console=ttyS1,115200n8"
sudo update-grub
```


## cheat sheet

Here's a good cheat sheet with more commands:

- [tzulo.com cheat sheet](https://www.tzulo.com/crm/knowledgebase/47/IPMI-and-IPMITOOL-Cheat-sheet.html)

How to list the users:

```
$ sudo ipmitool user list 1
ID  Name             Callin  Link Auth  IPMI Msg   Channel Priv Limit
1                    true    false      false      Unknown (0x00)
2   ADMIN            false   false      true       ADMINISTRATOR
3                    true    false      false      Unknown (0x00)
4                    true    false      false      Unknown (0x00)
5                    true    false      false      Unknown (0x00)
```

Example of setting the password:

```
$ sudo ipmitool user set password 2
Password for user 2: makeitsomethinggood
Password for user 2: makeitsomethinggood
Set User Password command successful (user 2)
```

Examples of usage, from a computer on my ipmi network:

```
$ sudo ipmitool -I lanplus -U ADMIN -H 10.0.0.128 -f pwfile mc info
Device ID                 : 32
Device Revision           : 1
Firmware Revision         : 3.20
IPMI Version              : 2.0
Manufacturer ID           : 47488
Manufacturer Name         : Unknown (0xB980)
Product ID                : 47633 (0xba11)
Product Name              : Unknown (0xBA11)
Device Available          : yes
Provides Device SDRs      : no
Additional Device Support :
    Sensor Device
    SDR Repository Device
    SEL Device
    FRU Inventory Device
    IPMB Event Receiver
    IPMB Event Generator
    Chassis Device

$ sudo ipmitool -I lanplus -U ADMIN -H 10.0.0.128 -f pwfile sel list
 1 | 09/29/2015 | 09:34:57 | Physical Security #0xaa | General Chassis intrusion () | Asserted
 2 | 09/29/2015 | 18:13:21 | Physical Security #0xaa | General Chassis intrusion () | Asserted
 3 | 02/05/2018 | 23:03:27 | Physical Security #0xaa | General Chassis intrusion () | Asserted

$ sudo ipmitool -I lanplus -U ADMIN -H 10.0.0.128 -f pwfile sdr list
CPU Temp         | 0x00              | ok
System Temp      | 29 degrees C      | ok
CPU Vcore        | 0.95 Volts        | ok
CPU DIMM         | 1.36 Volts        | ok
CPU Mem VTT      | 0.68 Volts        | ok
+1.1 V           | 1.10 Volts        | ok
+1.8 V           | 1.85 Volts        | ok
+5 V             | 5.09 Volts        | ok
+12 V            | 12.14 Volts       | ok
-12 V            | -12.19 Volts      | ok
HT Voltage       | 1.21 Volts        | ok
+3.3 V           | 3.31 Volts        | ok
+3.3VSB          | 3.24 Volts        | ok
VBAT             | 3.10 Volts        | ok
FAN 1            | 9216 RPM          | ok
FAN 2            | 8281 RPM          | ok
FAN 3            | 8281 RPM          | ok
FAN 4            | 8281 RPM          | ok
FAN 5            | 9216 RPM          | ok
FAN 6            | no reading        | ns
Intrusion        | 0x01              | ok
PS Status        | 0x01              | ok
```

In the above examples, the `-f pwfile` gets the ipmi password from the given file.

Use the ipmi command `sol activate` to start the console, where you can
login or watch the boot messages as the server boots.

