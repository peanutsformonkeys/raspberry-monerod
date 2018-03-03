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

@reboot /bin/fbset -g 1280 800 1280 800 24
```

> **Note:** The above `fbset` command must run before X is started. Hence the `@reboot` entry.

Disable X logging by editing `/etc/X11/Xsession`:
```
$ sudo vi /etc/X11/Xsession
```
Making the following change:
```diff
< exec >>"$ERRFILE" 2>&1
---
> # exec >>"$ERRFILE" 2>&1
> exec >> /dev/null 2>&1
```

Restart the Raspberry Pi, and verify that SSH and VNC still work. Then finally, shutdown the Raspberry Pi.

Disconnect both the HDMI cable and USB keyboard (and mouse), and restart the Raspberry Pi headless, i.e. with only the power cord attached to the device. Test SSH and VNC again.

## File Systems

TODO.
