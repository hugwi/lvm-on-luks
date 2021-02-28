# lvm-on-luks

This doesn't cover you from *evil maid attack* attacks.

Evil maid attacks are pretty hard to defend against. UEFI provides some protection on systems that can use secure boot, but this isn't yet supported by Linux. Here are a few defenses you can employ in the meantime.

    Keep your boot partititon on a removable drive you carry with you at all times. I'd reccomend putting it on a usb drive you can wear as jewlery (necklace, bracelet, earings, etc). So long as no one steals this and backdoors it, you're safe.
    Never leave your laptop unattended. Also, if your threat model includes ninjas, guard it when you sleep.

As you can see, the fixes for this attack are pretty far past the threshold of what the average user might consider a reasonable consession in favor of security :v

**Useful links**
* [useful link for full encryption](https://gist.github.com/huntrar/e42aee630bee3295b2c671d098c81268)

* [example of evil made attack issue](https://www.schneier.com/blog/archives/2009/10/evil_maid_attac.html)

*  [might be an alternative solution that also protect it from that. ](https://cryptsetup-team.pages.debian.net/cryptsetup/encrypted-boot.html) 

* [gitrepo for conducting attack](https://github.com/nyxxxie/de-LUKS)
* [prevent attack](https://www.wzdftpd.net/blog/implementing-the-evil-maid-attack-on-linux-with-luks.html)
* [good thread on arch linux about evil maid attck](https://bbs.archlinux.org/viewtopic.php?id=87336)
* [luksencrypt example](https://linuxhint.com/setup-luks-encryption-on-arch-linux/)

## Drive preparation

Before encrypting a drive, it is recommended to perform a secure erase of the disk by overwriting the entire drive with random dat a to prevent cryptographic attacks or unwanted file recover. 

As a best effort to minimize flash memory cache artifacts, consider performing a SSD memory cell clearing prior to instructions below.

On occasion, users may wish to completely reset an SSD's cells to the same virgin state they were manufactured, thus restoring it to its factory default write performance. Write performance is known to degrade over time even on SSDs with native TRIM support. TRIM only safeguards against file deletes, not replacements such as an incremental save. 

`hdparm -I /dev/sdX | grep frozen`

Should report not frozen 

`hdparm --user-master u --security-set-pass PasSWorD /dev/sdX`

` hdparm -I /dev/sdX`

Will not output 

```
Security:
        Master password revision code = 65534
                supported
                enabled
        not     locked
        not     frozen
        not     expired: security count
                supported: enhanced erase
        Security level high
        2min for SECURITY ERASE UNIT. 2min for ENHANCED SECURITY ERASE UNIT.
```

Clean the partition 

`hdparm --user-master u --security-erase PasSWorD /dev/sdX`

`hdparm -I /dev/sdX`

will not have not enabled again

```
Security:
        Master password revision code = 65534
                supported
        not     enabled
        not     locked
        not     frozen
        not     expired: security count
                supported: enhanced erase
        2min for SECURITY ERASE UNIT. 2min for ENHANCED SECURITY ERASE UNIT.
```
                       

Prior to creating any partitions, you should inform yourself about the importance and methods to securely erase the disk. This is described on [archlinux wiki - Securely wipe disk](https://wiki.archlinux.org/index.php/Securely_wipe_disk) but there's a simpler method (also described in the arch linux wiki [Drive preparation](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation). 

First, create a temporary encrypted container on the partition (using the form sdXY) or complete device (using the form sdX) to be encrypted:

`cryptsetup open --type plain -d /dev/urandom /dev/<block-device> to_be_wiped`

`# lsblk

NAME          MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda             8:0    0  1.8T  0 disk
└─to_be_wiped 252:0    0  1.8T  0 crypt
`

Wipe the container with zeros. A use of if=/dev/urandom is not required as the encryption cipher is used for randomness.

`dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress`

dd: writing to ‘/dev/mapper/to_be_wiped’: No space left on device

Remeber to Restrict /boot permissions

chmod 700 /boot

## Securing Hardware with Coreboot

[Uefi vs coreboot and how to secure firmware](https://blog.akendo.eu/post/2019.06.10-securing-hardware-with-coreboot/#flaws-in-uefi-and-secure-boot)

It is possible to define addition keys for the LUKS partition. This enables the user to create access keys for safe backup storage In so-called key escrow, one key is used for daily usage, another kept in escrow to gain access to the partition in case the daily passphrase is forgotten or a keyfile is lost/damaged. A different key-slot could also be used to grant access to a partition to a user by issuing a second key and later revoking it again.

`cryptsetup luksFormat --type luks1 --use-random -S 1 -s 512 -h sha512 -i 5000 /dev/nvme0n1p3`

## grub and luks doesn't seem to be optimal accoring to https://www.reddit.com/r/archlinux/comments/6ahvnk/grub_decryption_really_slow/

```
Welcome to /boot encryption sponsored by GRUB, where you fuck yourself over by adding a second, unaudited, shitty LUKS implementation to your system for zero gain.

Grub's implementation is really shitty and slow on your machine, since (unlike the kernel) it can only do pure software decryption or AESNI, not SSE-accelerated decryption. Easiest solution is to get rid of /boot encryption (it's useless, you only need one password anyway if everything's on BTRFS) and rely on the actually decent kernel LUKS.
6
level 2
Comment deleted by user
3 years ago
Continue this thread 
User avatar
level 1
ipha
3 years ago

When setting up the encryption did you set the --iter-time flag? It controls the length of time spent running password hash iterations and it defaults to 2000 ms. Seems like it might have been set to 20000.

https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode
3
User avatar
level 2
DragonSlayerC
3 years ago

Thanks for the suggestion! I had set it to 5000 because I would be fine with a 5 second unlock time. Apparently, though, GRUB has absolutely no optimization for LUKS other than AES-NI, which my laptop doesn't support, so it was extremely slow. I don't need an obscene amount of security, just enough to prevent snooping, so I redid the install but with only 1000 itertime instead, and now GRUB only need about 4 seconds to unlock and the boot process is also a couple of seconds faster, although it wasn't bad before at 20 seconds (GRUB was what really annoyed me). Thanks again!

Seen people using -i 5000 and --use-random but dont' think it's needed. use-random seems to mostly adress issue for embedded systems taking long time and iterations only use the default. 
```

`-S`
Once an encrypted partition has been created, the initial keyslot 0 is created (if no other was specified manually). Additional keyslots are numbered from 1 to 7. Which keyslots are used can be seen by issuing 
`-h` type of hash function. 
`-s` size of key-file used for the hash. 

If it is intended to use multiple keys and change or revoke them, the --key-slot or -S option may be used to specify the slot: 

#### Open the container (decrypt it and make available at /dev/mapper/cryptlvm)
```
cryptsetup open /dev/nvme0n1p3 cryptlvm
```

### Preparing the logical volumes
#### Create physical volume on top of the opened LUKS container
```
pvcreate /dev/mapper/cryptlvm
```

#### Create the volume group and add physical volume to it
```
vgcreate vg /dev/mapper/cryptlvm
```

#### Create logical volumes on the volume group for swap, root, and home
```
lvcreate -L 8G vg -n swap
lvcreate -L 32G vg -n root
lvcreate -l 100%FREE vg -n home
```

The size of the swap and root partitions are a matter of personal preference.

#### Format filesystems on each logical volume
```
mkfs.ext4 /dev/vg/root
mkfs.ext4 /dev/vg/home
mkswap /dev/vg/swap
```

#### Mount filesystems
```
mount /dev/vg/root /mnt
mkdir /mnt/home
mount /dev/vg/home /mnt/home
swapon /dev/vg/swap
```

## Move from one partition to another 

What you want is rsync.

This command can be used to synchronize a folder, and also resume copying when it's aborted half way. The command to copy one disk is:

rsync -avxHAX --progress / /new-disk/

The options are:

-a  : all files, with permissions, etc..
-v  : verbose, mention files
-x  : stay on one file system
-H  : preserve hard links (not included with -a)
-A  : preserve ACLs/permissions (not included with -a)
-X  : preserve extended attributes (not included with -a)


Michael Aaron Safyan's answer doesn't account for sparse files. -S option fixes that.

Also this variant doesn't spam with the each file progressing and doesn't do delta syncing which kills performance in non-network cases.

Perfect for copying filesystem from one local drive to another local drive.

rsync -axHAWXS --numeric-ids --info=progress2

Personally I don't think progress2 is that much better since its only show progress of a directory and not the entire progress. 



