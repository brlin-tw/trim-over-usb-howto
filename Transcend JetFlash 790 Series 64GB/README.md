# Transcend JetFlash 790 Series 64GB

A USB 3.1 consumer grade USB key by Transcend

## lsusb

```output
Bus 002 Device 005: ID 8564:1000 Transcend Information, Inc. JetFlash
```

## Block limits VPD page (SBC)

```output
# sg_vpd -p bl /dev/sdb
Block limits VPD page (SBC):
  Write same non-zero (WSNZ): 0
  Maximum compare and write length: 0 blocks [Command not implemented]
  Optimal transfer length granularity: 1 blocks
  Maximum transfer length: 65535 blocks
  Optimal transfer length: 65535 blocks
  Maximum prefetch transfer length: 65535 blocks
  Maximum unmap LBA count: 4194240
  Maximum unmap block descriptor count: 1
  Optimal unmap granularity: 1 blocks
  Unmap granularity alignment valid: false
  Unmap granularity alignment: 0 [invalid]
  Maximum write same length: 0 blocks [not reported]
  Maximum atomic transfer length: 0 blocks [not reported]
  Atomic alignment: 0 [unaligned atomic writes permitted]
  Atomic transfer length granularity: 0 [no granularity requirement
  Maximum atomic transfer length with atomic boundary: 0 blocks [not reported]
  Maximum atomic boundary size: 0 blocks [can only write atomic 1 block]
```

Relavant info:

```output
  Maximum unmap LBA count: 4194240
  Maximum unmap block descriptor count: 1
  Optimal unmap granularity: 1 blocks
```

## Logical block provisioning VPD page (SBC)

```output
# sg_vpd -p lbpv /dev/sdb
Logical block provisioning VPD page (SBC):
  Unmap command supported (LBPU): 1
  Write same (16) with unmap bit supported (LBPWS): 0
  Write same (10) with unmap bit supported (LBPWS10): 0
  Logical block provisioning read zeros (LBPRZ): 0
  Anchored LBAs supported (ANC_SUP): 0
  Threshold exponent: 0 [threshold sets not supported]
  Descriptor present (DP): 0
  Minimum percentage: 0 [not reported]
  Provisioning type: 0 (not known or fully provisioned)
  Threshold percentage: 0 [percentages not supported]
```

Relative output:

```txt
  Unmap command supported (LBPU): 1
```

## Read Capacity results

```output
$ sg_readcap -l /dev/sdb
Read Capacity results:
   Protection: prot_en=0, p_type=0, p_i_exponent=0
   Logical block provisioning: lbpme=0, lbprz=0
   Last LBA=123207679 (0x757ffff), Number of logical blocks=123207680
   Logical block length=512 bytes
   Logical blocks per physical block exponent=0
   Lowest aligned LBA=0
Hence:
   Device size: 63082332160 bytes, 60160.0 MiB, 63.08 GB
```

Relevant output:

```txt
   Logical block provisioning: lbpme=0, lbprz=0
   Logical block length=512 bytes
```

## Maximum bytes that a single SCSI UNMAP command can cover

```txt
Logical block length=512 bytes * Maximum unmap LBA count: 4194240 = 2147450880 bytes = 2047.9688 MiB ~= 2 GiB
```

## Workaround TRIM support detection

```txt
# sh -c 'echo unmap > /sys/devices/pci0000:00/0000:00:14.0/usb2/2-4/2-4:1.0/host1/target1:0:0/1:0:0:0/scsi_disk/1:0:0:0/provisioning_mode'
# bash -c 'echo $((4194240*512)) > /sys/block/sdb/queue/discard_max_bytes'
```

## Result

```txt
$ sudo blkdiscard -v /dev/sdb
/dev/sdb: Discarded 63082332160 bytes from the offset 0
[229643.067229]  sdb: sdb1 sdb2
$ sudo blkdiscard -v /dev/sdb1
/dev/sdb1: Discarded 63077613568 bytes from the offset 0
$ sudo blkdiscard -v /dev/sdb2
/dev/sdb2: Discarded 524288 bytes from the offset 0
```

* Erase commands can be run, however data isn't erased as expected
* Doesn't work even after adjusting the `discard_max_bytes` to a smaller value
