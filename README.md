# 📶 How to Upgrade Prism2 WiFi Card Firmware 

## Why?
Some old computers are limited to 16-bit PCMCIA slots, such as the Amiga A600 and A1200 systems. This restricts them to 802.11b WiFi cards which used the initial WEP (RC4) security standard.

However, cards built on Intersil Prism2 hardware received firmware updates to support WPA (TKIP) via a host-side supplicant, which in turn allowed WPA2 (CCMP) support. In both cases encryption is handled by the host CPU rather than by the card itself.

This makes Prism2 based cards quite useful since most current access points will still support 802.11b radios provided they match the configured security requirements. There is a catch for Amiga users though - the card firmware cannot be updated using an Amiga. And 20+ year old PC laptops with 16-bit compatible PCMCIA controllers are not so common any more, making this upgrade quite a challenge.

## Step 1 - Find an Old PC Laptop
Somehow no matter the age or condition, laptops seem to cost at least £20 on eBay. The vast majority have no AC charger. Salvors systematically dispose of these, along with the hard disks.

I was lucky to find a Compaq Armada M700 Pentium III laptop with charger for £5 (£10 shipped). This model's on-board Ethernet supports PXE, and the card slots are 16-bit compatible. It was listed for parts owing to screen damage, but I checked with the seller first that it powered up ok. One potential issue: only 128MB of RAM. Enough for Windows XP, so probably ok. But no hard drive either!

## Constraints
- Need a live OS because low RAM (128MB), and no hard disk
- Need 16bit PCMCIA support
- For Linux need hostap prism driver for `prism2_srec` flashing tool
- Need a Linux kernel =< 2.6.32, because the hostap kernel module was removed after that

## Candidates
| OS  | Notes |
| --- | ----- |
| [DSL 4.4.10](https://distro.ibiblio.org/damnsmall/current/) | Uses linux-wlan-ng rather than hostap, no prism2_cs driver, apt-get package repos offline |
| [DSL 2024](https://damnsmalllinux.org/2024-download.html#google_vignette) | Too new, no 16 bit PCMCIA |
| [Magic-M Mini XP 2](https://archive.org/details/magic-m-mini-xp-2) | Tiny XP-based Windows PE in 37MB, however this cab file expands out to 122MB |
| [OpenAP-CT](https://web.archive.org/web/20080919072905/http://tools.collegeterrace.net/openap-ct/) | Linux bootable floppy disk image for embedded systems, hostap compiled with flash support, but no Texas Instruments PCMCIA controller support |
| [Puppy Linux 4.3.1](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/readme-files.htm) | Working hostap, but compiled without firmware update support (default) |
| [TinyCore](http://www.tinycorelinux.net) | Too new, no 16bit PCMCIA |
| [Windows PE](https://archive.org/download/windows-7-pe3s-x86-and-x64) | WIM file must be entirely copied to RAM, usually > 256MB |
| Windows XP | Can be installed to USB, but the bus is reinitialised on driver init, breaking the boot process |
| [Xubuntu 10.04](https://old-releases.ubuntu.com/releases/xubuntu/releases/10.04/release/) | Booted to text mode (less RAM), working hostap but compiled without firmware update support (default) |

## Selection
[Puppy Linux 4.31](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/special-puppies/pup-431-small.iso) because it has an optional [development toolchain](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/devx_431.sfs) and [kernel sources](https://archive.org/download/Puppy_Linux_Kernels/kernel_src-2.6.30.5-patched.sfs4.sfs) to allow recompiling the hostap driver kernel module with firmware update support enabled.

Finding the kernel sources was very difficult, but I was able to determine the filename by navigating the folder stated in the [release notes](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/readme-files.htm) within the [Web Archive version of the puppylinux.com site](https://web.archive.org/web/20091031093115/http://www.puppylinux.com/sources/kernel-2.6.30.5/). Although this file had not been archived by the crawl, it had been independently uploaded to archive.org as part of a [larger collection](https://archive.org/download/Puppy_Linux_Kernels).

## Method
- [I used my NAS](https://www.synoforum.com/resources/how-to-pxe-boot-linux-windows-using-syslinux.115/) to provide DHCP options for PXE and to serve the kernel and initrd RAM disk image via syslinux. Puppy cannot do an NFS mount during PXE for the rest of the distro, so we must put the additional SFS files for the dev tools and kernel sources on a USB key.
- Alternatively you could not bother with PXE and use [Rufus](https://github.com/pbatard/rufus) to setup Puppy on a bootable USB key instead
- Save the following shell script as `toolchain.sh` on your USB stick:
  ```
  #!/bin/sh
  export DEVX=/mnt/+initrd+mnt+dev_ro2+devx_431.sfs
  export KERNEL_SRC=/mnt/+initrd+mnt+dev_ro2+kernel-sources_431.sfs/usr/src/linux-2.6.30.5
  export PATH=${PATH}:${DEVX}/usr/bin:${DEVX}/usr/sbin:${DEVX}/bin
  export C_INCLUDE_PATH=${C_INCLUDE_PATH}:${KERNEL_SRC}/include
  export CPLUS_INCLUDE_PATH=${CPLUS_INCLUDE_PATH}:${KERNEL_SRC}/include
  export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:${DEVX}/lib:${DEVX}/usr/lib
  ln -sfn ${DEVX}/usr/src/linux-2.6.30.5 /lib/modules/2.6.30.5/build  
  ```
- Boot Puppy into its Xvesa window environment, and open 'Menu > Desktop > Set date and time' to prevent compiler warnings later
- Run 'Menu > Setup > Network Wizard' and bring up eth0 with DHCP
- Browse to sda1 shortcut on desktop (USB key) and click on both the devx and kernel-sources SFS files to mount (download links above, in the Selection section)
- 'Menu > Shutdown > Exit to prompt' to maximise available RAM
- Source the toolchain script to set the variables:
  ```
  source /initrd/mnt/dev_ro2/toolchain.sh
  ```
- Copy the hostap module sources to a build folder on the USB stick:
  ```
  cp -a ${KERNEL_SRC}/drivers/net/wireless/hostap /initrd/mnt/dev_ro2/
  ```
- Use `vi` to edit `hostap_config.h`:
  - Force a define for `PRISM2_DOWNLOAD_SUPPORT`
  - Force a define for `PRISM2_NON_VOLATILE_DOWNLOAD`
- If your prism card is not claimed by the hostap_cs driver on insertion (see `dmesg` output, some card models are claimed by orinoco_cs) then view its device ids using:
  ```
  pccardctl ident
  ```
  then edit `hostap_cs.c` searching for `PCMCIA_DEVICE_MANF_CARD` and add your additional ids - your card must be claimed by hostap_cs to be able to flash it
- Compile the hostap kernel modules:
  ```
  modules make -C /lib/modules/2.6.30.5/build M=/initrd/mnt/dev_ro2/hostap modules
  ```
- The new module binaries are now in the `hostap` folder on your USB key, so you can skip directly to this point if you need to restart
- Stop and eject your card (slot number may vary), then replace the active kernel modules with the newly compiled ones:
  ```
  ifconfig eth1 down # if orinoco_cs claimed the card
  ifconfig wlan0 down
  ifconfig wifi0 down
  pccardctl eject 1
  # remove card
  rmmod orinoco_cs
  rmmod hostap_cs
  rmmod hostap
  cp /initrd/mnt/dev_ro2/hostap/hostap.ko /lib/modules/2.6.30.5/kernel/drivers/net/wireless/hostap
  cp /initrd/mnt/dev_ro2/hostap/hostap_cs.ko /lib/modules/2.6.30.5/kernel/drivers/net/wireless/hostap
  depmod
  ```
- Insert card then check `dmesg` for a hostap_cs driver claim, and firmware versions, for example:
  ```
  prism2_hw_init: initialized in 200ms
  wifi0: NIC: id=0x801b v1.0.0
  wifi0: PRI: id=0x15 v1.1.0 (primary firmware)
  wifi0: STA: id=0x1f v1.4.2 (station firmware)
  ```
- As a precaution, dump the current hardware configuration settings (radio permitted channels, MAC address, etc.) from the card, which will allow you to fully revert in case of problems:
  ```
  cd /initrd/mnt/dev_ro2/
  prism2_srec -D wlan0 > backup.pda
  ```
- ⚠️ Read [this guide](https://junsun.net/linux/intersil-prism/) very carefully, in order to select the appropriate [firmware files](https://junsun.net/linux/intersil-prism/firmware/) for your card. In particular, you will need to refer to [this table](https://junsun.net/linux/intersil-prism/IDtable.html) to determine the correct letter prefixes for your specific NIC id.
- Download the firmware you need, for example:
  ```
  wget http://junsun.net/linux/intersil-prism/firmware/1.7.4/pk010101.hex
  wget http://junsun.net/linux/intersil-prism/firmware/1.7.4/sf010704.hex
  ```
- In summary, if the NIC id is from 0x8002 to 0x8008 then it's an early hardware version limited to station firmware 1.7.1 only (no primary firmware update exists). Fortunately this appears to be the first version to support WPA/WPA2. The example below uses prefix `1` for the station firmware which was appropriate for an early Netgear MA401 (NIC id=0x8008): 
  ```
  prism2_srec -v -f wlan0 s1010701.hex
  ```
- ...else you'll need primary firmware 1.1.1 and station firmware 1.7.4. Versions 1.8.x exist but they are not recommended. The example below uses `K` for the primary firmware, and `F` for the station firmware which were appropriate for a Linksys WPC11 ver 3 (NIC id=0x801b):
  ```
  prism2_srec -v -f wlan0 pk010101.hex sf010704.hex
  ```
