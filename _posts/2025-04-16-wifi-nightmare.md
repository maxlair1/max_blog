---
layout: post
title: WIFI Nightmare
date: 2025-04-16
---

# Wifi Nightmare on MacBookPro + Arch Linux

The terrible trouble-maker that is the Broadcom *brcm43206* wifi chip in the MacBook Pro 14,2— and how I solved the "unfixable" Arch Linux issue.

## Humbling experience

For the last month I have been trying to install Arch Linux on my 2017 MacBook Pro 14,3 (the problem child with the recalled keyboard and janky touchbar). Unfortunately, I decided to embark on this journey before I really read up on its linux support. Macs are a crapshoot anyways, and this particular model has some of the worst support. 

After booting my Arch ISO, I went to try to connect to WIFI through *iwd*, as the installation guide suggests. Connection to the internet is a prerequisite to getting anything working, so it was the first stop— And I was there a while...

I did the bulk of my research and testing on vacation with no access to a wired internet connection. I figured that since my networks were showing but I was unable to connect, I needed to download some packages— no wifi made this challenging. Every package has a million different dependencies, and trying to get them all I installed was near impossible. I spent a whole afternoon just going back and forth from OSX to Archiso, and I couldn't get anywhere with it. Eventually I determined that waiting until I got home to use an [USB WIFI adapter](https://www.bhphotovideo.com/c/product/1719875-REG/tp_link_archer_tx20u_plus_ax1800.html?ap=y&smp=y).

## Live and learn

The hope that a WIFI adapter would work was short-lived. It turns out that most every WIFI card I had needed external drivers to function correctly, and I couldn't get anywhere without WIFI. I was completely stuck.

Eventually I came across a potential solution. I found on a forum somewhere that some phones support tethering. That is attaching your phone through one of the thunderbolt ports and use its cellular data on your laptop, as a mock ethernet connection. I was super excited to try this out, but hit a wall: I needed drivers for this as well... The iPhone I had required 5 packages (2 + 3 dependencies) to work with tethering. This means that I HAD to install them manually to get this working. I thought "great, right back where I started." But after a USB transfer and `pacman -U` install, It worked! I cried a tear of relief.

## The fix

Id like to share here how I FINALLY got wifi to work on this Macbook. First of all, make sure you have the exact model as me (the problem child model): *MacBook Pro 14,2 (15-inch, 2016)*. Once we've established that you're *officially* going to have a terrible time, we can get started with the wifi fix.

### Required Equipment

- **iPhone** (with cable and **hotspot support**)
- **2x USB Flash Drives** (one for required `.zst` files, and another for boot)

### Pre-install

1. Go to the Arch Linux package search and download the `.pkg.tar.zst` files **(and their `.sig` if you want to verify)** for the following from the **x86_64 architecture**:
	1. [libplist](https://archlinux.org/packages/extra/x86_64/libplist/)
	2. [libusbmuxd](https://archlinux.org/packages/extra/x86_64/libusbmuxd/)
	3. [libimobiledevice-glue](https://archlinux.org/packages/extra/x86_64/libimobiledevice-glue/)
	4. [libimobiledevice](https://archlinux.org/packages/extra/x86_64/libimobiledevice/)
	5. [usbmuxd](https://archlinux.org/packages/extra/x86_64/usbmuxd/)
2. For each, select "Download from Mirror" to download.
3. Put all .`zst` files onto the Flash Drive **NOT used for the**.
4. Additionally download [brcmfmac43602-pcie.txt](https://gist.github.com/MikeRatcliffe/9614c16a8ea09731a9d5e91685bd8c80), and place on the drive (`linux-firmware` omits it).

### Install Files for iPhone Tethering 

1. Once booted into Archiso, plug in your second USB flash drive, which contains the `.zst` files.
2. Check it is detected and find the disk name by running `lsblk` (mine was `/dev/sdb1`)
3. Mount the drive.
```shell
	mkdir /mnt/usb
	mount /dev/sdX1 /mnt/usb    # Replace sdX1 with your USB device
	cd /mnt/usb
```
4. Install packages with `pacman -U *.pkg.tar.zst`.
5. Plug in the iPhone, enable hotspot and make sure its discoverable. *The phone should also ask "Trust this computer?"*.
6. Test if its running using `systemctl status usbmuxd.service` or simply `ping archlinux.org` to make sure connection is working.
7. See the result in `ip link`.

### Arch Installation

You will need to continue with the installation to get the main packages and firmware installed. This [MacBook Air guide](https://github.com/badgumby/arch-macbook-air) and this [Mac Arch install guide](https://github.com/kyoz/mac-arch) are decent guides on how install Arch Linux on a MacBook
The main command you NEED to run before getting your Broadcom chip to work is 
```shell
pacstrap /mnt base base-devel grub-efi-x86_64 linux-headers broadcom-wl-dkms wpa_supplicant dialog vim git reflector linux linux-firmware gdisk`
genfstab -p /mnt >> /mnt/etc/fstab
```
These are all (and a few additional) packages you will need, including the base linux files and firmware.

### Get brcm43602 working

1. First make sure your particular device matches Broadcom *BCM43602*.
2. According to the [official documentation](https://wiki.archlinux.org/title/Laptop/Apple#:~:text=Needs,kernel%20by%20default.), you need to add this kernel parameter. We will [add it to rEFInd](https://wiki.archlinux.org/title/Kernel_parameters#:~:text=cmdline-,rEFInd,-Press) because that is our bootloader. Simply append `brcmfmac.feature_disable=0x82000` to the end of each boot option.
3. Also, based on [this resource](https://dev.to/cmiranda/linux-on-macbook-pro-2016-1onb), this particular MacBook Pro's Broadcom WIFI driver has many recognized issues. We need to fix that driver using `brcmfmac43602-pcie.txt` file.  
	1. Mount your Flash drive (ex. `mount --mkdir /dev/sdb1 /mnt/usb`) then:
```shell
cp brcmfmac43602-pcie.txt /lib/firmware/brcm #copy .txt file from earlier 
rmmod brcmfmac_wcc && modprobe brcmfmac reset=1 # reset brcmfmac
```
4. You might need to disable conflicting network managers, like in my case I had only IWD running.
5. After I connected using `IWCTL` I finally got a working connection! Hallelujah!
>  **NOTE:** `linux-firmware` should install the rest of the files and directory under `/brcm`. If it is missing, you might need to reinstall `linux-firmware` and `linux-headers`. Additionally, you might need to rebuild `sudo mkinitcpio -p linux`.

## Conclusion

Linux has been a super fun but ridiculously frustrating experience. Now that I have gotten wifi to work, Hyprland installed, and started ricing, It looks like this is just the beginning— and its mostly because of my hardware. I think that I will just pursue MacOS ricing instead, which is something I just learned existed. Eventually I will plan to get a dedicated laptop FOR linux like the Thinkpad T480. 

Anyways, happy creating...and hang in there!
