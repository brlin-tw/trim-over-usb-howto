# 伽利略 M2NVU31 M.2(NVMe) PCI-E SSD to USB3.1 Gen2

A consumer grade NVMe storge external closure

## Problems

* Only negotiated 5Gbps bandwidth instead of 10Gbps as announced
* No native TRIM support under GNU+Linux

## lsusb

```output
$ lsusb -s 002:003
Bus 002 Device 003: ID 152d:0562 JMicron Technology Corp. / JMicron USA Technology Corp. JMS567 SATA 6Gb/s bridge
```

Despite what the usb-ids database have claimed, the chip is actually JMS583 USB 3.1 Gen 2 to PCIe Gen3x2 Bridge Controller, as engraved on the chip.

```output
# lsusb -s 002:003 -vvv
Bus 002 Device 003: ID 152d:0562 JMicron Technology Corp. / JMicron USA Technology Corp. JMS567 SATA 6Gb/s bridge
Device Descriptor:
  bLength                18
  bDescriptorType         1
  bcdUSB               3.20
  bDeviceClass            0 
  bDeviceSubClass         0 
  bDeviceProtocol         0 
  bMaxPacketSize0         9
  idVendor           0x152d JMicron Technology Corp. / JMicron USA Technology Corp.
  idProduct          0x0562 JMS567 SATA 6Gb/s bridge
  bcdDevice            2.05
  iManufacturer           1 JMicron
  iProduct                2 External
  iSerial                 3 DD56419883898
  bNumConfigurations      1
  Configuration Descriptor:
    bLength                 9
    bDescriptorType         2
    wTotalLength       0x0079
    bNumInterfaces          1
    bConfigurationValue     1
    iConfiguration          0 
    bmAttributes         0x80
      (Bus Powered)
    MaxPower              896mA
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       0
      bNumEndpoints           2
      bInterfaceClass         8 Mass Storage
      bInterfaceSubClass      6 SCSI
      bInterfaceProtocol     80 Bulk-Only
      iInterface              0 
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x81  EP 1 IN
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst              15
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x02  EP 2 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst              15
    Interface Descriptor:
      bLength                 9
      bDescriptorType         4
      bInterfaceNumber        0
      bAlternateSetting       1
      bNumEndpoints           4
      bInterfaceClass         8 Mass Storage
      bInterfaceSubClass      6 SCSI
      bInterfaceProtocol     98 
      iInterface             10 MSC USB Attached SCSI
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x01  EP 1 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst               0
        Command pipe (0x01)
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x82  EP 2 IN
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst               0
        MaxStreams             32
        Status pipe (0x02)
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x83  EP 3 IN
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst              15
        MaxStreams             32
        Data-in pipe (0x03)
      Endpoint Descriptor:
        bLength                 7
        bDescriptorType         5
        bEndpointAddress     0x04  EP 4 OUT
        bmAttributes            2
          Transfer Type            Bulk
          Synch Type               None
          Usage Type               Data
        wMaxPacketSize     0x0400  1x 1024 bytes
        bInterval               0
        bMaxBurst              15
        MaxStreams             32
        Data-out pipe (0x04)
Binary Object Store Descriptor:
  bLength                 5
  bDescriptorType        15
  wTotalLength       0x002a
  bNumDeviceCaps          3
  USB 2.0 Extension Device Capability:
    bLength                 7
    bDescriptorType        16
    bDevCapabilityType      2
    bmAttributes   0x00000f0e
      BESL Link Power Management (LPM) Supported
    BESL value     3840 us 
  SuperSpeed USB Device Capability:
    bLength                10
    bDescriptorType        16
    bDevCapabilityType      3
    bmAttributes         0x00
    wSpeedsSupported   0x000e
      Device can operate at Full Speed (12Mbps)
      Device can operate at High Speed (480Mbps)
      Device can operate at SuperSpeed (5Gbps)
    bFunctionalitySupport   1
      Lowest fully-functional device speed is Full Speed (12Mbps)
    bU1DevExitLat          10 micro seconds
    bU2DevExitLat          32 micro seconds
  SuperSpeedPlus USB Device Capability:
    bLength                20
    bDescriptorType        16
    bDevCapabilityType     10
    bmAttributes         0x00000001
      Sublink Speed Attribute count 1
      Sublink Speed ID count 0
    wFunctionalitySupport   0x1100
    bmSublinkSpeedAttr[0]   0x000a4030
      Speed Attribute ID: 0 10Gb/s Symmetric RX SuperSpeedPlus
    bmSublinkSpeedAttr[1]   0x000a40b0
      Speed Attribute ID: 0 10Gb/s Symmetric TX SuperSpeedPlus
Device Status:     0x000c
  (Bus Powered)
  U1 Enabled
  U2 Enabled
```

## lsblk

```terminal
$ lsblk -D /dev/sdb
NAME   DISC-ALN DISC-GRAN DISC-MAX DISC-ZERO
sdb           0        0B       0B         0
├─sdb1        0        0B       0B         0
├─sdb2        0        0B       0B         0
├─sdb3        0        0B       0B         0
├─sdb4        0        0B       0B         0
└─sdb5        0        0B       0B         0
```

No TRIM support by default

## smartctl

```
sudo smartctl -a -d sntjmicron /dev/sdb
smartctl 7.2 2020-12-30 r5155 [x86_64-linux-5.11.0-16-generic] (local build)
Copyright (C) 2002-20, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Model Number:                       WDC WDS500G1B0C-00S6U0
Serial Number:                      1935AB800964
Firmware Version:                   201000WD
PCI Vendor/Subsystem ID:            0x15b7
IEEE OUI Identifier:                0x001b44
Total NVM Capacity:                 500,107,862,016 [500 GB]
Unallocated NVM Capacity:           0
Controller ID:                      1
NVMe Version:                       1.3
Number of Namespaces:               1
Namespace 1 Size/Capacity:          500,107,862,016 [500 GB]
Namespace 1 Formatted LBA Size:     512
Namespace 1 IEEE EUI-64:            001b44 8b442efe7c
Local Time is:                      Mon May  3 01:11:02 2021 CST
Firmware Updates (0x14):            2 Slots, no Reset required
Optional Admin Commands (0x0017):   Security Format Frmw_DL Self_Test
Optional NVM Commands (0x001f):     Comp Wr_Unc DS_Mngmt Wr_Zero Sav/Sel_Feat
Log Page Attributes (0x02):         Cmd_Eff_Lg
Maximum Data Transfer Size:         128 Pages
Warning  Comp. Temp. Threshold:     82 Celsius
Critical Comp. Temp. Threshold:     86 Celsius
Namespace 1 Features (0x02):        NA_Fields

Supported Power States
St Op     Max   Active     Idle   RL RT WL WT  Ent_Lat  Ex_Lat
 0 +     2.60W       -        -    0  0  0  0        0       0
 1 +     2.60W       -        -    1  1  1  1        0       0
 2 +     1.70W       -        -    2  2  2  2        0       0
 3 -   0.0250W       -        -    3  3  3  3     5000    9000
 4 -   0.0025W       -        -    4  4  4  4     5000   44000

Supported LBA Sizes (NSID 0x1)
Id Fmt  Data  Metadt  Rel_Perf
 0 +     512       0         2
 1 -    4096       0         1

=== START OF SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

SMART/Health Information (NVMe Log 0x02)
Critical Warning:                   0x00
Temperature:                        53 Celsius
Available Spare:                    100%
Available Spare Threshold:          10%
Percentage Used:                    0%
Data Units Read:                    335,657 [171 GB]
Data Units Written:                 569,044 [291 GB]
Host Read Commands:                 6,879,121
Host Write Commands:                9,763,664
Controller Busy Time:               6
Power Cycles:                       68
Power On Hours:                     329
Unsafe Shutdowns:                   41
Media and Data Integrity Errors:    0
Error Information Log Entries:      0
Warning  Comp. Temperature Time:    0
Critical Comp. Temperature Time:    0

Error Information (NVMe Log 0x01, 16 of 256 entries)
No Errors Logged
```

## SCSI supported VPD pages

```output
# sg_vpd /dev/sdb
Supported VPD pages VPD page:
  Supported VPD pages [sv]
  Unit serial number [sn]
  Device identification [di]
  Block limits (SBC) [bl]
  Block device characteristics (SBC) [bdc]
  Logical block provisioning (SBC) [lbpv]
  0xde
  0xdf
```

## SCSI standard inquiry

```output
# sg_vpd -p sinq /dev/sdb
standard INQUIRY:
  PQual=0  Device_type=0  RMB=0  LU_CONG=0  version=0x06  [SPC-4]
  [AERC=0]  [TrmTsk=0]  NormACA=0  HiSUP=1  Resp_data_format=2
  SCCS=0  ACC=0  TPGS=0  3PC=0  Protect=0  [BQue=0]
  EncServ=0  MultiP=0  [MChngr=0]  [ACKREQQ=0]  Addr16=0
  [RelAdr=0]  WBus16=0  Sync=0  [Linked=0]  [TranDis=0]  CmdQue=1
  Vendor_identification: JMicron 
  Product_identification: Tech            
  Product_revision_level: 0205
```

## SCSI all VPD pages dump

```
# sg_vpd -a /dev/sdb
Supported VPD pages VPD page:
  Supported VPD pages [sv]
  Unit serial number [sn]
  Device identification [di]
  Block limits (SBC) [bl]
  Block device characteristics (SBC) [bdc]
  Logical block provisioning (SBC) [lbpv]
  0xde
  0xdf

Unit serial number VPD page:
  Unit serial number: DD56419883898

Device Identification VPD page:
  Addressed logical unit:
    designator type: NAA,  code set: Binary
      0x3044564198838980
    designator type: T10 vendor identification,  code set: ASCII
      vendor id: JMicron 
      vendor specific: Tech            DD56419883898

Block limits VPD page (SBC):
  Write same non-zero (WSNZ): 0
  Maximum compare and write length: 0 blocks [Command not implemented]
  Optimal transfer length granularity: 8 blocks
  Maximum transfer length: 65535 blocks
  Optimal transfer length: 65535 blocks
  Maximum prefetch transfer length: 65535 blocks
  Maximum unmap LBA count: -1 [unbounded]
  Maximum unmap block descriptor count: 63
  Optimal unmap granularity: 0 blocks [not reported]
  Unmap granularity alignment valid: false
  Unmap granularity alignment: 0 [invalid]
  Maximum write same length: 0 blocks [not reported]
  Maximum atomic transfer length: 0 blocks [not reported]
  Atomic alignment: 0 [unaligned atomic writes permitted]
  Atomic transfer length granularity: 0 [no granularity requirement
  Maximum atomic transfer length with atomic boundary: 0 blocks [not reported]
  Maximum atomic boundary size: 0 blocks [can only write atomic 1 block]

Block device characteristics VPD page (SBC):
  Non-rotating medium (e.g. solid state)
  Product type: Not specified
  WABEREQ=0
  WACEREQ=0
  Nominal form factor not reported
  ZONED=0
  RBWZ=0
  BOCS=0
  FUAB=0
  VBULS=0
  DEPOPULATION_TIME=0 (seconds)

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

Only hex output supported
VPD page code=0xde:
 00     00 de 00 0c 4a 4d 00 00  00 00 00 00 00 00 00 00    ....JM..........

Only hex output supported
VPD page code=0xdf:
 00     00 df 00 11 4a 4d 00 00  00 00 00 00 00 00 00 00    ....JM..........
 10     00 00 00 00 00                                      .....
```

## Block limits VPD page (SBC)

```output
# sg_vpd -p bl /dev/sdb
Block limits VPD page (SBC):
  Write same non-zero (WSNZ): 0
  Maximum compare and write length: 0 blocks [Command not implemented]
  Optimal transfer length granularity: 8 blocks
  Maximum transfer length: 65535 blocks
  Optimal transfer length: 65535 blocks
  Maximum prefetch transfer length: 65535 blocks
  Maximum unmap LBA count: -1 [unbounded]
  Maximum unmap block descriptor count: 63
  Optimal unmap granularity: 0 blocks [not reported]
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
  Maximum unmap LBA count: -1 [unbounded]
  Maximum unmap block descriptor count: 63
  Optimal unmap granularity: 0 blocks [not reported]
  Unmap granularity alignment valid: false
  Unmap granularity alignment: 0 [invalid]
```

...unbounded `Maximum unmap LBA count`?

## SCSI Logical block provisioning (SBC)

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

```output
  Unmap command supported (LBPU): 1
```

## SCSI Read Capacity results

```
$ sg_readcap -l /dev/sdb
Read Capacity results:
   Protection: prot_en=0, p_type=0, p_i_exponent=0
   Logical block provisioning: lbpme=0, lbprz=0
   Last LBA=976773167 (0x3a38602f), Number of logical blocks=976773168
   Logical block length=512 bytes
   Logical blocks per physical block exponent=3 [so physical block length=4096 bytes]
   Lowest aligned LBA=0
Hence:
   Device size: 500107862016 bytes, 476940.0 MiB, 500.11 GB
```

Relevant output:

```
   Logical block provisioning: lbpme=0, lbprz=0
```

According to ref#1, Linux kernel will not read UNMAP related VPDs (hence not enabling TRIM support) when LBPME bit is not set, the kernel developer indicate that the bridge chip/firmware manufacturer does not comply to the SCSI spec and presented wrong information to the OS:

> We don't skip the entire VPD page if LBPME=1, just the parts that are
> related to the logical block provisioning.
> 
> The reason we read those values in the first place is so that we can set
up discard. If the device signals that it does not support discard we
have had no reason to read them or parse them.
> 
> Since there are a plethora of USB-SATA devices out there that get this
> incredibly wrong (including discarding blocks *outside* of the specified
> block range), I am not going to blindly enable this feature.
> 
> If you want to tinker with your own setup that's fine. And you are
> clearly capable of doing so.
> 
> I, however, have to be extremely cautious not to enable something that
> might inadvertently cause data corruption for users out there. That's
> all I care about.
> 
> If the device manufacturer had intended to support discards then they
> would presumably have set the LBPME flag per the spec. And if they
> intended to support it but messed up setting the single bit flag that
> enables the feature, then I would not trust their implementation.

## Maximum bytes that a single SCSI UNMAP command can cover

Unknown, since `Logical block length=512 bytes` and `Maximum unmap LBA count: -1 [unbounded]`'s product isn't countable.

However, after applying the [workaround](#workarounds), `lsblk -D` shows that it can discard 4GiB of blocks at once:

```
$ lsblk -D /dev/sdb
NAME   DISC-ALN DISC-GRAN DISC-MAX DISC-ZERO
sdb           0        4K       4G         0
├─sdb1     3072        4K       4G         0
├─sdb2        0        4K       4G         0
├─sdb3        0        4K       4G         0
├─sdb4        0        4K       4G         0
└─sdb5        0        4K       4G         0
```

Whether how this is calculated by the system is unknown.

## Conclusion

TRIM not natively supported by GNU+Linux due to the LBPME bit set to `0` instead of `1`, which is not conforming to the 4.7.3.2 section of the SCSI SBC-5 specification for devices that are classified as a resource provisioned logical unit(Solid-State Drives).

Which is really unfortunate for an exclosure specifically for housing NVMe SSDs.

## Workarounds

The following ugly udev configuration must be applied to enable TRIM:

```udev
# 99-workaround-buggy-usb-storage-not-supporting-trim.rules
ACTION=="add|change", ATTRS{idVendor}=="152d", ATTRS{idProduct}=="0562", SUBSYSTEM=="scsi_disk", ATTR{provisioning_mode}="unmap"
```

## Reference

1. [[1/1] sd: do not let LBPME bit stop the VPDs speak - Patchwork](https://patchwork.kernel.org/project/linux-scsi/patch/56df97e0.d08d420a.58e8f.3162@mx.google.com/)
1. sntjmicron[,NSID] | -d TYPE, --device=TYPE | RUN-TIME BEHAVIOR OPTIONS: | OPTIONS | smartctl(8) manpage
1. [Will Windows do unmap on USB drives in any case?](https://social.technet.microsoft.com/Forums/security/en-US/2cfc8c18-57ed-435d-a648-049cdda329bf/will-windows-do-unmap-on-usb-drives-in-any-case?forum=win10itprohardware)
1. [200679 – broken TRIM support for JMS578 in uas mode](https://bugzilla.kernel.org/show_bug.cgi?id=200679)
1. [sd: disable logical block provisioning if 'lbpme' is not set - Patchwork](https://patchwork.kernel.org/project/linux-scsi/patch/20190214221558.09174756@endymion/)
1. [The road for thin-provisioning](https://www.linux-kvm.org/images/7/77/2012-forum-thin-provisioning.pdf) by Paolo Bonzini(Red Hat, Inc.)
1. [debian - WRITE SAME manually zero - Unix & Linux Stack Exchange](https://unix.stackexchange.com/questions/244147/write-same-manually-zero)
1. [Thin Provisioning - Windows drivers | Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/storage/thin-provisioning)
1. [Notes for Linux SCSI logical block provisioning](https://gist.github.com/cathay4t/e80e02a737242a5f3824606543631bfe)
1. [Logical block provisioning | Direct access block device type model | Information technology -SCSI Block Commands – 5 (SBC-5) | T10](https://www.t10.org/cgi-bin/ac.pl?t=f&f=sbc5r00.pdf)
