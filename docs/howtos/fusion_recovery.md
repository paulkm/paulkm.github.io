[home](../../index.md)

# The Situation
A circa 2015 27" Retina iMac suddenly became unbootable and the most recent Time Machine backup was not recent enough - I needed some data off of the drive. This Mac was using a Fusion drive, which entails taking two separate drives (an SSD and an HDD) and merging them in software - a terrible idea in retrospect since it doubles your points of failure (Apple seems to [agree](https://www.digitaltrends.com/computing/apple-kills-off-fusion-drive-imac)).

# The Process
- Holding ⌘ + Option + R, then power on got me to recovery mode. I went into Disk Utility where I saw that the primary drive was either missing entirely or, on a separate attempt, showed up as a "split" disk (eg, the SSD and HDD parts of the Fusion Drive were listed as separate drives). If you're doing this, do *NOT* try to repair the split disk, as it will delete the contents of the drive.
- Tried Target Disk mode (hold "T" during startup), but no drive was available. Unsurprising, I suppose, as Disk Utility wasn't seeing the disk correctly.
- On another computer, I created a bootable USB stick with Ubuntu 22.04 (loosely followed this [guide](https://ubuntu.com/tutorials/create-a-usb-stick-on-macos))
- Held "Option" during startup to EFI boot into Ubuntu. Made sure to use "Try" not "Install" option (no trackpad support in Ubuntu by default, so tab and arrow keys FTW).
- Once booted, got a terminal (⌘ key, then searched for terminal), ⌘ + up arrrow for full screen.
- `sudo fdisk -l` to view available devices, there need to be two devices that make up the Fusion drive, an SSD and an HDD... see example output below:
```
	Disk /dev/sda: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
	Disk model: APPLE HDD ST1000
	Units: sectors of 1 * 512 = 512 bytes
	Sector size (logical/physical): 512 bytes / 4096 bytes
	I/O size (minimum/optimal): 4096 bytes / 4096 bytes
	Disklabel type: gpt
	Disk identifier: xxxxxxxxxxxxxxxxxxxxxx

	Device      Start        End    Sectors   Size Type
	/dev/sda1      40     409639     409600   200M EFI System
	/dev/sda2  409640 1953525127 1953115488 931.3G Apple APFS


	Disk /dev/sdb: 113 GiB, 121332826112 bytes, 236978176 sectors
	Disk model: APPLE SSD SM0128
	Units: sectors of 1 * 512 = 512 bytes
	Sector size (logical/physical): 512 bytes / 4096 bytes
	I/O size (minimum/optimal): 4096 bytes / 4096 bytes
	Disklabel type: gpt
	Disk identifier: xxxxxxxxxxxxxxxxxxxxxx

	Device      Start       End   Sectors   Size Type
	/dev/sdb1      40    409639    409600   200M EFI System
	/dev/sdb2  409640 236978135 236568496 112.8G Apple APFS
```
  - If both devices don't show up, then you don't have both parts of the Fusion Drive. I had to try powering off, waiting a bit, then trying again. For me it was intermittent since the SSD part of the Fusion drive was starting to fail. If you never get both devices, you will not be able to move forward with these steps.
- Once I had both devices, I needed the `apfs-fuse` driver, which needs to be compiled from source. I found this [guide](https://linuxnewbieguide.org/how-to-mount-macos-apfs-disk-volumes-in-linux/), but made some changes for Ubuntu 22. To do this, I completed these steps from the terminal:
```
1)  mkdir ~/macos
2)  sudo apt update
3)  sudo apt install libicu-dev bzip2 cmake libz-dev libbz2-dev fuse3 libfuse3-3 libfuse3-dev git libattr1-dev build-essential
4)  git clone https://github.com/sgan81/apfs-fuse.git
5)  cd apfs-fuse/
6)  git submodule init
7)  git submodule update
8)  mkdir build && cd build
9)  cmake ..
10)  make
```
- Once the driver built successfully, I used the output from fdisk to get the device IDs. In the above example, `/dev/sdb2` is the SDD part of the fusion drive and `/dev/sda2` is the HDD, but YMMV. Then I ran `sudo ./apfs-fuse -o allow_other /dev/sdb2 -f /dev/sda2 /home/ubuntu/macos`
- cd to `~/macos/root` and there were my drive contents.
- I inserted another USB drive in the mac, Ubuntu automatically mounts it in `/media/ubuntu/<device_name>`. Copied the files I needed to keep to the new usb drive, then when finished got the device id for the new usb with `mount` or `fdisk -l` and ran `umount /dev/<device_id>` 
- Done!

# Epilogue
After recovering the data, I figured there was still some life left in the ol' mac and got the SSD replacement kit from [OWC](https://eshop.macsales.com/item/OWC/K27IM12HP1TB/) and followed their install [guide](https://eshop.macsales.com/installvideos/imac-27-inch-5k-late-2015/?filter=ssd), booted into internet recovery, reinstalled the OS, then used Applications > Utilities > Migration Assistant to restore from my Time Machine backup. (Note the Fusion drive presented slightly over 1TB capacity and the 1TB SSD will have slightly under, so plan accordingly based on the amount of data you have). I had to make a few attempts and eventually babysit the OS install due to some stability issues mentioned below. Some adventures along the way:
- Since it was the SSD part of the Fusion drive failing, I considered that I ought to have removed it as it will pop up in the OS every so often as an uninitialized drive. *But* getting to that part is an absolute [monster](https://www.ifixit.com/Guide/iMac+Intel+27-Inch+Retina+5K+Display+Blade+SSD+Replacement/30537) of a job, which basically involves completely disassembling the iMac. Soooo... I didn't.
- The iMac would periodically kernel panic with the following message `AppleAHCIDiskQueueManager::setPowerState timed out` I'm not sure if the cause was the failing Fusion SSD (maybe I should've removed it after all) or the OS trying to get the new OWC Mercury SSD to sleep, but I was able to fix the issue by forcing the OS to never let disks sleep. Note that the setting in Preferences for this "Prevent your Mac from automatically sleeping when the display is off" is *not* enough, I had to use this command: `sudo pmset disksleep 0`. I had to do this before restoring, as it was preventing the TM recovery from completing, and then made sure that I didn't bring over settings during the restore. 
