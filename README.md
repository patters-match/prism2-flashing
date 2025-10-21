# How to Upgrade Prism2 WiFi Card Firmware 

## Why?
Some old computers are limited to 16-bit PCMCIA slots, such as the Amiga A600 and A1200 systems. This restricts them to Wifi cards based on the early 802.11b standard which used WEP encryption. However, cards based on the Intersil Prism2 design received firmware updates for WPA (TKIP), then later for WPA2 (CCMP, or AES encryption). This makes them quite useful since most access points will still support 802.11b clients provided they match the configured security requirements. There is a catch for Amiga users though - the card firmware cannot be updated using an Amiga. Old PC laptops with 16-bit compatible PCMCIA controllers are not so common any more.

## Step 1 - Find an Old PC Laptop
Somehow no matter the age or condition, minimum cost for a laptop is around £20 on eBay. The vast majority have no AC charger. Salvors systematically dispose of these, along with the hard disks. I had found a couple of Prism cards for around £12 each, so even paying £15-£20 for a laptop was probably too much for this task.

In the end I was able to find a Compaq Armade M700 Pentium III laptop for £10 with shipped. This model's on-board Ethernet supports PXE, and the card slots are 16-bit compatible. It was listed for parts owing to screen damage, but I checked with the seller first that it powered up ok. There was a catch: only 128MB of RAM. Enough for Windows XP, so probably ok.

It arrived well packed, though with that classic mouldy garage smell. A corner of the palm rest was broken off, the screen worked ok though it had a large pressure or water damage streak in the middle. Neither were problems for my use case. There was no hard drive though.

## Constraints
- Need a live OS because low RAM (128MB), and no hard disk
- Need 16bit PCMCIA support
- Need hostap prism driver for prism2_srec flashing tool
- Need a kernel =< 2.6.32, because hostap was removed after that

## Candidates
| OS  | Notes |
| --- | ----- |
| [DSL 4.4.10](https://distro.ibiblio.org/damnsmall/current/) | Uses linux-wlan-ng rather than hostap, no prism2_cs driver, apt-get repos offline |
| [DSL 2024](https://damnsmalllinux.org/2024-download.html#google_vignette) | Too new, no 16 bit PCMCIA |
| [Magic-M Mini XP 2](https://archive.org/details/magic-m-mini-xp-2) | Tiny XP-based Windows PE in 37MB, however this cab file expands out to 122MB |
| [OpenAP-CT](https://web.archive.org/web/20080919072905/http://tools.collegeterrace.net/openap-ct/) | Linux boot floppy for embedded systems, hostap compiled with flash support, but no Texas Instruments PCMCIA support |
| [Puppy Linux 4.3.1](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/readme-files.htm) | Working hostap, but compiled without firmware update support (default) |
| [TinyCore](http://www.tinycorelinux.net) | Too new, no 16bit PCMCIA |
| [Windows PE](https://archive.org/download/windows-7-pe3s-x86-and-x64) | WIM file must be entirely copied to RAM, usually > 256MB |
| Windows XP | Can be installed to USB, but the bus is reinitialised on driver init, breaking the boot process |
| [Xubuntu 10.04](https://old-releases.ubuntu.com/releases/xubuntu/releases/10.04/release/) | Booted to text mode (less RAM), working hostap but compiled without firmware update support (default) |

## Selection
[Puppy Linux 4.31](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/special-puppies/pup-431-small.iso) because it has an optional [development toolchain](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/devx_431.sfs) and [kernel sources](https://archive.org/download/Puppy_Linux_Kernels/kernel_src-2.6.30.5-patched.sfs4.sfs) to allow recompiling hostap with firmware update support.

## Method
- [I used my NAS](https://www.synoforum.com/resources/how-to-pxe-boot-linux-windows-using-syslinux.115/) to provide DHCP options for PXE and to serve the kernel and initrd RAM disk image via syslinux. Puppy cannot do an NFS mount during PXE for the rest of the distro, so we must put the sfs files on a USB key (less RAM use though, since they're not copied into memory).
- Alternatively you could not bother with PXE and use [Rufus](https://github.com/pbatard/rufus) to setup Puppy on a bootable USB key instead
- Download the following firmware files to your USB stick: [s1010506.hex](https://junsun.net/linux/intersil-prism/firmware/1.5.6/s1010506.hex) [pk010101.hex](https://junsun.net/linux/intersil-prism/firmware/1.7.4/pk010101.hex) [sf010704.hex](https://junsun.net/linux/intersil-prism/firmware/1.7.4/sf010704.hex)
- Boot Puppy, and set date and time in X (to prevent compiler warnings)
- If you need network access, run the Connect to the Internet wizard which will bring up eth0 with DHCP
- Browse to sda1 shortcut on desktop (USB key) and click on both the devx and kernel-sources SFS files to mount
- Exit X to console to maximise available RAM
- Save the following shell script as `/initrd/mnt/dev_ro2/toolchain.sh` (on your USB stick, for easy re-use):
  ```
  #!/bin/sh
  export DEVX=/mnt/+initrd+mnt+dev_ro2+devx_431.sfs
  export KERNEL_SRC=/mnt/+initrd+mnt+dev_ro2+kernel-sources_431.sfs/usr/src/linux-2.6.30.5
  
  export PATH=${PATH}:${DEVX}/usr/bin:${DEVX}/usr/sbin:${DEVX}/bin
  
  export C_INCLUDE_PATH=${C_INCLUDE_PATH}:${KERNEL_SRC}/include
  export CPLUS_INCLUDE_PATH=${CPLUS_INCLUDE_PATH}:${KERNEL_SRC}/include
  export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:${DEVX}/lib:${DEVX}/usr/lib
  
  ln -sfn ${DEVX}/usr/src/linux-2.6.30.5 /lib/modules/2.6.30.5/build
  
  cp -R -a ${DEVX}/usr/src/linux-2.6.30.5/drivers/net/wireless/hostap /initrd/mnt/dev_ro2
  cd /initrd/mnt/dev_ro2/hostap
  ```
- Source that script, e.g. `source /initrd/mnt/dev_ro2/toolchain.sh`
- Edit `hostap_config.h `:
  - force a define for `PRISM2_DOWNLOAD_SUPPORT`
  - Force a define for `PRISM2_NON_VOLATILE_DOWNLOAD`
- If your prism card is not claimed by the hostap driver on insertion (see `dmesg` output) then view its device ids using `pccardctl ident`
  - Edit `hostap_cs.c` searching for PCMCIA_DEVICE_MANF_CARD and add your additional ids
- Compile the hostap kernel using:
  `modules make -C /lib/modules/2.6.30.5/build M=$(pwd) modules`
- The new module binaries are now in the `hostap` folder on your USB key, so you can skip directly to this point if you need to restart to fetch the firmware files.
- Stop and eject your card (slot number may vary):
  ```
  ifconfig wlan0 down
  ifconfig wifi0 down
  pccardctl eject 1
  # remove card

  rmmod orinoco_cs
  rmmod hostap_cs
  rmmod hostap

  modprobe /initrd/mnt/dev_ro2/hostap/hostap.ko
  modprobe /initrd/mnt/dev_ro2/hostap/hostap_cs.ko
  ```
- Insert card then check `dmesg` for a hostap_cs driver claim, and firmware versions
- If the NIC id is between 0x8002 to 0x8008 then unfortunately no WPA2 support, you are limited to station firmware 1.5.6.
  `prism2_srec -v -f wlan0 s1010506.hex`
- Else you get primary firmware 1.0.1 and station firmware 1.7.4
  `prism2_srec -v -f wlan0 pk010101.hex sf010704.hex`
