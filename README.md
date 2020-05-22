Samsung Gear 360 Firmware Notes
===============================

Just some notes on Samsung Gear 360 and its firmware. Experiments here were done using rooted Samsung Galaxy S5 DUOS phone (SM-900FD) and a Linux machine (Arch Linux).

Android APK
-----------

Manager application for Android contains few useful information. APK file can be [downloaded from this location](http://www.apkmirror.com/apk/samsung-electronics-co-ltd/samsung-gear-360-manager/samsung-gear-360-manager-1-0-4-release/samsung-gear-360-manager-1-0-4-android-apk-download/), there's also [this thread](http://forum.xda-developers.com/android/software/mod-samsung-gear-360-manager-device-t3400383) on xda-developers.

Next thing is to decompile .apk file, I used [this online service](http://www.javadecompilers.com/apk) to do it. Resulting source code is ~87 MB big.

Note: always check these online sites for worms/virises/etc. as they come and go and someone might put something nasty there.

Few interesting bits:

- file ```com/samsung/android/samsunggear360manager/app/btm/FWConstants.java``` contains location of proxy file that contains link to firmware file: ```public static final String FW_DOWNLOAD_SERVER_URL = "https://www.samsungimaging.com/common/support/firmware/downloadUrlList.do?prd_mdl_name=SM-C200&loc=global";```, now, this contains [link to firmware file](https://www.samsungimaging.com/file/download?XmlIdx=280&file=C200GLU0APE4_160519_1848_REV00_user.bin) (also check for latest version at the end of this file),
- file ```com/samsung/android/samsunggear360manager/app/btm/datatype/BTJsonSerializableMsgId.java``` contains constants with message types that are used during Bluetooth communication.

Communication
-------------

Android application uses Bluetooth communication to send commands to the camera, wifi direct is used to send image/video data.

### Dumping Bluetooth Traffic

Newer versions of Android have built-in functionality to dump Bluetooth traffic (you have to have [Developer Mode](https://www.google.nl/?ion=1&espv=2#q=how%20to%20enable%20developer%20mode%20on%20android) enabled). Go to Settings → Developer options → Select Bluetooth HCI snoop log. Log/dump file is then saved in ```/sdcard/btsnoop_hci.log``` (or not: see [stackoverflow](http://stackoverflow.com/questions/28445552/bluetooth-hci-snoop-log-not-generated)). This file can be viewed with [Wireshark](https://www.wireshark.org/).

![Android Bluetooth dump setting](bluetooth_dump_256p.png?raw=true "Android Bluetooth dump setting")
![Wireshark and Bluetooth dump](wireshark_bt.png?raw=true "Wireshark and Bluetooth dump")

Communication uses [JSON Format](https://en.wikipedia.org/wiki/JSON). Under Linux you can initiate Bluetooth serial connection with Gear360, but the data exchange protocol is not yet known (i.e. couldn't make it to work):

    > # Note: this doesn't work?
    > # 00:11:22:33:44:55 is a Bluetooth address of Gear360
    > # Use "hcitool scan" to find out BT address
    > sudo rfcomm bind 0 00:11:22:33:44:55
    > ls /dev/rfcomm0
    /dev/rfcomm0
    > # Send some bytes using keyboard, you can also use PuTTY
    > sudo screen /dev/rfcomm0 115200

From the Android itself we can also, after successful connection with Gear360, scan for open ports (you'll need [ssh](http://arachnoid.com/android/SSHelper/) server on your phone + [nmap for Android](https://www.google.nl/?ion=1&espv=2#q=nmap%20android%20binary)).

OS scan:

    root@MSM8974:/storage/_temp/nmap-6.46/nmap-6.46/bin # ./nmap --osscan-guess -O 192.168.49.10                                     
    
    Starting Nmap 6.46 ( http://nmap.org ) at 2016-07-16 22:48 GMT
    Nmap scan report for 192.168.49.10
    Host is up (0.0049s latency).
    Not shown: 997 closed ports
    PORT     STATE SERVICE
    53/tcp   open  domain
    7676/tcp open  imqbrokerd
    9001/tcp open  tor-orport
    MAC Address: 62:F1:89:7C:35:64 (Unknown)
    Device type: general purpose
    Running: Linux 2.6.X|3.X
    OS CPE: cpe:/o:linux:linux_kernel:2.6 cpe:/o:linux:linux_kernel:3
    OS details: Linux 2.6.32 - 3.9
    Network Distance: 1 hop
    
    OS detection performed. Please report any incorrect results at http://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 4.86 seconds

Port scan:

    root@MSM8974:/storage/_temp/nmap-6.46/nmap-6.46/bin # ./nmap -sV 192.168.49.10                                                   
    
    Starting Nmap 6.46 ( http://nmap.org ) at 2016-07-16 22:46 GMT
    Nmap scan report for 192.168.49.10
    Host is up (0.0025s latency).
    Not shown: 997 closed ports
    PORT     STATE SERVICE    VERSION
    53/tcp   open  tcpwrapped
    7676/tcp open  http       Samsung AllShare httpd
    9001/tcp open  tcpwrapped
    MAC Address: 62:F1:89:7C:35:64 (Unknown)
    
    Service detection performed. Please report any incorrect results at http://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 13.10 seconds

```tcpdump``` on Android can be used to dump wifi traffic, links to individual files can be extracted and then downloaded:

    root@MSM8974:/storage/_temp/nmap-6.46/nmap-6.46/bin # wget http://192.168.49.10:7679/smp00780060001010029.mp4
    Connecting to 192.168.49.10:7679 (192.168.49.10:7679)
    smp00780060001010029   7% |******                | 11694k  0:00:36 ETA


Firmware Structure
------------------

Recent firmware (```0.7```) is in a single file with a bit weird file format (couldn't find any references to it), [binwalk](http://binwalk.org/) is able to detect Linux kernels, but nothing else:

    DECIMAL       HEXADECIMAL     DESCRIPTION                                                                             --------------------------------------------------------------------------------                                      304           0x130           uImage header, header size: 64 bytes, header CRC: 0x863045F5, created: 2016-03-05 02:02:42, image siz
    e: 6684536 bytes, Data Address: 0x86008000, Entry Point: 0x86008000, data CRC: 0xD1C62632, OS: Linux, CPU: ARM, image type: OS Kern
    el Image, compression type: none, image name: "Linux-3.5.0"                                                           22256         0x56F0          gzip compressed data, maximum compression, from Unix, NULL date (1970-01-01 00:00:00)                 6729640       0x66AFA8        CRC32 polynomial table, little endian                                                                 6737469       0x66CE3D        uImage header, header size: 64 bytes, header CRC: 0xC60A14F1, created: 2016-05-17 02:11:31, image siz
    e: 3881064 bytes, Data Address: 0x86008000, Entry Point: 0x86008000, data CRC: 0xBE653D6D, OS: Linux, CPU: ARM, image type: OS Kern
    el Image, compression type: none, image name: "Linux-3.5.0"                                                           6759421       0x6723FD        gzip compressed data, maximum compression, from Unix, NULL date (1970-01-01 00:00:00)                 10618597      0xA206E5        uImage header, header size: 64 bytes, header CRC: 0x9B58B71E, created: 2016-02-01 02:57:39, image siz
    e: 6765472 bytes, Data Address: 0x86008000, Entry Point: 0x86008000, data CRC: 0x4B4643B0, OS: Linux, CPU: ARM, image type: OS Kern
    el Image, compression type: none, image name: "Linux-3.5.0"                                                           10640549      0xA25CA5        gzip compressed data, maximum compression, from Unix, NULL date (1970-01-01 00:00:00)

Still that information turned out to be useful to decode header of the binary file. If we take kernel offsets and file sizes we can find similar values in the firmware file (they differ a bit).

First 64 bytes seem to be a information about the file (magic bytes, firmware name, version, etc.):

    > hexdump -n 64 -C ./firmware.bin
    00000000  53 4c 50 00 30 2e 37 30  00 00 00 00 53 4d 43 32  |SLP.0.70....SMC2|
    00000010  30 30 00 00 00 00 00 00  00 00 00 00 53 4d 43 32  |00..........SMC2|
    00000020  30 30 47 4c 55 30 41 50  45 34 00 00 0a 00 00 00  |00GLU0APE4......|
    00000030  01 56 45 52 5f 52 65 76  30 2e 36 00 00 00 00 00  |.VER_Rev0.6.....|

Next there are 10 sections of 24 bytes in size: size (4 byte), ??? (4 bytes), offset (4 bytes), ??? (4 bytes), ??? (8 bytes):

    > hexdump -s 64 -n 240 -e '"%04_ax |" "%08x %08x %08x %08x %08x%08x\n"' ./firmware.bin
    0040 |0065ffb8 443caadc 00000130 ffffffff 0000000000000000
    0058 |0000cd55 3c568255 006600e8 7fffffff 3939344200384635
    0070 |003b38a8 edb0cac9 0066ce3d 65ffffff 0000000000000000
    0088 |00673be0 eeb4cbaf 00a206e5 543fffff 0000000000000000
    00a0 |00005b88 fd4ba0cf 010942c5 4321ffff 0000000000000000
    00b8 |00d25100 1ccc313b 01099e4d 32100fff 0000000000000000
    00d0 |0e9684d3 f40bc6fc 01dbef4d 210000ff 0000000000000000
    00e8 |004e7f30 86fae22f 10727420 1000000f 0000000000000000
    0100 |00005000 22b82134 10c0f350 ffffffff 0000000000000000
    0118 |034445cf b763ed14 10c14350 7fffffff 0000000000000000
    #     size     ???      offset   ???      ???
    # One of those has to be checksum, possibly the last field

The same but in decimal:

    hexdump -s 64 -n 240 -e '"%04_ax |" "%10u %10u %10u %10u %08x%08x\n"' ./firmware.bin
    0040 |   6684600 1144826588        304 4294967295 0000000000000000
    0058 |     52565 1012302421    6684904 2147483647 3939344200384635
    0070 |   3881128 3987786441    6737469 1711276031 0000000000000000
    0088 |   6765536 4004826031   10618597 1413480447 0000000000000000
    00a0 |     23432 4249592015   17384133 1126301695 0000000000000000
    00b8 |  13783296  483143995   17407565  839913471 0000000000000000
    00d0 | 244745427 4094412540   31190861  553648383 0000000000000000
    00e8 |   5144368 2264588847  275936288  268435471 0000000000000000
    0100 |     20480  582492468  281080656 4294967295 0000000000000000
    0118 |  54805967 3076779284  281101136 2147483647 0000000000000000
    #           size        ???     offset        ??? ???

Based on this we can extract portions of the firmware and see if we'll have better results:

    > dd if=firmware.bin of=chunk0.bin skip=304 count=6684600 bs=1

Now we can see the chunks file types:

    > file *.bin
    chunk0.bin: u-boot legacy uImage, Linux-3.5.0, Linux/ARM, OS Kernel Image (Not compressed), 6684536 bytes, Sat Mar  5 02:02:42 2016, Load Address: 0x86008000, Entry Point: 0x86008000, Header CRC: 0x863045F5, Data CRC: 0xD1C62632
    chunk1.bin: data
    chunk2.bin: u-boot legacy uImage, Linux-3.5.0, Linux/ARM, OS Kernel Image (Not compressed), 3881064 bytes, Tue May 17 02:11:31 2016, Load Address: 0x86008000, Entry Point: 0x86008000, Header CRC: 0xC60A14F1, Data CRC: 0xBE653D6D
    chunk3.bin: u-boot legacy uImage, Linux-3.5.0, Linux/ARM, OS Kernel Image (Not compressed), 6765472 bytes, Mon Feb  1 02:57:39 2016, Load Address: 0x86008000, Entry Point: 0x86008000, Header CRC: 0x9B58B71E, Data CRC: 0x4B4643B0
    chunk4.bin: data
    chunk5.bin: data
    chunk6.bin: lzop compressed data - version 1.030, LZO1X-1(15), os: Unix
    chunk7.bin: lzop compressed data - version 1.030, LZO1X-999, os: Unix
    chunk8.bin: data
    chunk9.bin: lzop compressed data - version 1.030, LZO1X-1(15), os: Unix

Looks better, let's check what's inside ```chunk7.bin```:

    > lzop -d ./chunk7.bin -o chunk7.img
    > file chunk7.img
    chunk7.img: Linux rev 1.0 ext4 filesystem data, UUID=c1295226-fb6d-4a51-9f5c-9776fec7bb4d, volume name "opt" (needs journal recovery) (extents) (large files)

Now we can mount this filesystem:

    > mkdir chunk7_mount
    > sudo mount -o loop ./chunk7.img chunk7_mount
    > cd chunk7_mount
    > ls
    apps  data  _dbspace  driver  etc  home  lost+found  pref  share  storage  usr  var  version  dbspace  ug

Note: chunks6 and 7 are ext4, chunk9 looks to be a swap partition. The other? Don't know.

chunk6 is root filesystem, chunk7 is ```/opt``` mount point.

Most likely there is a chunk for each of the default Tizen partitions described [here](https://docs.tizen.org/docs/open-source-tizen/porting/kernel.page#tizen-partition-layout).

![Tizen partition layout](https://docs.tizen.org/docs/open-source-tizen/porting/media/800px-Partitionlayout.png)

Getting Inside
--------------

Looks like there's not way to get a remote console; trying to connect on any of the non-HTTP ports gets you nowhere. For the moment I didn't find any hooks to enable debug mode or something smiliar. Possibly there's a console but on a hardware serial port.

### Interesting files

```/usr/apps/com.samsung.di-camera-app/bin/di-camera-app``` - looks like the main camera application, anything interesting there:
    
    > strings ./di-camera-app    
    sdcard_name
    sdcard_name_eng
    /sdcard/osd_capture
    /sdcard/temp/
    make directorr /sdcard/temp/
    /sdcard/temp/temp_%d_%d.txt
    /opt/storage/sdcard/
    /opt/storage/sdcard/%s.bin
    /opt/storage/sdcard/%s_eng.bin
    /opt/storage/sdcard/smc200.bin
    /opt/storage/sdcard/smc200_eng.bin
    /sdcard/test.internet
    /sdcard/SYSTEM/
    /sdcard/SYSTEM/system.bin
    chmod +t /sdcard/system/system.bin
    set url file:///sdcard/1.html
    /sdcard/

Putting smc200.bin file on the sdcard doesn't do anything.

File: ```/etc/passwd```:

    > grep -v false /etc/passwd
    root::0:0:root:/root:/bin/sh
    bin:*:1:1:bin:/bin:
    daemon:*:2:2:daemon:/sbin:
    ftp:*:14:50:FTP User:/home/ftp:
    system:x:1000:1000:system:/home/system:/bin/sh
    app:x:5000:5000:In-house application:/home/app:/bin/sh
    sshd:x:112:65534::/var/run/sshd:/usr/sbin/nologin

### Grepping

Grepping the whole filesystem for ```sdcard``` and ```.tg``` did not result in any interesting discoveries, except if you put ```info.tg``` on sdcard and start the camera - some kind of test mode is enabled, to turn it off you have to remove the file from the sdcard.

### chroot

It is possible to inspect some applications using qemu-user-static (see this answer on [stackoverflow](http://unix.stackexchange.com/a/222981), also [here](https://wiki.gentoo.org/wiki/Crossdev_qemu-static-user-chroot)). That way we can chroot into mounted Gear360 images and run programs which are there (not everything will work as it is not real hardware).

### info.tg

It looks like the trick described here: [telnet on nx500](https://github.com/ottokiksmaler/nx500_nx1_modding/blob/master/Running-telnet-server-on-camera.md) almost works. The problem is that if you follow that guide a script will be executed, but... since camera stays in test mode it is not possible to connect to it. One solution would be to initiate a connection from the script, or somehow exit test mode.

Somewhere in the image there's a mention about ```info.tgw```, it still displays test menu, but it is possible to connect to camera, but... script is not executed.

Links
-----

Few links:

- [official Samsung Gear360 website](http://www.samsung.com/global/galaxy/gear-360/),
- [source code for Samsung Gear360](https://opensource.samsung.com/uploadSearch?searchValue=c200), 1.4 GB, note: not everything is open source in the final firmware, there are also no instructions how to flash the firmware),
- [firmware file version 0.7](https://www.samsungimaging.com/file/download?XmlIdx=280&file=C200GLU0APE4_160519_1848_REV00_user.bin) (~340 MB),
- [firmware file version 0.83](https://www.samsungimaging.com/file/download?XmlIdx=291&file=C200GLU0AQF1_170619_2058_REV00_user.bin) (~270 MB),
- [firmware file version 2017-11-21](https://www.samsungimaging.com/file/download?XmlIdx=291&file=C200GLU0AQK1_171121_1257_REV00_user.bin) (~266 MB),
- [NX500/NX1 modding](https://github.com/ottokiksmaler/nx500_nx1_modding), modding resources for a different Samsung cameras,
- [FCC page](https://fccid.io/A3LSMC200),
- [Internal images from FCC page](https://fccid.io/A3LSMC200/Internal-Photos/Internal-Photos-20160222-v1-05-Internal-Photos-SM-C200-2910602).

Note that some things could be copied from NX-* camera tutorials/guides (some things are similar).
