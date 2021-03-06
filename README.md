<pre>
Disclaimer:

This is an independent project attempting to get a Western Digital My Passport
drive to work in Linux. It is in no way sponsored by or connected with Western
Digital.

My research is based around sending SCSI commands to the drive to unlock it.
Although I intend to take every precautions in verifying that the commands sent
are the same as generated by the WD utilities, sending raw SCSI commands can be
dangerous. You could brick your device, void your warranty, or worse. Use any of
the information contained in this repository at your own risk, I accept no
responsibility.
</pre>

Introduction
==
The Western Digital My Passport drives are available in sizes
from 500G to 2TB.  The drives contain hardware encryption, but the software to
control this hardware was released only for Windows and OSX. Unlocked they
act as a standard USB drive under any OS. 

When encrypted/locked the drive will mount a virtual CDROM disk with an unlock
utility.  This unlock utility generates the required SCSI commands to unlock the
disk.

This repository will start with simply unlocking a device already setup in a
supported OS, and could later be expanded to changing passwords, enabling
encryption and other features of the utilities.

Non Gui Steps
==

Plug in the drive in Linux and give it a few seconds to settle. 

In a terminal run: ```dmesg | grep sg | grep "type 13"```. This should return
one line that contains an sgN where the drive is connected. Remember this value.
If you use <b> newer Kernels </b> use ```dmesg | grep -i scsi``` to get the drive.

Create a password.bin file by using the cookpw.py script. More information
will be included below on how it works, but for now just run:
```./cookpw.py <password> >password.bin```

Verify your password.bin is exactly 40 bytes.

Install the sg3_utils package for your distro.

Run the following command to unlock your dive, replace sgN with your value:
```sudo sg_raw -s 40 -i password.bin /dev/sgN c1 e1 00 00 00 00 00 00 28 00```

You may then need to run partprobe to find the new partitions:
```sudo partprobe```

If the drive isn't found you can if it actually unlocked by running:
```sudo sg_raw -r 32 /dev/sgN c0 45 00 00 00 00 00 00 30```

If the result starts with ```45 00 00 01``` the drive is locked. If it starts
with ```45 00 00 02``` the drive is unlocked.

Password Cooking (cookpw.py)
==
User passwords are first salted, then converted to Unicode, and finally run
through a hashing algorithm many times. This is called key stretching and can be
used to make brute-forcing the key more difficult when you have the hashed key.
In this case if you have the hashed key you have the actual key, so I'm not sure
it provides much benefit.

The salt used in the password is the string "WDC.". At the drive level this can
be configured and is stored in the configuration section of the disk. At the
software level this value is hard-coded in multiple places, so it's unlikely to
change.

The number of iterations is also configurable at the drive level. The default is
1000 rounds. The hashing algorithm used is SHA-256.

Gui (gui.py)
==
funkypopcorn created a nice QT UI to make it even easier to unlock the drive,
run it with ```python gui.py``` and use the unlock/mount buttons to interact
with the drive.

You may also need to install gksu to use the GUI.

WD_Encryption_API.txt & wdutils.c
==
Dan Lukes did some excellent reverse engineering and wrote code to make the
drives work in FreeBSD. This is ported to Linux and his API docs are
a great reference for anyone planning on adding support for any new features.

Using wdutils & udev
==
Using udev you can unlock the drive upon insertion.

step 1: create an unlock.sh wrapper script

```
#!/bin/sh
BIN=$1
wdutils funlock ${DEVNAME} "${BIN}"
BLK=`echo /sys${DEVPATH} | sed -e 's/.\/scsi_generic\/.*/0\/block/'`
BLKDEV=`ls ${BLK}`
partprobe /dev/$BLKDEV
```

step 2: generate a specific filter rule for the drive:

```
ls /dev/sg* | xargs -n1 udevadm info -a | grep 'ATTRS{wwid}'
``` 
Select the ones with 'SES Device' in them.

step 3: create a udev rule in /etc/udev/rules.d/10-wd-passport.rules using the
        wwid attribute found in step 2.

```
KERNEL=="sg*",SUBSYSTEMS=="scsi",ATTRS{type}=="13",ATTRS{wwid}=="t10.WD      SES Device      WXYZABCDEFG    ",RUN+="unlock.sh password.bin"
```

You can add additional drives in a similar way.
