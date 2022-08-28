# Setting up Raspberry Pi as a media/backup server using SMB

**Disclaimer:** This guide works for Raspberry Pi Model 4B Running Raspbian 64-bit, YMMV for different device models and operating systems. The intent here is to use the Raspberry Pi as a low powered file server for sharing media on a home network, different configuration may be required in different environments.  
**Original Credit**:
[ShellHacks](https://www.shellhacks.com/raspberry-pi-mount-usb-drive-automatically/) and [aallan](https://aallan.medium.com/adding-an-external-disk-to-a-raspberry-pi-and-sharing-it-over-the-network-5b321efce86a)


#### Set static IP via command line

- Setting a static IP for your Raspberry Pi means your shared directories will always be in the same network location.
1. Use `ifconfig` to discover the name of your network interface
```
$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.2  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::98ca:960e:817d:c96e  prefixlen 64  scopeid 0x20<link>
        ether dc:a6:32:a0:d4:60  txqueuelen 1000  (Ethernet)
        RX packets 342233  bytes 50840997 (48.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 565541  bytes 613928838 (585.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 41  bytes 5048 (4.9 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 41  bytes 5048 (4.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.9  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::25d4:a378:f3b3:6b05  prefixlen 64  scopeid 0x20<link>
        ether dc:a6:32:a0:d4:62  txqueuelen 1000  (Ethernet)
        RX packets 3326  bytes 281560 (274.9 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 229  bytes 39040 (38.1 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```
- Here `eth0` is our Ethernet interface and `wlan0` is our wireless interface.

2. Set IP to a static value including netmask
```
$ sudo ifconfig eth0 192.168.1.2 netmask 255.255.255.0
```
3. Add static route to default gateway (the IP address of your router)
```
$ sudo route add default gw 192.168.1.1 eth0
```

#### Identify drives
1. Run `lsblk` to identify disks and partitions

```shell
$ lsblk -fp
NAME             FSTYPE LABEL      UUID      FSAVAIL   FSUSE%  MOUNTPOINT
/dev/sda
└─/dev/sda1      vfat   USD Drive  FC05-DF26
/dev/mmcblk0
├─/dev/mmcblk0p1 vfat   boot       634...    199.9M    21%     /boot
└─/dev/mmcblk0p2 ext4   rootfs     805...    24.3G     12%     /
```

- The `f` is to display file systems and the `p` is to display partition information
- Drives are listed as a block device (`sda`/`sdb` etc) followed by the partition (`sda1`, `sdb1` etc)

#### Create mount point

```
$ sudo mkdir /mnt/usb0
```

#### Mount drives according to filesystem type
1. If Raspbian has auto mounted your drives it may be necessary to unmount them before proceeding. **YMMV**, you can try proceeding with Step 2 but if you encounter any errors you can try `umount`:
```
$ sudo umount /media/pi/MyDrive
```
**OR**
```
$ sudo umount /dev/sda
```

2. Mount drives using the correct command for matching filesystem of your drive


`FAT`

```
$ sudo mount -t vfat /dev/sda1 /mnt/usb0 -o umask=000
```

`NTFS:`

```
$ sudo apt install ntfs-3g
$ sudo mount -t ntfs /dev/sda1 /mnt/usb0 -o umask=000
```

`exFAT:`

```
$ sudo apt install exfat-fuse
$ sudo mount -t exfat /dev/sda1 /mnt/usb0
```

`EXT4:`

```
$ sudo mount -t ext4 /dev/sda1 /mnt/usb0
```

3. To verify drive is mounted

```
$ ls -lt /mnt/usb0
```

4. To unmount ***IF NEEDED***

```
$ sudo umount /mnt/usb0
```

<br>

#### Install Samba and create share/s via config file

1. Install the Samba software
```
$ sudo apt install samba samba-common-bin
```

2. Edit the Samba configuration file
```
$ sudo nano /etc/samba/smb.conf
```
- Append the following to the end of the file. The word in square brackets [] will be the name of your share on the network, the Comment line lets you write a description for yourself. You can do this multiple times for multiple shares.
```
[share]
Comment = Shared Folder
Path = /mnt/usb
Browseable = yes
Writeable = Yes
only guest = no
create mask = 0777
directory mask = 0777
Public = no
Guest ok = yes
```
- If you don't want guest access omit the `Guest ok = yes`
- Setting `Public = yes` means anyone can open the share without authentication. This can be helpful for testing but authentication is recommended for security.

#### Create Samba user and give them ownership of the mounted directories

*This is to provide authentication for connecting machines*


1. Use `adduser` to add a new user
```
$ sudo adduser pismb
```

2. Use `chown` to change ownership of the directories to the Samba user we created.
```
sudo chown -R pismb: /mnt/usb0
```

3. Run `id` to get your UID (user ID) and GID (group ID). Note this down for the next step.
```
id pismb
```

#### Set up auto mounting

1. Run `blkid` to get the UUID of your drive/s

```
$ sudo blkid
/dev/mmcblk0p1: LABEL_FATBOOT="boot" LABEL="boot" UUID="6341-C9E5" TYPE="vfat" PARTUUID="ea7d04d6-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="80571af6-21c9-48a0-9df5-cffb60cf79af" TYPE="ext4" PARTUUID="ea7d04d6-02"
/dev/sda1: UUID="FC05-DF26" TYPE="vfat" PARTUUID="2d72d270-01"
```

2. Make a backup of and open `/etc/fstab` in your text editor of choice

```
$ sudo cp /etc/fstab /etc/fstab.back
$ sudo nano /etc/fstab
```

3. Depending on filesystem, append one of the below lines to `fstab`

*Remember to replace `UUID` with your drive/s UUID and `/mnt/usb0` with the directory you created*

`FAT`

```
$ UUID=FC05-DF26 /mnt/usb0 vfat defaults,auto,users,rw,nofail,umask=000 0 0
```

`NTFS:`

```
$ UUID=FC05-DF26 /mnt/usb0 ntfs defaults,auto,users,rw,nofail,umask=000 0 0
```

`exFAT:`

```
$ UUID=FC05-DF26 /mnt/usb0 exfat defaults,auto,users,rw,nofail 0 0
```

`EXT4:`

```
$ UUID=FC05-DF26 /mnt/usb0 ext4 defaults,auto,users,rw,nofail 0 0
```

4. Save and exit - In Nano this is Ctrl + Shift + X followed by pressing Y to save

5. Restart the Samba service
```
$ sudo systemctl restart smbd
```


<br>


# Notes

- Tested using NTFS and exFAT.