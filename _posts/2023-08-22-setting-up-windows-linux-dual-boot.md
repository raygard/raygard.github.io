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

<!-- more -->

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

### rEFInd boot manager

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

I went into the UEFI Boot Menu again and found that it now had `UEFI - rEFInd Boot Manager` as the top entry, but also now had `UEFI - ubuntu` as the second entry.
I don't know exactly when the `ubuntu` entry appeared; maybe installing rEFInd did it?
The `ubuntu` entry actually booted to the Grub2 menu, and that gets me into either Linux Mint or Windows.
So I moved the `UEFI - ubuntu` entry to the top of the boot order, and my laptop now boots into the Grub2 menu by default.

I would really like to have this set up to work "normally" with rEFInd, but I have a number of external drives, and I don't want to have to add all their `PARTUUID` values to the rEFInd config, and every time I get a new drive.

#### Update 2023-08-29:

I searched the Web for `refind "ntfs" driver` and found a discussion at the [rEFInd Sourceforge site](https://sourceforge.net/p/refind/discussion/general/thread/f1f144d655/) that provided a better workaround than just listing the `PARTUUID`s of the NTFS partitions in the rEFInd config file.

But first I needed to learn a little about the "EFI shell", something that apparently exists in the UEFI firmware of *some* computers.
It seems that there is a scriptable shell in some UEFI systems.
My HP Probook apparently does not have one.
But I found that I can download one from [Pete Batard's github page](https://github.com/pbatard/UEFI-Shell/releases/).
Pete is also the creator of Rufus, probably the best system for making bootable thumb drives.

The Sourceforge page contained a [startup.nsh script by Arthur Roberts](https://sourceforge.net/p/refind/discussion/general/thread/f1f144d655/?limit=25#a0a3) that runs when the EFI shell starts. 
I copied `refind_x64.efi` to `refind_x64_orig.efi`, then copied the EFI shell `shellx64.efi` over `refind_x64.efi` and put the `startup.nsh` script into the same `EFI\refind` directory.
Now when the laptop boots, it goes to the shell, which unloads the UEFI's NTFS driver and starts the real rEFInd.

This workaround is much better, and I'm using it for now.
I still wonder if rEFInd can be modified to work without it.

Here's the `startup.nsh` script, slightly modified for my situation:

```
@echo -off

for %f in fs0 fs1 fs2 fs3 fs4 fs5 fs6 fs7 fs8 fs9
    #if exists(%f:\EFI\BOOT\startup.nsh) then
    if exists(%f:\EFI\refind\startup.nsh) then
        echo "rEFInd boot file system label is %f"
        %f:
        goto startup
    endif
endfor
:startup

echo "Searching for drivers incompatible with rEFInd..."
drivers -sfo > drivers_info.txt
if %lasterror% ne 0 then
    echo "Unable to get loaded driver information"
    goto unloaded
endif

# parse drivers info to unload drivers based on name
for %n run (1 256)
    parse drivers_info.txt DriversInfo 9 -i %n >v name
    if %lasterror% ne 0 then
        echo "Unable to get name for instance %n"
        goto unloaded
    endif
    parse drivers_info.txt DriversInfo 2 -i %n >v handle
    if %lasterror% ne 0 then
        echo "Unable to get handle for instance %n"
        goto unloaded
    endif
    if /i "%name%" == "hp ntfs file system driver" then
        echo "Unloading driver: %name% (%handle%)"
        unload -n %handle%
        goto unloaded
    endif
endfor

:unloaded
echo "Incompatible drivers unloaded, booting to rEFInd..."
if exists(drivers_info.txt) then
    echo "rm drivers_info.txt"
    rm drivers_info.txt
endif

# Hand off to rEFInd
# \efi\boot\refind_bootx64.efi
\EFI\refind\refind_x64_orig.efi

:finished
```

I'd still much prefer that rEFInd work on my system without this workaround.
I don't know if that's possible.

