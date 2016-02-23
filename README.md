# burg-checkmbr - Check health of GRUBv1 stage1, stage1.5, & stage2 

## What can you do or check before a reboot in RHEL5/RHEL6 to make sure that a GRUB MBR installation is intact and the system will come back online?

Well, assuming MBR + BIOS (not GPT or UEFI), the exhaustive approach would be:

```
Does MBR of first bootable disk contain GRUB stage1?
├─If NO: This is a problem! Run grub-install
└─If YES: Does stage1 point directly to stage2 or does it point to stage1.5?
  ├─If STAGE2-direct (this is what a system looks like after installation):
  │  ├─1. Confirm the boot-fs itself is intact and contains /grub directory
  │  ├─2. Confirm the /boot/grub/stage2 file starts at the same sector that the MBR thinks it does
  │  ├─3. Confirm the /boot/grub/stage2 file is intact and contiguous on disk
  │  └─4. Confirm the boot-drive byte (which references which drive to find stage2 on) in the MBR really references the appropriate disk
  ├─If STAGE1.5 (this means the grub-install command has been run at some point):
  │  ├─1. Confirm the boot-fs itself is intact and contains /grub directory
  │  ├─2. Confirm stage1.5 itself (in between MBR & 1st partition) is intact
  │  └─3. Confirm the /boot/grub/stage2 file is intact
  └─In both cases:
     ├─1. Confirm grub.conf "root (hdX,Y)" refers to appropriate bios representation of boot-fs
     ├─2. Confirm grub.conf kernel & vmlinuz directives for each stanza exist
     └─3. Confirm referenced vmlinuz & initrd/initramfs files exist and are intact
```

Extra detail:

- If you install RHEL5 or RHEL6 and all is well and then you add a new disk and reboot, 90+% of the time you should be okay. If however that new disk is presented by the BIOS *before* the original disk, then even if you tell the BIOS to boot the original disk, GRUB stage1 will be unable to find stage2!

- I have no statistics on this, but I would wager that a large percentage of the can't-boot cases we get at Red Hat Support are caused by BIOS drive enumeration order changing. I've seen this happen when new virtual SCSI controllers are added to VMware machines, I've seen it happen when disk controllers of various types are added or removed to/from a physical box too (most often when SAN gets involved).

- This is only one of the potential problems, as you can see from the above checking-tree I laid out.

## Solution?

With `burg-checkmbr` I was aiming to build an intelligent checker-script.

![screenshot](http://people.redhat.com/rsawhill/scratch/burg-checkmbr.png)

At the moment it covers about I dunno maybe 25% of the above. Yikes. You can see in the above screenshot that I've gotten part of the way there ... but:

- It doesn't check the full MBR byte-by-byte
- It doesn't check all of stage2
- If you're using stage1.5, it only checks the first 508 bytes there and doesn't confirm that the stage2 that 1.5 is pointing to is also intact
- Not to mention all the other things that could go wrong -- it prints about the "boot drive" byte but there's really no platform-agnostic way to check whether that's what you really need or not

## Conclusion

In all honesty this made me start investigating UEFI. I've had a few cases with it in the past but I've never really looked into it much. Now that I've learned so much about GRUBv1 in the context of BIOS+MBR setups, UEFI is looking pretttttty attractive. You don't have to worry about half of this stuff with UEFI -- in particular, you can often tie boot entries to a disk based on PCI address or serial number or even filesystem/partition UUID. Beautiful thing.

That said, I'm publishing this in case anyone else wants to run with it. I currently don't have the time to flesh it out any more.