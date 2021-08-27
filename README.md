# wyzecamv3 Telnet enabler

## Analyzing the `.bin`

By running `binwalk -t demo_wcv3.bin` we can figure out a couple of things.
```bash
 
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
        -   It is a squashfs compressed filesystem
        -   It starts at 0x1F0040
    -   squashfs(2)
        -   It has a size of 3 526 592 bytes (file size - 0x5C0040)
        -   It is a squashfs compressed filesystem
        - It starts at 0x5C0040

## Breaking up the binary

