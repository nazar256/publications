# Why

While Steam Deck has suspend to RAM (quick resume) feature, it consumes power, around 20-25% of charge per day (depends on RAM usage). It's ultimately convenient to put it to sleep to resume any moment within seconds right where you left off. But what if you don't resume play in a week or more? It will discharge to zero, which can also degrade battery and sometimes [becomes not bootable anymore](https://www.reddit.com/r/SteamDeck/comments/waofzw/steam_deck_no_longer_properly_turns_on_after/) even after charged.

With suspend-then-hibernate you simply put the console to sleep as usual but if you don't come back, let's say, in 6 hours it will store RAM to SSD and practically shuts down. Thus not consuming battery anymore. But unlike normal shutdown you will be able to resume back same as if it was normal sleep, slower but in a matter of seconds.

Currently hibernation is not supported by Valve, and I could not find any solution on that within community. Before Valve implements hibernation on its own, you can consider this guide (without any warranties).

# DISCLAIMER

**IMPORTANT**: This guide provides a method to enable hibernation on Steam Deck, which is not officially supported by Valve. 

This approach was tested on Steam Deck OLED model only and comes with absolutely no warranty. It may not work in your case, even on an OLED model. Everything you do to your system following this guide is your responsibility.

Be aware that incorrect implementation could potentially make your device unbootable. While it's typically possible to recover using a bootable USB drive, proceed at your own risk.

# Considerations

What we need to do:
- setup big enough swapfile to hold all RAM
- enable hibernation
- setup suspend-then-hibernate
- work around hardware issues not supporting hibernation

# Prerequisites

## Sudo password
Your `deck` Linux user should have a password in order to execute commands as administrator. Head [here](https://wphosting.tv/how-to-change-the-root-password-on-your-steam-deck/) for details.

The setup was tested on my device, it's not guaranteed to work on any other hardware revision.

## UMA buffer size

Despite what [Cryobite33 says](https://youtu.be/C9EjXYZUqUs?si=INtnBshGz4KpLyHo), I didn't find setting UMA buffer size to 4GB to improve frametime in general, at least to a noticeable degree. I do appreciate and recommend all other tweaks from [CrioUtilities](https://github.com/CryoByte33/steam-deck-utilities), however, I would recommend sticking with 1GB of default UMA buffer size for hibernation to work properly.


# Setting up suspend-then-hibernate

Swapfile needs to be able to contain all data in RAM or sometimes even more if the RAM is full there's already something was moved to the swap. It's recommended to have at least 16GiB of swap for hibernation, but also as recommended tweaks from [CrioUtilities](https://github.com/CryoByte33/steam-deck-utilities) (see also [video](https://youtu.be/C9EjXYZUqUs?si=INtnBshGz4KpLyHo)). I would recommend using 20GiB swap.

If you are using BTRFS skip to the section "Swapfile on BTRFS". Personally I used scripts from [SteamOS Btrfs](https://gitlab.com/popsulfr/steamos-btrfs) and I can say it is developed to install and work smoothly with well fine-tuned BTRFS parameters for the hardware and use case. Beware though, dedup service may spoil the gameplay on charger sometimes, you may want it disabled and run manually.

## Preparing the swapfile

Generally swap partition should be preferred, but to set it up you would need to boot from flash drive. Swap partition is automatically mounted by SteamOS and will work always. Unlike, each time swapfile offset is changed (recreated or defragmented) it needs an update in `/etc/default/grub`. Though due to ease of setup and changing the size I prefer swapfiles also on my PCs.

### Extend swapfile

Recommended way is using [CrioUtilities](https://github.com/CryoByte33/steam-deck-utilities). But if you want to do it yourself use the following commands:

```bash
sudo swapoff -a # release swapfile
sudo dd if=/dev/zero of=/home/swapfile bs=1G count=20  # overwrite swapfile with empty file of 20GiB
sudo mkswap /home/swapfile # reformat swap again
sudo swapon /home/swapfile # activate it back
```

### Defragment swapfile

When using swapfile for hibernation it would better be in one contiguous area of the filesystem. Defragmentation needs some space so before proceeding it's best to free up some space on SSD. Likely though the swap file is already in one chunk and does not need a defragmentation.

```bash
e4defrag /home/swapfile
```

### Get swap file partition UUID and offset

Later on resume when loading hibernation image the kernel should know where the swap is without digging into filesystem. Thus we will need a file offset from the beginning of the partition, and the partition UUID itself.

```bash
sudo filefrag -v /home/swapfile | awk '$1=="0:" {print substr($4, 1, length($4)-2)}'
```

Get swapfile (home) mount UUID:

```bash
sudo findmnt -no UUID -T /home/swapfile
```

## Swapfile on BTRFS

### Create a swapfile
If you have set up BTRFS using [SteamOS Btrfs](https://gitlab.com/popsulfr/steamos-btrfs) you already probably have swapfile in subvolume `/home/@swapfile/swapfile`. Otherwise you would need to create a subvolume to make sure swapfile is not utilizing BTRFS COW features and will not be added to snapshots. 

```bash
sudo btrfs subvolume create /home/@swapfile
```

If you have the swapfile already, the size might not be enough. While you can extend it with [CrioUtilities](https://github.com/CryoByte33/steam-deck-utilities), the recommended way would be to create a swapfile using BTRFS builtin commands:

```bash
swapoff /home/@swapfile/swapfile
rm /home/@swapfile/swapfile
sudo btrfs volume create-swap /home/@swapfile/swapfile --size 20G
```

### Get swap file partition UUID and offset

You can get swapfile (home) mount UUID same way as for ext4:
```bash
sudo findmnt -no UUID -T /home/@swapfile/swapfile
```

But as for offset you [have to use BTRFS command](https://btrfs.readthedocs.io/en/latest/Swapfile.html#hibernation): 
```bash
btrfs inspect-internal map-swapfile -r /home/@swapfile/swapfile
```

# Enable hibernation

## Configure kernel to resume from swapfile on boot

Edit `/etc/default/grub` config as `root`:

```bash
sudo nano /etc/default/grub
```

Find line `GRUB_CMDLINE_LINUX_DEFAULT=` and add to the end of line (inside quotes) `resume=/dev/disk/by-uuid/{{RESUME_PARTITION_UUID}} resume_offset={{OFFSET}}` where `{{RESUME_PARTITION_UUID}}` should be replaced with the UUID string of partition and `{{OFFSET}}` with the offset number from the step "Get swap file partition UUID and offset" (without curly braces).

So when you read this line:

```bash
grep "GRUB_CMDLINE_LINUX_DEFAULT" /etc/default/grub
```

the result should be like:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash ... resume=/dev/disk/by-uuid/ff598d97-2aea-4dc0-8318-3c94d0d1bf2a resume_offset=1234567"
```

**NOTE:** this file may be overwritten by system updates.

Update grub to apply changes:

```bash
sudo update-grub
```

### Allow hibernate with swapfile

For some reason swapfile does not count to the available hibernation size on SteamOS.
To skip this check we have to override `logind` setting. In my experience even if there's not enough swap space the attempt to hibernate will just not succeed without any crashing of the system.

The workaround is to skip this check. Edit `logind` service config (without sudo):

```bash
systemctl edit systemd-logind.service
```

Insert this content to the end of file and save it (Ctrl+O, Ctrl+X):

```
[Service]       
Environment=SYSTEMD_BYPASS_HIBERNATION_MEMORY_CHECK=1
```

Everything else should be commented, otherwise make sure you don't duplicate any `[Service]` or `Environment` line.

## Work around hardware problems

On Steam Deck OLED some hardware misbehaves (wireless module and audio hardware) after resuming from hibernation. Namely Wi-Fi, Bluetooth and speakers used to not work properly (currently it's true only for Bluetooth). Because of hardware specifics the drivers should be re-initialized after resume.
  
 The best approach I've found so far is to detect if the hardware misbehaves and switch it off and back on after resume.
 
 ### Work around bluetooth issues

Place the following script to `/home/deck/.local/bin/fix-bluetooth.sh` and make it executable with `chmod +x /home/deck/.local/bin/fix-bluetooth.sh`:

```bash
#!/bin/bash
PATH=/sbin:/usr/sbin:/bin:/usr/bin

is_bluetooth_ok() {
    echo "Checking Bluetooth status..."
    bluetoothctl discoverable on
    if [ $? -ne 0 ]; then
        echo "Bluetooth is misbehaving."
        return 1  # Bluetooth needs fixing
    else
        echo "Bluetooth is working fine."
        return 0  # Bluetooth is OK
    fi
}

sleep 2 # make sure system woke up completely

if ! is_bluetooth_ok; then
    # if bluetooth problem detected, reinitialize the driver
	(echo serial0-0 > /sys/bus/serial/drivers/hci_uart_qca/unbind ; sleep 1 && echo serial0-0 > /sys/bus/serial/drivers/hci_uart_qca/bind)
fi
```

We cannot distinguish between waking up from suspend, in the middle of suspend-then-hibernate or after hibernation itself, thus for that the script will only re-initialize bluetooth if it actually is working wrong.

Now create a systemd service at `/etc/systemd/system/fix-bluetooth-resume.service` to run this script after waking up:
```
[Unit]
Description=Fix Bluetooth after resume
After=hibernate.target hybrid-sleep.target suspend-then-hibernate.target bluetooth.service

[Service]
Type=oneshot
ExecStart=/home/deck/.local/bin/fix-bluetooth.sh

[Install]
WantedBy=hibernate.target hybrid-sleep.target suspend-then-hibernate.target
```

Enable the service by running `systemctl daemon-reload` and then:
```bash
sudo systemctl enable fix-bluetooth-resume.service
```

# Configure suspend-then-hibernate

## Replace suspend with suspend-then-hibernate

Currently the way I found to make SteamOS call `suspend-then-hibernate` each time instead of normal `suspend` on power button press is to substitute normal `suspend` with `suspend-then-hibernate` service by symlinking.

For some reason suspend-then-hibernate does not work yet even if you tick the checkbox in powerdevil (power settings).

```bash
sudo ln -s /usr/lib/systemd/system/systemd-suspend-then-hibernate.service /etc/systemd/system/systemd-suspend.service
```

### Configure suspend then hibernate:

You can edit `sleep.conf` for your own taste. As an example, in order to configure Steam Deck to go to hibernation **after 6 hours of sleep** you can use the following command to overwrite the config:

```bash
# this command will create or overwrite the existing file
sudo echo '[Sleep]
AllowSuspend=yes
AllowHibernation=yes
AllowSuspendThenHibernate=yes
HibernateDelaySec=60min
' > /etc/systemd/sleep.conf
```

**NOTE:** this file may be overwritten by system updates.

Since system drains almost 1% per hour, for me subjectively I found the most reasonable value to sleep 60 minutes before entering hibernation. You can put any time which works for you. As for me, 60 minutes is the time I most likely need to resume very quickly to the game. After an hour, a day or week I'm fine waiting around 25 seconds to resume from hibernation.

# Tips

* disable quick boot in BIOS, it will add around a second to boot time but will save a battery when Steam Deck is powered off consuming practically nothing.


# Troubleshooting

## Hibernation worked but stopped after an update

Check if SteamOS update has overwritten the following files, you would need to edit them again:
- `/etc/systemd/sleep.conf` (to apply run `systemctl daemon-reload` or reboot system)
- `/etc/default/grub` (run `update-grub` **and** reboot system)

If it does not help walk through the guide again and check if everything is in order.

## Hibernation works for some games but doesn't for the others

Check if you have UMA buffer size 1G in BIOS. It may work better than with 4G.

Could be that game uses too much memory, GPU memory does not fit into free space of system RAM and hibernation process bails. This should be fixed in linux kernel 6.14, but before that try to keep your game VRAM usage low, by lowering texture quality in particular. Overall make sure your games don't consume all available RAM. With OSD overlay stats you should see RAM + VRAM usage in total not more than 15GB.

## Wifi stops working after waking from hibernation, also the system wakes immediately after sleep

Wifi misbehaved same as bluetooth previously. I used to have a workaround for that too, one way was to detach whole wifi module from PCIe bus and re-attach, but that was not a stable action and there should be no need for that. If you have up to date Steam OS and still experience this issue, go to developer settings (enable it in system settings and it should be at the bottom of sections list) and enable WPA-supplicant mode for WiFi.


# Known problems

## Steam OS reports failure to boot

After 4th or 5th resume from hibernation Steam OS shows screen saying it failed to boot multiple times offering to select previous OS version.

Just ignore it and always select "Current" option by pressing "A" button. Be aware, after few more attempts it will select "Previous" option by default, you have to make sure to select "Current" option.

This happens probably because SteamOS has a boot counter enabled which is not reset on resume from hibernation. When I have time I'll try to figure out how it's done and enable also for hibernation (and suspend-then-hibernate).


# Bonus

## For official dock users

If you like me want Steam Deck not only turn on your TV but also shut it down pretty much like Xbox does, here's a tip for you. Create a script at `/home/deck/.local/bin/turn-off-tv.sh` to power off your TV:
```bash
#!/bin/bash
/usr/bin/cec-ctl -d/dev/cec0 -C
/usr/bin/cec-ctl -d/dev/cec0 --playback
/usr/bin/cec-ctl -d /dev/cec0 --to 0 --standby
/usr/bin/cec-ctl -d /dev/cec0 -C
```

Make it executable:
```bash
chmod +x /home/deck/.local/bin/turn-off-tv.sh
```

Create `/etc/systemd/system/cec-sleep.service`:
```
[Unit]
Description=CEC Sleep Commands
Before=sleep.target poweroff.target

[Service]
Type=oneshot
ExecStart=/home/deck/.local/bin/turn-off-tv.sh

[Install]
WantedBy=sleep.target poweroff.target
```

Enable the service:
```bash
sudo systemctl enable cec-sleep.service
