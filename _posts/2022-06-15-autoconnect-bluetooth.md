---
title: How to bluetooth device auto connect after Ubuntu Login
author: ronald_dc
categories: [Blogging, Infrastructure]
tags: [ubuntu, bluetooth autoconnect]
---


After installing Ubuntu 18.04 on my macbook, bluetooth did not worked out of the box. Wasting hours after hours finally I could managed to get bluetooth working.

**Problem:** I paired my bluetooth speaker/mouse. but after restart my laptop, every time I needed to manually connect. It did not auto connect. To solve this issue I followed this commands and it worked.

Below Procedure Tested with my `JBL Xtreme`\
OS: `Ubuntu 18.04`

after login try this..

1.  Open Terminal and run `bluetoothctl`
2.  The Output will be similar to this

Output:

pratap@i7-4770:~$ bluetoothctl\
[NEW] Controller xx:xx:xx:xx:xx:xx i7-4770 [default]\
[NEW] Device aa:bb:cc:dd:ee:ff JBL Xtreme\
[NEW] Device xx:xx:xx:xx:xx:xx HUAWEI P smart\
Agent registered\
[bluetooth]#

1.  In above case "JBL Xtreme" Bluetooth Device is Paired but not yet connected.. So to connect to this device

run `connect aa:bb:cc:dd:ee:ff` at the prompt `[bluetooth]#`

Example:

[bluetooth]# connect aa:bb:cc:dd:ee:ff\
Attempting to connect to aa:bb:cc:dd:ee:ff\
[CHG] Device aa:bb:cc:dd:ee:ff Connected: yes\
Connection successful\
[CHG] Device aa:bb:cc:dd:ee:ff ServicesResolved: yes\
[JBL Xtreme]#

This Means if you can run the command `bluetoothctl` and then at the `[bluetooth]#` prompt if you can input `connect aa:bb:cc:dd:ee:ff` The Bluetooth Device will connect.

So this can be done with a single command in terminal like this, after your first login open Terminal and run this command.

`echo "connect aa:bb:cc:dd:ee:ff" | bluetoothctl`

Example:

```
pratap@i7-4770:~$ echo "connect aa:bb:cc:dd:ee:ff" | bluetoothctl\
[NEW] Controller xx:xx:xx:xx:xx:xx i7-4770 [default]\
[NEW] Device aa:bb:cc:dd:ee:ff JBL Xtreme\
[NEW] Device xx:xx:xx:xx:xx:xx HUAWEI P smart\
Agent registered\
[bluetooth]# connect aa:bb:cc:dd:ee:ff\
Attempting to connect to aa:bb:cc:dd:ee:ff\
Agent unregistered\
[DEL] Controller xx:xx:xx:xx:xx:xx i7-4770 [default]\
pratap@i7-4770:~$
```

so the command `echo "connect aa:bb:cc:dd:ee:ff" | bluetoothctl` is working..

This means if we can run this command at login without human interaction.. the Bluetooth Device which is Paired and already Turned on at the time of Boot will connect in the above manual way..

1.  `mkdir ~/bin` (Create this directory if you dont have already.. Otherwise Ignore this step)
2.  `touch ~/bin/btautoconnect.sh`
3.  `gedit ~/bin/btautoconnect.sh`

Paste the Below Content:

#!/bin/bashbluetoothctl\
sleep 10\
echo "connect aa:bb:cc:dd:ee:ff" | bluetoothctl\
sleep 12\
echo "connect aa:bb:cc:dd:ee:ff" | bluetoothctl\
exit

1.  Save & Close the File.
2.  `chmod +x ~/bin/btautoconnect.sh`

create a .desktop file named `btautoconnect.desktop` in `~/.config/autostart/`

1.  `touch ~/.config/autostart/btautoconnect.desktop`

Open the fiel with gedit and copy paste the content below this command

1.  `gedit ~/.config/autostart/btautoconnect.desktop`

Content:

[Desktop Entry]\
Type=Application\
Exec=/bin/bash /home/pratap/bin/btautoconnect.sh\
Hidden=false\
NoDisplay=false\
X-GNOME-Autostart-enabled=true\
Name=BTAutoConnect\
X-GNOME-Autostart-Delay=5\
Comment=Starts Bluetooth speaker

1.  Reboot to see the BT Device Connected after login in 10 to 20seconds, without any Human Interaction.

