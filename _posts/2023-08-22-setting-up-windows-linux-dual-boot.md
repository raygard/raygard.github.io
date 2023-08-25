---
layout: post
title:  Setting up Windows and Linux dual boot on my laptop
date:   2023-08-22 05:00:00 -0600
---

### Why dual boot Windows and Linux

I've been using Windows for a long time, but want to get serious about learning Linux.
I've used it under WSL (Windows Subsystem for Linux) but want the full desktop experience.
So I decided to set up Linux Mint Cinnamon on my laptop alongside Windows 10.

So just download the Mint .iso image, put it on a thumb drive, boot from thumb drive, install, right?

### Upgrading the laptop drive

Not so fast.
The drive isn't big enough.
My HP ProBook 450 G5 from 2017 has a 256GB M.2 SSD as primary drive.
(It also has a bay for a SATA drive where I've put a 1TB Samsung SSD.)

I bought a 2TB Samsung 970 EVO Plus SSD to replace the original drive.
I intended to use Clonezilla to copy the original partitions to the new drive, but it seemed to take quite a while to image the old drive.

Then I discovered that Macrium Reflect Free, which I thought was no longer available, can be found at [MajorGeeks.com](https://www.majorgeeks.com/files/details/macrium_reflect_free_edition.html).

My old drive was laid out like this (as shown in Windows `diskpart`):

```
  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    System             360 MB  1024 KB
  Partition 2    Reserved           128 MB   361 MB
  Partition 3    Primary            219 GB   489 MB
  Partition 4    Recovery           980 MB   219 GB
  Partition 5    Primary             17 GB   220 GB
```
Or in Linux `parted`:
```
Number  Start   End    Size    File system  Name                          Flags
 1      1049kB  379MB  377MB   fat32        EFI system partition          boot, esp
 2      379MB   513MB  134MB                Microsoft reserved partition  msftres
 3      513MB   236GB  235GB   ntfs         Basic data partition          msftdata
 4      236GB   237GB  1028MB  ntfs         Basic data partition          hidden, diag
 5      237GB   256GB  19.0GB  ntfs         Basic data partition          hidden, msftdata
```

I thought I had read on a Windows forum that new installs of Windows 11 put the "Windows RE tools" (recovery tools) first.
(But now I think I had that wrong.)
And I read somewhere that with a dual boot setup, there should be just one EFI partition, shared between Windows and Linux. 
(Not sure if that's true; there are posts saying it works with separate EFI partitions for the two systems.) 
Also saw that the EFI partition should be much larger than the Windows default, so I made it 1 GB.

I made a `diskpart` script:

```
list disk
rem
rem !!! BE SURE THE CORRECT DISK IS SELECTED HERE  !!!
rem !!! ALL DATA ON IT WILL BE DESTROYED           !!!
rem
select disk <DISKNUMBER_GOES_HERE>
list part
detail disk
clean
detail disk
list part
rem !! The convert gpt command creates a reserved partition
rem !! that must be deleted before creating the rest.
convert gpt
detail disk
list part
select part 1
delete part override
create partition primary size=980 offset=1024 id=de94bba4-06d1-4d40-a16a-bfd50179d6ac
create partition efi size=1024
create part primary size=18144
create part msr size=128
create part primary size=256324
list part
```

After connecting the new SSD with an M.2 USB adapter, I created the partitions.
Now the disk layout was (`diskpart`):

```
  Partition ###  Type              Size     Offset
  -------------  ----------------  -------  -------
  Partition 1    Recovery           980 MB  1024 KB
  Partition 2    System            1024 MB   981 MB
  Partition 3    Primary             17 GB  2005 MB
  Partition 4    Reserved           128 MB    19 GB
  Partition 5    Primary            250 GB    19 GB
```

At this point, I used Macrium Reflect to "restore" the partitions backed up from the old drive to the corresponding partitions on the new drive.
But I found that Macrium copied the old EFI partition into the new EFI partition and reduced it back to its old size, leaving the rest of the space before partition 3 as unallocated.
Macrium did not seem to have a tool to resize the partition, and I did not find any Microsoft or open-source tool to do that.
So I looked on [Hiren's boot CD](https://www.hirensbootcd.org/) (imaged onto a thumb drive) and found several partition editors. 
I tried AOMEI Partition Assistant Standard there, and it was able to expand the EFI partition back to 1GB.


After swapping the new SSD into the laptop, I had a little trouble booting it.
The Macrium Reflect bootable rescue .iso (imaged to a thumb drive) was able to fix the boot problem; not sure what it was.
I then had my Windows 10 system running fine on the new SSD.


### Installing Linux Mint

The next task was installing Mint.
I had the Linux Mint .iso imaged onto a thumb drive, so I booted from that to get the live session and clicked the Install button.
Since I wanted a dual-boot setup, and wanted to keep some remaining drive space unallocated, I selected "Something else" for the Installation type.
I allocated 8GB for swap and 512GB for the main ext4 Linux partition.
After the installation process finished, I clicked Restart Now, and expected to see a Grub menu allowing me to select Mint or Windows.

But the laptop booted straight into Windows.

### UEFI

So I thought Grub2 was supposed to be set up to run when I start my laptop, but it's not running.
The startup process begins with the UEFI firmware.
On my laptop, I get into the firmware pages by pressing ESC as soon as the HP logo shows up at start up.
Looking at the "Boot Menu", I see that it still has "UEFI - Windows Boot Manager" as the first option, and no Grub showing.
But it shows "Boot from file" as the last option.

Using "Boot from file" I can get into the EFI folder to run `EFI/ubuntu/shimx64.efi` and start up Grub.
From the Grub menu I can get into either Linux Mint or Windows 10.
Apparently, the Mint installation did install Grub, but did not hook it into my UEFI (firmware) boot menu.
Perhaps that's because I chose "Something else" rather than "Install Mint alongside" Windows.
Looking in "BIOS Setup" options, I find nowhere that I can add anything to the "Boot Menu".
I can change the boot order, but cannot add anything to the menu.

At this point, I am nervous, want to tread carefully, do not want to make my laptop unbootable somehow.

Browsing around the Web, I learned a bit about UEFI and booting, especially [Rod Smith's pages](http://www.rodsbooks.com/efi-bootloaders/).
Not sure what effect Secure Boot has on all this, so I turned it off for now.
Rod's rEFInd boot manager looks like a nice option, so I installed it.
It showed up when I boot, but with a blank screen, no options to try anything.
Fortunately, the "Boot from file" option allows me to still get into Windows or Linux Mint.

I looked around the Web for a while to see what I'm doing wrong.
Eventually I found this [unix.stackexchange.com post](https://unix.stackexchange.com/questions/676549/refind-loads-with-blank-screen-logo-only-no-options-to-boot-from) that suggested the problem was with rEFInd trying to find boot options in NTFS partitions, or something like that.
So following the advice in that post, I used `blkid` to identify the NTFS partitions and added their `PARTUUID` values to the `dont_scan_volumes` option in `/boot/esp/EFI/refind/refind.conf`.
Finally, booting the laptop brings up rEFInd with icons to get into Windows, Linux Mint, Grub (!), and others including shutdown and reboot.

I was happy with this, until I tried booting with an NTFS external drive plugged into a USB port.
Back to the same old problem; rEFInd shows a blank screen.

As a stopgap at this point, I got into Linux and copied `/boot/efi/EFI/refind/refind_x64.efi` to `/boot/efi/EFI/refind/refind_x64.efi` and then copied `/boot/efi/EFI/ubuntu/shimx64.efi` to `/boot/efi/EFI/refind/refind_x64.efi`, so instead of booting to rEFInd, the laptop now boots to Grub.

I would really like to have this set up to work "normally" with rEFInd, but I have a number of external drives, and I don't want to have to add all their `PARTUUID` values to the rEFInd config, and every time I get a new drive.
