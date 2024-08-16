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

## Research process

This section documents the process of determining the support:

### Determine the kernel name of the storage device

Run the following commands in a text terminal to determine the kernel name of the storage device:

```bash
lsblk_opts=(
    # Don't print sub-device nodes
    --nodeps

    # Exclude loopback devices(e.g. snaps)
    --exclude 7

    # Specify output columns that are useful in determining the kernel
    # name of the storage device block device
    --output NAME,VENDOR,MODEL,SERIAL,SIZE
)
lsblk "${lsblk_opts[@]}"
```

<!--
According to [the command output](lsblk.out.txt) we can determine that the kernel name of the storage device is `_kernel_name_`.
-->

### Check the SCSI vital product data(VPD) pages supported by the SCSI device

Run the following command _as root_ in a text terminal to determine which SCSI vital product data(VPD) pages supported by the SCSI device:

```bash
device_kernel_name=_kernel_name_
device="/dev/${device_kernel_name}"
sg_vpd "${device}"
```

<!--
According to [the command's output](sg_vpd.out.txt) the following VPD pages are available:

* Supported VPD pages \[sv\]
* Unit serial number \[sn\]
* Device identification \[di\]
*
-->

### Check whether the storage device has declared implementation of the Logical Block Provisioning Management feature specified by SBC-4

Run the following command _as root_ in a text terminal to query the
response of the SCSI READ CAPACITY (16) command of the storage device:

```bash
device_kernel_name=_kernel_name_
device="/dev/${device_kernel_name}"
sg_readcap_opts=(
    # Use the 16 byte cdb variant of the READ CAPACITY command, allow
    # proper results for storage drives capacity over 2TiB
    # (2**32 - 2) * 512 / 1024 / 1024 / 1024
    --long
)
sg_readcap "${sg_readcap_opts[@]}" "${device}"
```

<!--
According to [the command's output](sg_readcap-long.out.txt) we can
verify that the storage drive _claims_ that it does not support the
Logical Block Provisioning Management feature specified by SBC-4,
which _contradicts_ with the support of the UNMAP command in the
previous steps:

```txt
Read Capacity results:
   Protection: prot_en=0, p_type=0, p_i_exponent=0
   Logical block provisioning: lbpme=0, lbprz=0

    ...stripped...

```
-->

### Check the supported parameters of the SCSI UNMAP command of the storage device

<!--
Let's temporarily disregard the fact that the storage device have
claimed it do not have Logical Block Provisioning Management and
assume that it _do_ in fact support the UNMAP SCSI command, let's check
whether there's any limitations that needs to take note of during the
usage of UNMAP SCSI commands.
-->

Run the following command _as root_ in a text terminal to query the
Block limits VPD page of the storage device:

```bash
device_kernel_name=_kernel_name_
device="/dev/${device_kernel_name}"
sg_vpd_opts=(
    --page=bl
)
sg_vpd "${sg_vpd_opts[@]}" "${device}"
```

<!--
According to [the command's output](sg_vpd-bl.out.txt) we can verify
that the storage drive _claim_ it can notify _block_quantity_ unused logical
blocks in a single SCSI UNMAP command:

```txt
Block limits VPD page (SBC):

    ...stripped...

  Maximum unmap LBA count: _block_quantity_
```
-->

<!--
What is the size of a logical block anyway, let's check [the previous
output of the `sg_readcap --long` command](sg_readcap-long.out.txt) for
that:

```text
Read Capacity results:

    ...stripped...

   Logical block length=512 bytes
```

So the size of a logical block is 512 bytes, after the following
calculation and unit conversion we can conclude that the storage
device can accept _batch_size_ bytes(_batch_size_human_readable_ to be exact)
of unused blocks notification in a single UNMAP SCSI command:

$$
\begin{align}
Notifiable\ unused\ memory\ size\ per\ UNMAP\ command &= _block_quantity_\ blocks \times 512\ B/block \div 1024\ KiB/B \div 1024\ MiB/KiB \div 1024\ GiB/MiB \\
&\approx _batch_size_human_readable_ GiB
\end{align}
$$

At least in the assumption that the storage device did announce it
correctly(which we already know, it didn't in some places).
-->

### Enable unused block notification support by force

Although the missing declaration of the SCSI Logical Block Provisioning
Management feature, there's still possibility that the drive actually
implemented the unused block notification functionality in the firmware.
Let's try to find out.

**Warning:** There's a reason why the functionality is not enabled by
default as there's a chance that the controller have a problomatic
reaction when facing the UNMAP SCSI command, which may results in
problems including but not limited to:

* The drive simply fails and no longer functions properly, rendering
  it no longer usable for data storage/access.
* The drive erratically respond to the command and erases memory blocks
  that are not requested, leads to data loss.

**Only continue if you can take the responsibility of device
failure/data recovery.**

In order to force enable the unused block notification feature, we need
to first determine the address of the SCSI device of the storage device
, which can be queried by running the following command in a text
terminal:

```bash
lsscsi
```

<!--
From [the command's output](lsscsi.out.txt) we can determine that the
address of the SCSI device is `0:0:0:0`.
-->

We can set the max bytes of unused data to notify in a single UNMAP SCSI command by running the following commands _as root_ after setting the proper `scsi_device_address`:

```bash
scsi_device_address=_address_
echo unmap > "/sys/class/scsi_disk/${scsi_device_address}/provisioning_mode"
```

Now that the UNMAP command is enabled, run the following commands to check whether the data size of each UNMAP SCSI command set by the kernel is sane:

```bash
device_kernel_name=_kernel_name_
cat "/sys/block/${device_kernel_name}/queue/discard_max_bytes"
```

<!--
From [the command's output](sysfs-block-queue-discard_max_bytes-after-overriding-provision_mode.out.txt)
you can notice that the system detected that you can notify 4,294,966,784
bytes(=4GiB - 512bytes) of unused data in a single UNMAP SCSI command,
**which contradicts with the _batch_size_ bytes limitation we
previously determined**.

Let's set it to the proper value by running the following command _as
root_:

```bash
device_kernel_name=_kernel_name_
echo _batch_size_ > "/sys/block/${device_kernel_name}/queue/discard_max_bytes"
```
-->

### Test whether unused block notification actually work

Now we can test whether the unused block notification really work by
triggering a whole drive/partition block device discard operation by
running the following commands _as root_:

```bash
device_kernel_name=_kernel_name_
device="/dev/${device_kernel_name}"
blkdiscard_opts=(
    # Disable safeguard checks
    --force

    # Print details of the operation
    --verbose
)
blkdiscard "${blkdiscard_opts[@]}" "${device}"
```

<!--
You should see the following output, indicate that the discard
operation is a success:

```txt
blkdiscard: Operation forced, data will be lost!
/dev/_kernel_name_: Discarded _storage_size_ bytes from the offset 0
```

However, if you check the binary content of the storage drive you'll
find that the drive's content isn't erased at all:

```txt
device_kernel_name=_kernel_name_
device="/dev/${device_kernel_name}"
$ sudo xxd -l 128 "${device}"
00000000: eb63 9000 0000 0000 0000 0000 0000 0000  .c..............
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0080 0100 0000  ................
00000060: 0000 0000 fffa 9090 f6c2 8074 05f6 c270  ...........t...p
00000070: 7402 b280 ea79 7c00 0031 c08e d88e d0bc  t....y|..1......
```

Therefore the unused block notification does not work at all in this
storage device, even when the device has claimed support for the
UNMAP SCSI command.
-->

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
* The lsblk(8) manual page  
  Explains how to use the `--nodeps` and `--output` command-line options.
* The sg_vpd(8) manual page  
  Explains how to use the `--long` `sg_vpd` command-line option.
