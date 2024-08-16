# Transcend JetFlash 790 Series 64GB

A USB 3.1 consumer grade USB key by Transcend

## Identification

### USB ID

According to [the output of the `lsusb` command](lsusb.out.txt):

**Vendor ID:** `8564`  
**Product ID:** `1000`

## Verdict

Despite that the device has advertised SCSI UNMAP command support, it does not complied the SCSI standard by advertising the LBPME bit in the reponse of the READ CAPACITY SCSI command and therefore, unused block notification support will not be enabled on GNU/Linux system by default.

After further investigation we found that the storage device cannot properly process the UNMAP command that it claimed to support, thus therefore, cannot support notification of unused memory blocks in the first place.

This vertict is based on the 0x1100(1.1?) firmware revision of the storage device, result may vary on different firmware revision/product SKUs.

## Process of determining unused blocks notification support

### Determine the block device path of the storage device

Run the following commands in a text terminal to determine the kernel block device name of the storage device:

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

### Check the SCSI vital product data(VPD) pages supported by the SCSI device

Run the following command _as root_ in a text terminal to determine which SCSI vital product data(VPD) pages supported by the SCSI device:

```bash
sg_vpd /dev/sda
```

According to [the command's output](sg_vpd.out.txt) the following VPD pages are available:

* Supported VPD pages \[sv\]
* Unit serial number \[sn\]
* Device identification \[di\]
* Block limits (SBC) \[bl\]
* Logical block provisioning (SBC) \[lbpv\]

### Check whether the storage device supports the SCSI UNMAP command

Run the following command _as root_ in a text terminal to query the
Logical block provisioning VPD page of the storage device:

```command
sg_vpd_opts=(
    --page=lbpv
)
sg_vpd "${sg_vpd_opts[@]}" /dev/sda
```

According to [the command's output](sg_vpd-lbpv.out.txt) we can verify
that the storage drive _claim_ it has UNMAP SCSI command support:

```output
  Unmap command supported (LBPU): 1
```

### Check whether the storage device has declared implementation of the Logical Block Provisioning Management feature specified by SBC-4

Run the following command _as root_ in a text terminal to query the
response of the SCSI READ CAPACITY (16) command of the storage device:

```output
sg_readcap_opts=(
    # Use the 16 byte cdb variant of the READ CAPACITY command, allow
    # proper results for storage drives capacity over 2TiB
    # (2**32 - 2) * 512 / 1024 / 1024 / 1024
    --long
)
sg_readcap "${sg_readcap_opts[@]}" /dev/sda
```

According to [the command's output](sg_readcap-long.out.txt) we can
verify that the storage drive _claims_ that it does not support the
Logical Block Provisioning Management feature specified by SBC-4,
which _contradicts_ with the support of the UNMAP command in the
previous steps.

### Check the supported parameters of the SCSI UNMAP command of the storage device

Let's temporarily disregard the fact that the storage device have
claimed it do not have Logical Block Provisioning Management and
assume that it _do_ in fact support the UNMAP SCSI command, let's check
whether there's any limitations that needs to take note of during the
usage of UNMAP SCSI commands.

Run the following command _as root_ in a text terminal to query the
Block limits VPD page of the storage device:

```command
sg_vpd_opts=(
    --page=bl
)
sg_vpd "${sg_vpd_opts[@]}" /dev/sda
```

According to [the command's output](sg_vpd-bl.out.txt) we can verify
that the storage drive _claim_ it can notify 4,194,240 unused logical
blocks in a single SCSI UNMAP command:

```txt
Block limits VPD page (SBC):

    ...stripped...

  Maximum unmap LBA count: 4194240
```

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
device can accept 2,147,450,880 bytes(2GiB - 32KiB to be exact)
of unused blocks notification in a single UNMAP SCSI command:

$$
\begin{align}
Notifiable\ unused\ memory\ size\ per\ UNMAP\ command &= 4,194,240\ blocks \times 512\ B/block \div 1024\ KiB/B \div 1024\ MiB/KiB \div 1024\ GiB/MiB \\
&\approx 1.99996... GiB
\end{align}
$$

At least in the assumption that the storage device did announce it
correctly(which we already know, it didn't in some places).

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

From [the command's output](lsscsi.out.txt) we can determine that the
address of the SCSI device is `0:0:0:0`.

We can set the max bytes of unused data to notify in a single UNMAP SCSI command by running the following commands _as root_ after setting the proper `scsi_device_address`:

```bash
scsi_device_address=0:0:0:0
echo unmap > "/sys/class/scsi_disk/${scsi_device_address}/provisioning_mode"
```

Now that the UNMAP command is enabled, run the following commands to check whether the data size of each UNMAP SCSI command set by the kernel is sane:

```bash
device_name=sda
cat "/sys/block/${device_name}/queue/discard_max_bytes"
```

From [the command's output](sysfs-block-queue-discard_max_bytes-after-overriding-provision_mode.out.txt)
you can notice that the system detected that you can notify 4,294,966,784
bytes(=4GiB - 512bytes) of unused data in a single UNMAP SCSI command,
**which contradicts with the 2,147,450,880 bytes limitation we
previously determined**.

Let's set it to the proper value by running the following command _as
root_:

```bash
device_name=sda
echo 2147450880 > "/sys/block/${device_name}/queue/discard_max_bytes"
```

### Test whether unused block notification actually work

Now we can test whether the unused block notification really work by
triggering a whole drive/partition block device discard operation by
running the following commands _as root_:

```bash
device_name=sda
blkdiscard_opts=(
    # Disable safeguard checks
    --force

    # Print details of the operation
    --verbose
)
blkdiscard "${blkdiscard_opts[@]}" "/dev/${device_name}"
```

You should see the following output, indicate that the discard
operation is a success:

```txt
blkdiscard: Operation forced, data will be lost!
/dev/sda: Discarded 63082332160 bytes from the offset 0
```

However, if you check the binary content of the storage drive you'll
find that the drive's content isn't erased at all:

```txt
$ sudo xxd -l 128 /dev/sda
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

## References

The following materials are referenced during the development of this research:

* [‎Gemini - LBP in SCSI](https://gemini.google.com/share/e90021aaed7c)  
  Introduces what Logical Block Provisioning Management (LBP) in SCSI do.
* [SCSI device location codes - IBM Documentation](https://www.ibm.com/docs/en/aix/7.3?topic=codes-scsi-device-location)  
  Explains the format of the SCSI device location code.
* [SCSI Addressing](https://tldp.org/HOWTO/SCSI-2.4-HOWTO/scsiaddr.html)  
  Explains the format of the SCSI device addresses.
* [Queue sysfs files | Info on the Block I/O (BIO) layer | Linux kernel plaintext documentation](https://www.kernel.org/doc/Documentation/block/queue-sysfs.txt)  
  Explains the definition of the discard_max_bytes sysfs file.
