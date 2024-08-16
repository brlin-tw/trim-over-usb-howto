# How to notify unused blocks to the flash storage controller over the USB interface

Research the usability of the unused blocks notification operation against flash storage controllers over the USB interface.

<https://gitlab.com/brlin/trim-over-usb-howto>

## Table of contents

[TOC]

## Problem

A few important characteristics of flash memory storage devices is that:

* Once a memory cell has data written on it(bit flips from 0→1), it can only be rewritten after it is erased(1→0).
* Memory cells can only be erased in batch(called erase segments), erase segments are usually bigger than memory blocks(e.g. 4MiB).
* When the count of erased memory cells are low the memory controller will need to do the erase on-the-fly, which results in a very poor write performance.

Some flash storage drives supports notifying unused memory blocks to the controller from the Operating System to do garbage collection(GC) using the following protocol commands:

* ATA: TRIM
* NVM Express(NVMe): DEALLOCATE
* SCSI: UNMAP
* SD/MMC: ERASE/DISCARD

in order to let the storage controller do the garbage collection in the background to maintain write performance.  However, the operation may no be available if:

* It is not implemented (common in lower-end flash storage drives)
* There's a translation layer between the storage controller and the operating system, like a USB external hard-drive enclosure/cable that translates the SATA/NVMe/MMC... protocol commands into the counterpart SCSI commands which the USB mass storage/UASP specification uses.

  Another issue is that even if the translation layer support such operation, the Operating System may not utilize it due to the storage drive not complying to the relevant specifications.

## Solution

We'll need to know:

* How to check whether the storage drive supports such operation.
* If the storage drive supports such operation but the Operating System didn't enables it due to uncompliance, how to override it.
* If the storage drive claimed to support such operation, how to check whether it works or not.

The answer of these questions varies based on the following factors:

* The manufacturer and model of the storage drive/enclosure
* The firmware version of the storage drive/enclosure

We'll individually collect related info of each device to avoid bias.

## Research results

We currently have research results for the following products:

* [伽利略 M2NVU31 M.2(NVMe) PCI-E SSD to USB3.1 Gen2](<伽利略 M2NVU31 M.2(NVMe) PCI-E SSD to USB3.1 Gen2>)
* [Transcend JetFlash 790 Series 64GB](<Transcend JetFlash 790 Series 64GB>)

## References

The following material are referenced during the development of this project:

* [Discard over USB - Gentoo Wiki](https://wiki.gentoo.org/wiki/Discard_over_USB)  
  Explains how to enable discard/trim operation for block devices via the USB bus.
* [External SSD with TRIM support - Solid state drive - ArchWiki](https://wiki.archlinux.org/title/Solid_state_drive#External_SSD_with_TRIM_support)  
  Explains how to check and enable TRIM operation on solid storage drives over a USB-to-SATA bridge chips.
* [Hardware support - Trim (computing) - Wikipedia](https://en.wikipedia.org/wiki/Trim_(computing)#Hardware_support)  
  Explains the equivalent commands of unused block notification operation for each protocols.
* [sd: disable logical block provisioning if 'lbpme' is not set - Patchwork](https://patchwork.kernel.org/project/linux-scsi/patch/20190214221558.09174756@endymion/)  
  Explains the reasoning that Linux Kernel requires the LBPME bit in order to enable unused memory blocks notification support.
* [SCSI Commands Reference Manual 100293068, Rev. J](https://www.seagate.com/files/staticfiles/support/docs/manual/Interface%20manuals/100293068j.pdf)  
  Documents the Serial Attached SCSI(SAS) commands in the SCSI specifications, sections that may include useful information are:
    + 5.4 Vital product data parameters
* [Enabling TRIM on an external SSD on a Raspberry Pi | Jeff Geerling](https://www.jeffgeerling.com/blog/2020/enabling-trim-on-external-ssd-on-raspberry-pi)
* [Trying to get SSD boot working on pi4 - Page 2 - Raspberry Pi Forums](https://www.raspberrypi.org/forums/viewtopic.php?p=1708655#p1708655)
