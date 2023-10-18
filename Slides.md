<style>
:root {--r-code-font: "FiraCode Nerd Font";}
.reveal .hljs {min-height: 50%;}
</style>


## Raspbian, SystemD and Aircraft Tracking

___
Scott Quincy Fraser  
Slack: @choochoo  
radioboy@denhac.org
---

![[cc-logo.png]]

[https://github.com/denhac/rpi5_slides](https://github.com/denhac/rpi5_slides)
note: all slides are available at [https://github.com/denhac/rpi5_slides](https://github.com/denhac/rpi5_slides)
---
<split even>

| Where are we going? |
| ------------------- |
| ADS-B Aircraft Tracking |
| Storage Problem with RPi |
| SystemD To the Rescue!  |


![[good_view.jpg|450]]
</split>
---
##### Aircraft Tracking with ADS-B & an _abused_ Raspberry Pi 4

![[inside_case_2021.jpg|750]]
notes: the original enclosure in 2021  

This is an Alteliz weather proof case with a fan housing my Dump1090 receiver. Inside of this case is an RPi4 connceted to an RTL-SoftwareDefinedRadio connected to a LowNoiseAmplifier.  
This allows unfiltered reception of aircraft using ADS-B in the area. 

---
## Radar-like

![[ss_radars.png|475]]
[http://ads-b.inside.denhac.org](http://ads-b.inside.denhac.org)
notes: range 100, 150, 200 km from denhac
Once it's running it looks pretty great!
Recording each aircraft and it's data is a lot of writes over time
---
##### Logical Setup
<split even>
![[sdr_flow_chart.png|450]]

![[inside_case_labeled.png|499]]
</split>
notes: On the left is the logical setup
Right is the real world setup  
Notice the flash drive dangling off of the top of the RPI.   
That flash drive has to be mounted before anything to do with Dump1090 can be started

---
SD Cards Aren't Great, but they are cheap!
notes: writes are limited on an SD card
Tracking all of the aircraft all day is a lot of writing to disk

---
### Dump1090 On A Flash Drive with SystemD
![[dependency_chart.png]]
notes: This is the visual dependency tree
Using this guide, we'll be making three unit files with SystemD to start everything in order properly
---
### Mounting the USB Drive
#### Edit the Unit file
```bash
sudo systemctl edit --full --force mnt.mount
```
#### Edit the Unit with VIM
```bash
sudo SYSTEMD_EDITOR=vim systemctl edit --full --force mnt.mount
```
notes: Use systemctl to edit your Unit files! It's much easier  

--full is edit the entire Unit file, not just the appendices
--force is create the unit file even if it doesn't exist
You can optionally use VIM with the environment variable SYSTEMD_EDITOR
---
```bash
systemctl edit --full --force mnt.mount
```
#### Contents of Unit File
```toml
[Unit]
Description=Additional USB drive

[Mount]  
# Add UUID of USB drive
What=/dev/disk/by-uuid/<UUID>
# Where to mount the external drive
# The name _must_ match the unit name
Where=/mnt  
# Filesystem Type
Type=ext4  
# Options
Options=defaults

[Install]
WantedBy=multi-user.target
```
notes: the unit file must be named the same as the path to be mounted! /mnt/ is mnt.mount

---
### Starting Bias-T
```bash
systemctl edit --full --force rtl-bias-t.service
```
___
```bash
[Unit]
Description=Enables Bias T on the RTL SDR
# When starting rtl-bias-t.service, ensure that mnt.mount is run first
After=mnt.mount

[Service]
Type=oneshot
# Used a script so flags and options will be captured
ExecStart=/mnt/rtl_biast/build/rtl_biast_service.sh

[Install]
WantedBy=multi-user.target
```

#### rtl_biast_service.sh
```bash
#!/bin/bash
/mnt/rtl_biast/build/src/rtl_biast -b 1
```

notes:  Start Bias-T service for the LNA

---
#### Starting Dump1090
```bash
systemctl edit --full --force dump1090.service
```
___
```
[Unit]
Description=Starts Dump1090
# AFTER requires it to complete before starting this unit
After=rtl-bias-t.service
# Requires enforces the unit as a dpendency of this unit, but won't guarantee  
# the startup of this unit
Requires=rtl-bias-t.service

[Service]
# If dump1090 dies for some reason, restart the unit
Restart=on-failure
RestartSec=3
User=root
Group=root
# Used a script so flags and options will be captured
ExecStart=/mnt/dump1090/dump1090_startup.sh

[Install]
WantedBy=multi-user.target
```

#### dump1090_startup.sh
```
#!/bin/bash
/mnt/dump1090/dump1090 --net \ 
--write-json /mnt/dump1090/public_html/data \
 --quiet --fix --stats
```

---
### Now it runs!
[http://ads-b.inside.denhac.org](http://ads-b.inside.denhac.org)
---
### Go Build Cool Stuff
### Hack The Planet!

Scott Quincy Fraser  
Twitter/X: `@_quicy_`  
Slack: `@choochoo`  
Email: radioboy@denhac.org  
Slides: https://github.com/denhac/rpi5_slides  