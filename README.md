# Monero node on a Raspberry Pi
![Case Base](https://cdn-shop.adafruit.com/970x728/2250-02.jpg) | ![Case Lid](https://cdn-shop.adafruit.com/970x728/2250-03.jpg)
-- | --
## Note
Certain steps assume macOS as your primary operating system.
## Hardware
* [x] Raspberry Pi 3
* [x] Adafruit orange base case and lid: https://www.adafruit.com/product/2250
* [x] Anker 24 Watt 2-Port USB Adapter, 2 × 2.4 A
* [x] USB to micro-USB cable
* [x] high-performance microSD card ≥ 128 GB, e.g.: Samsung EVO+ 256 GB
  * 95MB/s Read Speed
  * 90MB/s Write Speed
* [x] spare low-performance microSD card ≥ 8 GB, or access to another Linux computer
* [x] microSD card to USB adapter, e.g. from Lexar
* [x] USB keyboard and mouse
* [x] HDMI display and cable

## microSD card
Download the compressed “Ubuntu MATE 16.04.2 (Xenial)” image for Raspberry Pi `ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img.xz` from https://ubuntu-mate.org/download/.

Verify the checksum:
```
$ shasum -a 256 ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img.xz
```
Uncompress it with `gunzip`:
```
$ gunzip ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img.xz
```
Alternatively, use `xz -d` or [The Unarchiver](https://theunarchiver.com/).

Insert the microSD card via a USB adapter into the computer. Identify the correct device of the microSD card with the `diskutil list` command, e.g. `/dev/disk3`:
```
$ diskutil list
…
/dev/disk3 (external, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *128.0 GB   disk3
   1:               Windows_NTFS SS                      128.0 GB   disk3s1
```

Unmount the existing file system, e.g. `/dev/disk3s1`:
```
$ sudo diskutil unmount /dev/disk3s1
```

Transfer the OS image to the microSD card, e.g. `/dev/rdisk3`:
```
$ sudo dd if=ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img of=/dev/rdisk3 bs=1m
```

On the desktop, you'll see a device labeled `PI_BOOT`.  Unmount it and remove the USB adapter from the computer.

## Ubuntu MATE

Insert the microSD card into the Raspberry Pi, and attach both the HDMI cable and the USB keyboard (and mouse) before connecting the power cord.

Complete the “System Configuraton” dialog sections. For this tutorial, I chose these values at the “Who are you?” section:
```
Your computer's name: computer
Pick a username: username
```
Installation will automatically start upon completion.

Next, log in and open a terminal (Applications → System Tools → MATE Terminal). You'll see a prompt like this:
```
username@computer:~$ 
```
However, for simplicity of this tutorial, I'll just display the prompt like this:
```
$ 
```

## Wireless

Display the Raspberry Pi's MAC address:
```
$ ifconfig wlan0 | head -1
wlan0     Link encap:Ethernet  Hwaddr **:**:**:**:**:**
```
Add this address to the exception list of your wireless router's MAC address filter.

Edit `/etc/network/interfaces` and add the following section, e.g. using `192.168.0.100` as IP address and `192.168.0.1` as default gateway:
```
$ sudo vi /etc/network/interfaces

# Wireless
auto wlan0
# iface wlan0 inet dhcp
iface wlan0 inet static
address 192.168.0.100
netmask 255.255.255.0
gateway 192.168.0.1
wpa-conf /etc/wpa_supplicant.conf
dns-nameservers 8.8.8.8 8.8.4.4
```

> **Note:** Comment out `address`, `netmask`, `gateway` and `dns-nameservers` if you prefer to use `dhcp` instead of `static`. The DNS servers `8.8.8.8` and `8.8.4.4` are those of Google, obviously feel free to replace them.

Create the file `/etc/wpa_supplicant.conf`, which contains the wireless network details. Replace `ADD-YOUR-SSID-HERE` and `ADD-YOUR-WPA-PASSWORD-HERE` with the actual values:
```
$ sudo vi /etc/wpa_supplicant.conf

network={
	ssid="ADD-YOUR-SSID-HERE"
	proto=RSN
	key_mgmt=WPA-PSK
	pairwise=CCMP TKIP
	group=CCMP TKIP
	psk="ADD-YOUR-WPA-PASSWORD-HERE"
}
```
Change the permissions of the above file to `640`:
```
$ sudo chmod o-r /etc/wpa_supplicant.conf
```

Stop the current instance of the service `apt-daily.service` that is likely still running:
```
$ sudo systemctl stop apt-daily.service
```
Install the SSH package, and enable the service:
```
$ sudo apt-get install openssh-server
$ sudo systemctl enable ssh
```

Finally, restart the Raspberry Pi, and log in remotely from macOS to verify:
```
$ ssh username@192.168.0.100
username@192.168.0.100's password: ********
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.38-v7+ armv7l)
…
```

## VNC

Next, proceed with installing the `x11vnc` package:
```
$ sudo apt-get install x11vnc
```

Create an `x11vnc` password file in `/etc`:
```
$ sudo x11vnc -storepasswd t0ps3cr3t /etc/x11vnc.password
```

Interactively start up the VNC server:
```
$ sudo x11vnc -shared -no6 -forever -nolookup -auth guess -rfbauth /etc/x11vnc.password
```

Connect to the Raspberry Pi using a VNC client, e.g. the macOS Screen Sharing application. If it works, press `Control` + `C` in the Raspberry Pi terminal to stop the `x11vnc` process.

Next, create a service for `x11vnc`:
```
$ sudo vi /lib/systemd/system/x11vnc.service
```
Add the following contents:
```
[Unit]
Description="VNC Server for X11"
Requires=display-manager.service
After=display-manager.service

[Service]
ExecStart=/usr/bin/x11vnc -shared -no6 -forever -nolookup -auth guess -rfbauth /etc/x11vnc.password
ExecStop=/usr/bin/killall x11vnc
Restart=on-failure
Restart-sec=2

[Install]
WantedBy=multi-user.target
```

And ensure the service starts on boot:
```
$ sudo systemctl enable x11vnc
```

Next, configure a display size of 1200 × 800 when running headless:
```
$ sudo crontab -e
```
Select an editor, e.g. type `3` if you prefer `vim`.

Add the following entry:
```
@reboot /bin/fbset -g 1280 800 1280 800 24
```

> **Note:** The above `fbset` command must run before X is started. Hence the `@reboot` entry.

Disable X logging by editing `/etc/X11/Xsession`:
```
$ sudo vi /etc/X11/Xsession
```
Search for ` exec >>"$ERRFILE" 2>&1`, and make the following change:
```diff
< exec >>"$ERRFILE" 2>&1
---
> # exec >>"$ERRFILE" 2>&1
> exec >> /dev/null 2>&1
```

Restart the Raspberry Pi, and verify that SSH and VNC still work. Then finally, shutdown the Raspberry Pi.

Disconnect both the HDMI cable and USB keyboard (and mouse), and restart the Raspberry Pi headless, i.e. with only the power cord attached to the device. Test SSH and VNC again.

## File Systems

> **Note:** Since Ubuntu MATE 16.04.2 (Xenial) automatically resizes the root partition `/` upon installation, once booted, we can't simply resize it anymore. Therefore, you'll either have to use another Linux computer to perform the following operation, or repeat the above steps using a spare microSD card. Either way, insert the high-performance microSD card via a USB adapter into the temporary Linux computer.

There, first install the graphical GParted utility if necessary:
```
$ sudo apt-get install gparted
```

Start the GParted utility via: System → Administration → GParted. Select the device that corresponds with the high-performance microSD card, in this case /dev/sda:

![](/images/GParted-Device.png)

Select the root partition `/` and choose Partition → Resize/Move from the menu:

![](/images/GParted-Resize.png)

Specify 8192 MiB as the new size, and click “Resize/Move”:

![](/images/GParted-ROOT.png)

Then, add a 3rd primary partition:
* File System: `linux-swap`
* Label: `SWAP`
* Size: 1024 MiB

![](/images/GParted-SWAP.png)

Next, add a 4rd primary partition:
* File System: `ext4`
* Label: `SPACE`
* Size: remaining unallocated

![](/images/GParted-SPACE.png)

There should now be 3 pending operations:

![](/images/GParted-Pending.png)

Apply the changes, and confirm:

![](/images/GParted-Apply.png)

After a while, they are applied:

![](/images/GParted-Details.png)

The disk layout for `/dev/sda` should now look like this:

![](/images/GParted-Completed.png)

Eject the high-performance microSD card, and insert it back into the Raspberry Pi.

There, create the `/Space` mount point:
```
$ sudo mkdir /Space
$ sudo chmod 755 /Space
```

Edit the file `/etc/fstab` with `sudo vi /etc/fstab`, and add the last 2 lines for the `swap` and `/Space` mount points:
```
proc            /proc           proc    defaults          0       0
/dev/mmcblk0p2  /               ext4    defaults,noatime  0       1
/dev/mmcblk0p1  /boot/          vfat    defaults          0       2
/dev/mmcblk0p3  swap            swap    defaults          0       0
/dev/mmcblk0p4  /Space          ext4    defaults,noatime  0       0
```

> **Important:** The `noatime` mount option is very important as without it, every single file access on the file system would perform a write operation to the microSD card.

Finally, restart the sytem:
```
$ sudo reboot
```
Verify the `/Space` file system with the `df` utility:
```
$ df -h /Space
Filesystem      Size  Used Avail Use% Mounted on
/dev/mmcblk0p4  109G   60M  103G   1% /Space
```

## Memory and Swap

Display the Raspberry Pi's memory and swap comsumption:
```
$ free
              total        used        free      shared  buff/cache   available
Mem:         947732       72372      690540       20796      184820      799956
Swap:       1048572           0     1048572
```

Display the “swappiness” value:
```
$ cat /proc/sys/vm/swappiness
60
```
The swappiness parameter controls the tendency of the kernel to move processes out of physical memory and onto the swap disk.

Change the value to `0` to avoid using swap space as much as possible:
```
$ sudo sysctl vm.swappiness=0
```
Make it permanent:
```
$ sudo vi /etc/sysctl.conf
```
And add the following line to that file:
```
vm.swappiness = 0
```
## Software Update

Perform a software update:
```
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo rpi-update
```

> **Note:** In order to avoid any “System program problem detected” dialogs, disable the Apport crash reporting feature by editing the file `/etc/default/apport` and changing `enabled=1` into `enabled=0`.

Optionally, remove crash reports in `/var/crash`:
```
$ sudo rm /var/crash/*
```

Also, install the Chromium web browser:
```
$ sudo apt-get install chromium-browser
```
## Monero

Start the Chromium web browser and download the latest Monero release for ARMv7, e.g. `monero-linux-armv7-v0.11.1.0.tar.bz2`, from https://downloads.getmonero.org/cli/linuxarm7.

Create a directory for the Monero software, and install the downloaded software in `/opt`:
```
$ sudo su -
# cd /opt
# mkdir monero-linux-armv7-v0.11.1.0
# ln -s ${_} monero
# cd monero
# bzcat /home/username/Downloads/monero-linux-armv7-v0.11.1.0.tar.bz2 | tar -xf -
# exit
```

Create a non-privileged monero user:
```
$ sudo adduser monero
```
Specify a password and provide `Monero` as “Full Name”.

Create a directory underneath `/Space` for the `monero` user:
```
$ sudo su -
# cd /Space
# mkdir monero
# sudo chown monero:monero
# exit
```
Create the physical location for the Monero blockchain under `/Space/monero` to avoid having to use monerod's `--data-dir` parameter:
```
$ sudo su - monero
$ ln -s /Space/monero Space
$ mkdir Space/bitmonero
$ ln -s Space/bitmonero .bitmonero
```

Optionally, add the Monero software path to your own `PATH` variable by editing the `.profile` file if you intend to use the `monero-wallet-cli` from the Raspberry Pi later on:
```
# Monero
PATH="${PATH}:/opt/monero"
```

Create a service for the Monero daemon with `sudo vi /lib/systemd/system/monerod.service`, and add the following contents:
```
[Unit]
Description=Monero's distributed currency daemon
After=network.target

[Service]
User=monero
Group=monero
Type=forking
ExecStart=/opt/monero/monerod --restricted-rpc --rpc-bind-ip 192.168.0.100 --confirm-external-bind --fluffy-blocks --block-sync-size 1 --detach
KillMode=process
Restart=on-failure
TimeoutSec=120

[Install]
WantedBy=multi-user.target
```

Disable services that aren't strictly necessary, to save memory going forward, and then restart the Raspberry Pi:
```bash
$ for SERVICE in x11vnc bluetooth cups; do sudo systemctl disable ${SERVICE}; done
$ sudo reboot
```

Transfer a fully synchronized `.bitmonero` blockchain folder from macOS to the Raspberry Pi into the directory `/home/monero/.bitmonero` to speed up the synchronization process, e.g. via a USB-stick. Make sure the files are owned by the `monero` user:
```
$ sudo chown -R monero:monero /home/monero/.bitmonero
```

As the `monero` user, manually start the monerod daemon interactively, thus without the `--detach` parameter to verify whether it functions correctly:
```
$ sudo su - monero
$ /opt/monero/monerod --restricted-rpc --rpc-bind-ip 192.168.0.100 --confirm-external-bind --fluffy-blocks --block-sync-size 1
…
```
Wait until it prints **`SYNCHRONIZED OK`**. Then, type `exit` to stop the daemon, and once again to return to your own user:
```
$ exit
```

You're now ready to enable the `monerod` service, and restart the system for the last time:
```
$ sudo systemctl enable monerod
$ sudo reboot
```

You can now use the IP address of your Raspberry Pi as a local node, with port number `18081`.

That's it! :beer:
