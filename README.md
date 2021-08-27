# wyzecamv3 Telnet enabler

## Analyzing the `.bin`

By running `binwalk -t demo_wcv3.bin` we can figure out a couple of things.
```
 
DECIMAL       HEXADECIMAL     DESCRIPTION
---------------------------------------------------------------------------------------------------------------------
0             0x0             uImage header, header size: 64 bytes, header CRC: 0xECCED73E, created: 2021-06-11
                              12:44:24, image size: 9555968 bytes, Data Address: 0x0, Entry Point: 0x0, data CRC:
                              0x49FA1464, OS: Linux, CPU: MIPS, image type: Firmware Image, compression type: none,
                              image name: "jz_fw"
64            0x40            uImage header, header size: 64 bytes, header CRC: 0x5AF494C9, created: 2021-04-14
                              09:44:55, image size: 1796125 bytes, Data Address: 0x80010000, Entry Point:
                              0x803DEE20, data CRC: 0xFA3AFB78, OS: Linux, CPU: MIPS, image type: OS Kernel Image,
                              compression type: lzma, image name: "Linux-3.10.14__isvp_swan_1.0__"
128           0x80            LZMA compressed data, properties: 0x5D, dictionary size: 67108864 bytes, uncompressed
                              size: -1 bytes
2031680       0x1F0040        Squashfs filesystem, little endian, version 4.0, compression:xz, size: 3853792 bytes,
                              384 inodes, blocksize: 131072 bytes, created: 2021-06-11 12:44:23
6029376       0x5C0040        Squashfs filesystem, little endian, version 4.0, compression:xz, size: 3524226 bytes,
                              104 inodes, blocksize: 131072 bytes, created: 2021-06-11 12:44:23
```
By reading the output from binwalk on the latest firmware (4.36.3.19) we can start analyzing a couple of things:

1.  We have a uImage header which is used by the bootloader with a couple of CRC checksums
    -   It has a size of 64 bytes (0x40 - 0x0)
    -   It has no compression
    -   It starts at 0x0
2.  We have what is most likely a linux kernel `Linux-3.10.14`
    -   It has a size of 2 031 616 bytes (0x1F0040 - 0x40)
    -   It is compressed with LZMA
    -   It starts at 0x40
3.  We have two squashfs filesystems which we will analyze later
    -   squashfs(1) 
        -   It has a size of 3 997 696 bytes (0x5C0040 - 0x1F0040)
        -   It's xz compressed
        -   It starts at 0x1F0040
    -   squashfs(2)
        -   It has a size of 3 526 592 bytes (file size - 0x5C0040)
        -   It's xz compressed
        - It starts at 0x5C0040

## Breaking up the binary
Once the binary is unpacked using the wyze_extractor.py script. `./wyze_extractor.py unpack FILE_NAME` 

We can start messing with the FS.

### rcS
Right off the bat, there is nothing too interesting in rcS other than it calls `/system/init/app_init.sh`.

If we were to add an entry to start telnet via the command `telnetd &` it would be blocked by `/bin/iCamera` from the second squashfs entry.

Running `strings iCamera | grep -i "telnetd"` gives us:
```bash
telnetd &
killall -9 telnetd
```
To avoid having our telnet session getting killed by `iCamera` we can call it using busybox instead.

We can then add `busybox telnetd &` to the `rcS` file

### Root password
(Haven't received the cam yet cant test the root passwd)

## Reassembling the binary
### squashfs filesystems
We check the blocksize of the old squashfs with `unsquashfs -s squashfs_1`
To get the squashfs 1 and 2 back to their compressed states we run `mksquashfs squashfs_1_out/ squashfs_1_new -comp xz -b 131072` on both old squashfs folders

### Combining everything
We move both `squashfs_NUM_new` to `squashfs_NUM`

We run `./wyze_extractor.py pack FILE_NAME`

### Generating the uImage header
We start by fetching the required info with `binwalk -t uimage_header`
```
DECIMAL       HEXADECIMAL     DESCRIPTION
---------------------------------------------------------------------------------------------------------------------
0             0x0             uImage header, header size: 64 bytes, header CRC: 0xECCED73E, created: 2021-06-11
                              12:44:24, image size: 9555968 bytes, Data Address: 0x0, Entry Point: 0x0, data CRC:
                              0x49FA1464, OS: Linux, CPU: MIPS, image type: Firmware Image, compression type: none,
                              image name: "jz_fw"
```
We can now run `mkimage -A MIPS -O linux -T firmware -C none -a 0 -e 0 -n jz_fw -d exploited_wyze.bin demo_wcv3_exploited.bin`

## Flashing the micro sd card with the new firmware
Start by wiping the card clean by running:
```bash
fdisk /dev/mmcblk0 # Make sure you run it one the right drive
```
Then deleted the partition
```
Command (m for help): d
Selected partition 1
Partition 1 has been deleted
```
Create a new partition
```
Command (m for help): n
Partition type
    p   primary (0 primary, 0 extended, 4 free)
    e   extended (container for logical partitions)
Select (default p): p

...

Created a new partition 1 of type 'Linux' and of size XX.X GiB
```
Change type to Fat32
```
Command (m for help): t
Selected partition 1
Partition type (type L to list all types): b # W95 FAT32
```
Write the changes
```
Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks
```
Finally we format the drive
```bash
mkfs.vfat /dev/mmcblk0pX
```

## Flashing the cam
See [link](https://support.wyze.com/hc/en-us/articles/360031490871-How-to-flash-your-Wyze-Cam-firmware-manually)
