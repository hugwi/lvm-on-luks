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


