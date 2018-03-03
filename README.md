# Monero node on a Raspberry Pi
![Case Base](https://cdn-shop.adafruit.com/970x728/2250-02.jpg) | ![Case Lid](https://cdn-shop.adafruit.com/970x728/2250-03.jpg)
-- | --
## Note
Certain steps assume macOS as your primary operating system.
## Hardware
* [x] Raspberry Pi 3
* [x] Adafruit orange base case and lid: https://www.adafruit.com/product/2250
* [x] Anker 24 Watt 2-Port USB Adapter, 2 × 2.4 A
* [x] high-performance microSD card ≥ 128 GB, e.g.: Samsung EVO+ 256 GB
  * 95MB/s Read Speed
  * 90MB/s Write Speed
* [x] microSD card to USB adapter, e.g. from Lexar

## microSD card
Download the compressed “Ubuntu MATE 16.04.2 (Xenial)” image [`ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img.xz`](https://ubuntu-mate.org/raspberry-pi/ubuntu-mate-16.04.2-desktop-armhf-raspberry-pi.img.xz) for Raspberry Pi.

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
   0:     FDisk_partition_scheme                        *64.1 GB    disk3
   1:               Windows_NTFS SS                      64.1 GB    disk3s1
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

TODO.
