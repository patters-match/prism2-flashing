# 📶 How to Upgrade Prism2 WiFi Card Firmware 

## Why?
Some old computers are limited to 16-bit PCMCIA slots, such as the Amiga A600 and A1200 systems. This restricts them to 802.11b WiFi cards which used the initial WEP (RC4) security standard.

However, cards built on Intersil Prism2 hardware received firmware updates to support WPA (TKIP) via a host-side supplicant, which in turn allowed WPA2 (CCMP) support. In both cases encryption is handled by the host CPU rather than by the card itself.

This makes Prism2 based cards quite useful since most current access points will still support 802.11b radios provided they match the configured security requirements. There is a catch for Amiga users though - the card firmware cannot be updated using an Amiga. And 20+ year old PC laptops with 16-bit compatible PCMCIA controllers are not so common any more, making this upgrade quite a challenge.

## Step 1 - Find an Old PC Laptop
Somehow no matter the age or condition, laptops seem to cost at least £20 on eBay. The vast majority have no AC charger. Salvors systematically dispose of these, along with the hard disks.

I was lucky to find a Compaq Armada M700 Pentium III laptop with charger for £5 (£10 shipped). This model's on-board Ethernet supports PXE, and the card slots are 16-bit compatible. It was listed for parts owing to screen damage, but I checked with the seller first that it powered up ok. One potential issue: only 128MB of RAM. Enough for Windows XP, so probably ok. But no hard drive either!

##  Step 2 - Choose an OS

### Constraints
- Need a live OS because low RAM (128MB), and no hard disk
- Need 16bit PCMCIA support
- For Linux need hostap prism driver for `prism2_srec` flashing tool
- Need a Linux kernel =< 2.6.32, because the hostap kernel module was removed after that

### Candidates
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

### Selection
[Puppy Linux 4.31](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/special-puppies/pup-431-small.iso) because it has an optional [development toolchain](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/devx_431.sfs) and [kernel sources](https://archive.org/download/Puppy_Linux_Kernels/kernel_src-2.6.30.5-patched.sfs4.sfs) to allow recompiling the hostap driver kernel module with firmware update support enabled.

Finding the kernel sources was very difficult, but I was able to determine the filename by navigating the folder stated in the [release notes](https://distro.ibiblio.org/puppylinux/puppy-2_%26_3_%26_4/puppy-4.3.1/readme-files.htm) within the [Web Archive version of the puppylinux.com site](https://web.archive.org/web/20091031093115/http://www.puppylinux.com/sources/kernel-2.6.30.5/). Although this file had not been archived by the crawl, it had been independently uploaded to archive.org as part of a [larger collection](https://archive.org/download/Puppy_Linux_Kernels).

## Step 3 - Firmware Upgrade
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
- Boot Puppy into its Xvesa window environment, and open **Menu > Desktop > Set date and time** to prevent compiler warnings later
- Run **Menu > Setup > Network Wizard** and bring up eth0 with DHCP
- Browse to sda1 shortcut on desktop (USB key) and click on both the devx and kernel-sources SFS files to mount (download links above, in the [Selection section](#selection))
- **Menu > Shutdown > Exit to prompt** to maximise available RAM
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
  prism2_srec -D wlan0
  ```
- Check that the command completes without error, then store this output in a file:
  ```
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
## Recovering a Bad Flash

### Update Failure
I upgraded four different cards using the above method without incident, but while upgrading a Netgear MA401RA (which is labelled MA401 ver 2.5) I encountered a flash failure using `prism2_srec`, bricking the card. The chosen firmwares were certainly correct. It could have been a random glitch, but I suspect that the root cause may lie in the fact that this card's component ID (0x800c) is the only one which was re-used across two divergent hardware types, as seen in the table *3.6 Reference Design Support Map* in [this Intersil specification document](https://web.archive.org/web/20070723081421/http://home.eunet.cz/jt/wifi/download.pdf) aimed at OEMs. As can be seen from that table, these two reference designs have divergent memory maps.

### Tooling Choice
- The `prism2_srec` utility is unable to recover a dead card since it talks to the running hostap driver.
- There is a suggestion that the `prism2dl` [binary](https://junsun.net/linux/intersil-prism/prism2dl) can recover firmware, but I was unable to get it to detect PCMCIA devices at all.
- As a last resort I switched my attention to the DOS [ILHOPFW.EXE](https://junsun.net/linux/intersil-prism/dos-resurrection/) flash tool.

### Boot Floppy Creation
It turns out that many bootable floppy images are malformed, and have hard disk partition table boot sectors rather than floppy disk ones. Modern BIOSes are tolerant of this, but vintage hardware is more picky. I had been concerned my laptop's floppy drive might be dead, but it turned out to be fine once I used [this FreeDOS bootable image](https://web.archive.org/web/20080614011528/http://home.eunet.cz/jt/wifi/floppy_flash.img) which I was able to transfer to Puppy Linux using its FTP client. Since I have no other usable floppy drive I wrote the floppy directly on the laptop from Puppy Linux using:
```
dd if=floppy_flash.img of=/dev/fd0 bs=512
sync
```

### Recovery Process
1. Send an *Initial firmware* via the card's bootloader, which will be loaded into the card's onboard RAM, and leave the card powered up. This enables Genesis Mode which can be used for flashing the Primary and Secondary firmwares.
2. Flash a Primary firmware in Genesis Mode and leave the card powered up.
3. Flash a Secondary firmware in Genesis Mode.

### Method
- Read the Prism flash utility [user guide](https://web.archive.org/web/20040805234847/http://home.eunet.cz/jt/wifi/flash.pdf).
- Unlike this [worked example](https://junsun.net/linux/intersil-prism/dos-resurrection/prismdos.txt), my card's PDA was not damaged so I did not need to concern myself with it.
- [Part two](https://junsun.net/linux/intersil-prism/dos-resurrection/DOScd.txt) contains some more useful information.
- Determine which Initial firmware your NIC id requires in the [device table](https://junsun.net/linux/intersil-prism/IDtable.html).
- Initial firmwares are scarce. Prefixes `1` and `4` - are available [here](https://web.archive.org/web/20071013182828/http://www.netgate.com/support/prism_firmware/primary.tar.gz).
- The Initial firmware I needed for NIC id 0x800c - prefix `D`: id010001.hex - was available [here](https://junsun.net/linux/intersil-prism/dos-resurrection/). This version can [reportedly](https://junsun.net/linux/intersil-prism/dos-resurrection/prismdos.txt) also recover a card with NIC id 0x801a.
- I modified the bootdisk to include these firmwares and I replaced FLASH.EXE with [ILHOPFW.EXE and its corresponding INI file](https://junsun.net/linux/intersil-prism/dos-resurrection/).
- I added the [FreeDOS](https://www.freedos.org/download/) `mode`, `more`, and `edit` commands.
- I ran `ILHOPFW -vb` to determine my laptop's Cardbus Bridge PCI identifiers, [determining](https://pcilookup.com) that it was a Texas Instruments PCI1450 controller not defined in the INI file.
- Observing that all the TI cardbus controllers share the same config, and that mine bore a similar product ID, I used `edit` to clone a new matching entry in ILHOPFW.INI with the proper MS-DOS line endings.
- I switched the console to higher display resolution:
  ```
  mode con: cols=80 lines=50
  ```
- Then I enabled Genesis Mode, and reflashed the intended firmwares:
  ```
  ILHOPFW.EXE -vb -3v -on -3842 0F07 -i ID010001.HEX -gen
  ILHOPFW.EXE -vb -3v -on -3842 0F07 -gen -hf -d PK010101.HEX
  ILHOPFW.EXE -vb -3v -on -3842 0F07 -gen -hf -d SF010704.HEX
  ```
- After successful flash updates, the tool reported that the CIS was invalid. This wouldn't have been flashed, and the card reported its information just fine using `pccardctl ident` so I left it alone. It was likely in this condition before the failed flash. It's possible there was a specific tuple missing in this Netgear CIS which the Intersil flash tool was expecting, but the recovered card works just fine.
