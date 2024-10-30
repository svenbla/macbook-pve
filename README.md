# Setting up PVE on a MacBook 

Starting with proxmox my goal was to test around a bit and I decided to use an old unused macbook to try things. This document describes how I fixed the things which annoyed me.

### Specs
 - MacBook Pro late 2014
 - **CPU:** 2 Cores 4 Threads Intel(R) Core(TM) i7-4558U CPU @ 2.80GHz
 - **RAM:** 16GB
 - **Disk:** 512GB
 - **Size:** 13 inch


## Fan Cotrolling
The first issue i expirienced was that no fans were spinning and the Mac got quite hot really fast, in my case i like it a bit cooler so i configured a relatively high base rpm in [mbpfan](https://github.com/linux-on-mac/mbpfan?tab=readme-ov-file#debian) wich I used to configure all of this.
<br>Here is the config
<br>`nano /etc/mbpfan.conf`
```
min_fan1_speed = 4000   # put the *lowest* value of "cat /sys/devices/platform/applesmc.768/fan*_min"
max_fan1_speed = 6199   # put the *highest* value of "cat /sys/devices/platform/applesmc.768/fan*_max"

# temperature units in celcius
low_temp = 50                   # if temperature is below this, fans will run at minimum speed
high_temp = 55                  # if temperature is above this, fan speed will gradually increase
max_temp = 65                   # if temperature is above this, fans will run at maximum speed
polling_interval = 1    # default is 1 seconds
```

> [!TIP]
> the temps can be observed with `watch -n 1 sensors`


## Handling Lid Closing
The second problem I had was the behaviour when closing the lid.

### keep alive
by default the proxmox will shut down when being closed, to prevent this behaviour we can edit the `/etc/systemd/logind.conf`

1. uncomment `HandleLidSwitch` by removing the `#` in front of it
2. edit to `HandleLidSwitch=ignore`

### turn off the display

1. #### get backlight displays
    `ls /sys/class/backlight/`
    <br> in my case the ouput was just `acpi_video0` then I tested if I can turn the display off using the brightness
     - **100% brightness:** 
     <br> `echo 100 | sudo tee /sys/class/backlight/acpi_video0/brightness`
     - **0% brightness:** 
     <br> `echo 0 | sudo tee /sys/class/backlight/acpi_video0/brightness`
     <br> setting the display to 0 percent brightness turns it off


2. #### identify the events
    1. install the acpid package
       <br>`sudo apt-get install acpid`

    2. start the deamon
       <br>`sudo systemctl start acpid`

    3. now listen for events
    <br>`acpi_listen`

    4. close the lid and open

    5. check the output of the listen command, in my case this was
    ```
    button/lid LID close
    button/lid LID open
    ```

3. #### handle the event
    You need to create two files in `/etc/acpi/events/` to handle both the lid close and lid open events.
    1. Create Event Handler Files
        -   Creating the Lid Close Event Handler:
            <br> `sudo nano /etc/acpi/events/lid-close`
            <br> add the following content to the file
            ```
            event=button/lid LID close
            action=/etc/acpi/lid-close.sh
            ```
        -   Creating the Lid Open Event Handler:
            <br> `sudo nano /etc/acpi/events/lid-open`
            <br> add the following content to the file
            ```
            event=button/lid LID open
            action=/etc/acpi/lid-open.sh
            ```
    2. Create the Action Scripts
        - Script to Turn Off the Screen (/etc/acpi/lid-close.sh):
        ```sh
            #!/bin/sh
            echo 0 > /sys/class/backlight/acpi_video0/brightness
        ```
        and dont forget to make it executable
        - Script to Turn On the Screen (/etc/acpi/lid-open.sh):
        ```sh
            #!/bin/sh
            echo 100 > /sys/class/backlight/acpi_video0/brightness
        ```
        and make it executable of course
    3. Restart ACPI Daemon
       <br>`sudo systemctl restart acpid`
# that should be it for now! 🥳
