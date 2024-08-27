# OWC USB-C Travel Dock E(SD card reader)

The SD card reader of [the OWC USB-C Travel Dock E dock](https://www.owc.com/solutions/usb-c-travel-dock-e).

## Table of contents

[TOC]

## Identification

### USB

According to [the output of the `lsusb` command](lsusb.out.txt):

**Vendor ID:** `0BDA`  
**Product ID:** `0329`  
**Device version:** `29.1.2`

## Variables

This section documents the variables that may affect the analysis
results:

### SD card for testing

Sandisk Ultra 32GB micro SD card(SDHC/UHS-I/Class 10)  
Supports ERASE command, optimal erase size 4MiB

**OEM ID:** `0x5344`  
**Manufacturer ID:** `0x000003`  
**Manufacture date:** `05/2021`  
**Name:** `SC32G`  
**Hardware revision:** `0x8`  
**Firmware revision:** `0x0`

```txt
brlin@brlin-mz-530-gx:~$ sudo blkdiscard -fv /dev/mmcblk0
blkdiscard: Operation forced, data will be lost!
/dev/mmcblk0: Discarded 31914983424 bytes from the offset 0
brlin@brlin-mz-530-gx:~$ sudo xxd -l 128 "${device}"
00000000: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000070: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

## Verdict

This product, at least for the firmware version it is tested on, doesn't
support receiving unused block notification from the operating system.

## Analysis

This section documents the process of determining the support:

### Determine the block device path of the storage device

Run the following command in a text terminal to determine the kernel
block device name of the storage device:

```bash
lsblk_opts=(
    # Don't print sub-device nodes
    --nodeps

    # Exclude loopback devices(e.g. snaps)
    --exclude 7

    # Specify output columns that are useful in determining the kernel block device name
    --output NAME,VENDOR,MODEL,SERIAL,SIZE
)
lsblk "${lsblk_opts[@]}"
```

According to [the command output](lsblk.out.txt) we can determine that the kernel block device name is `sda`.

### Determine whether unused block notification support is enabled by the operating system by default

Run the following commands in a text terminal to determine the kernel name of the storage device:

```bash
device_kernel_name=sda
device="/dev/${device_kernel_name}"
lsblk_opts=(
    # Don't print sub-device nodes
    --nodeps

    # Print information about the unused block notification capabilities
    # for each device.
    --discard
)
lsblk "${lsblk_opts[@]}" "${device}"
```

According to [the command output](lsblk-discard-native.out.txt) we can
determine that the operating system didn't enable the unused block
notification feature for this storage device.

### Check whether the storage device has declared implementation of the Logical Block Provisioning Management feature specified by SBC-4

Run the following command _as root_ in a text terminal to query the
response of the SCSI READ CAPACITY (16) command of the storage device:

```bash
device_kernel_name=sda
device="/dev/${device_kernel_name}"
sg_readcap_opts=(
    # Use the 16 byte cdb variant of the READ CAPACITY command, allow
    # proper results for storage drives capacity over 2TiB
    # (2**32 - 2) * 512 / 1024 / 1024 / 1024
    --long
)
sg_readcap "${sg_readcap_opts[@]}" "${device}"
```

According to [the command's output](sg_readcap-long.out.txt) we can
verify that the storage drive _claims_ that it does not support the
Logical Block Provisioning Management feature specified by SBC-4,
which indicates that it doesn't have support of the UNMAP command:

```txt
Read Capacity results:
   Protection: prot_en=0, p_type=0, p_i_exponent=0
   Logical block provisioning: lbpme=0, lbprz=0

    ...stripped...

```

### Check the SCSI vital product data(VPD) pages supported by the SCSI device

Now we need to check whether the storage device have claimed support
of the UNMAP command via other means.  Run the following command
_as root_ in a text terminal to determine which SCSI vital product
data(VPD) pages supported by the SCSI device:

```bash
device_kernel_name=sda
device="/dev/${device_kernel_name}"
sg_vpd "${device}"
```

According to [the command's output](sg_vpd.out.txt) we can find out
that the storage device only supports the following VPD pages:

* 0x00  Supported VPD pages [sv]
* 0x80  Unit serial number [sn]
* 0x83  Device identification [di]

As neither of the following VPD pages are supported, we unable to
determine UNMAP command support via what the storage device have
advertised to the system:

* Block limits
* Logical block provisioning

### Enable unused block notification support by force

Although neither the SCSI Logical Block Provisioning Management feature
nor the UNMAP SCSI command is claimed to be supported by the storage
device, there's still possibility that the drive actually implemented
the unused block notification functionality, let's try to find out.

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

From [the command's output](lsscsi.out.txt) we can determine that the
address of the SCSI device is `0:0:0:0`.

We can force enable the unused block notification support feature using
the UNMAP SCSI command by running the following commands _as root_ after
setting the proper `scsi_device_address`:

```bash
scsi_device_address=0:0:0:0
echo unmap > "/sys/class/scsi_disk/${scsi_device_address}/provisioning_mode"
```

### Test whether unused block notification actually work

Now we can test whether the unused block notification really work by
triggering a whole drive/partition block device discard operation by
running the following commands _as root_:

```bash
device_kernel_name=sda
device="/dev/${device_kernel_name}"
blkdiscard_opts=(
    # Disable safeguard checks
    --force

    # Print details of the operation
    --verbose
)
blkdiscard "${blkdiscard_opts[@]}" "${device}"
```

You should see the following output, indicate that the discard
operation seems to be a success:

```txt
blkdiscard: Operation forced, data will be lost!
/dev/sda: Discarded 31914983424 bytes from the offset 0
```

However, if you check the binary content of the storage drive you'll
find that the drive's content isn't erased at all:

```txt
device_kernel_name=mmcblk0
device="/dev/${device_kernel_name}"
sudo xxd -l 128 "${device}"
00000000: 474e 5520 5061 7274 6564 204c 6f6f 7062  GNU Parted Loopb
00000010: 6163 6b20 3000 0000 3f00 ff00 0000 0000  ack 0...?.......
00000020: 0024 b703 dc0e 0000 0000 0000 0200 0000  .$..............
00000030: 0100 0600 0000 0000 0000 0000 0000 0000  ................
00000040: 0000 2967 4523 0142 524c 494e 2d43 414d  ..)gE#.BRLIN-CAM
00000050: 3332 4641 5433 3220 2020 0000 0000 0000  32FAT32   ......
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000070: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

Also if you check the kernel log you'll find illegal request error
messages regarding the `blkdiscard` attempt we just done:

```txt
kernel: sd 0:0:0:0: [sda] tag#0 FAILED Result: hostbyte=DID_OK driverbyte=DRIVER_OK cmd_age=0s
kernel: sd 0:0:0:0: [sda] tag#0 Sense Key : Illegal Request [current]
kernel: sd 0:0:0:0: [sda] tag#0 Add. Sense: Invalid field in cdb
kernel: sd 0:0:0:0: [sda] tag#0 CDB: Unmap/Read sub-channel 42 00 00 00 00 00 00 00 18 00
kernel: critical target error, dev sda, sector 0 op 0x3:(DISCARD) flags 0x4000 phys_seg 1 prio class 0
kernel: critical target error, dev sda, sector 8388607 op 0x3:(DISCARD) flags 0x0 phys_seg 1 prio class 0
kernel:  sda: sda1
```

Therefore the unused block notification does not work at all in this
storage device.

## Additional info

The storage drive's Device Identification VPD page [contains data that can't be fully parsed](sg_vpd-di.out.txt)([binary](sg_vpd-di.out.bin)).

## References

* [OWC USB-C Travel Dock E](https://www.owc.com/solutions/usb-c-travel-dock-e)  
  The product page from the manufacturer website.
