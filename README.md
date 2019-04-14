## configuring full disc encryption for two drives with Ubuntu 18.04 LTS

WARNING: THIS DESTROYS ALL DATA ON BOTH DRIVES.

## References:

This procedure is entirely based upon andor-kiss' how to guide (first reference below) with some small changes and edits:

1.	[how-do-i-install-18-04-using-full-disk-encryption-with-two-drives-ssd-hdd](https://askubuntu.com/questions/1034079/how-do-i-install-18-04-using-full-disk-encryption-with-two-drives-ssd-hdd)
2.	[encrypting-a-second-hard-drive-on-ubuntu-14-10-post-install](https://­davidyat.es/2015/04/03/encrypting-a-second-hard-drive-on-ubuntu-14-10-post-install/)
3.	[move-home-folder-to-second-drive](https://askubuntu.com/questions/21321/move-home-folder-to-second-drive)

## Install Ubuntu 18.04 LTS on the primary drive (probably the smaller faster drive)

Install Ubuntu 18.04 LTS onto your primary drive and check

1.	erase the drive
2.	encrypt the installation, and
3.	LVM management.

Note that this *may* install or reuse a MBR on the *other* secondary drive. If you want the MBR to be on the same primary drive as your OS, one simple way to do this is to physically unplug the sata cable to your other drive before installation -- that way the ubuntu installation wizard will only have one option of where to put the MBR. ( if you do this, observe the usual precautions involved when playing with hardware: e.g. turn the machine off first, unplug the power, wait a bit, ground yourself for static...)

After installation, we should have a fresh ubuntu install with full disk encryption on the primary drive.

## formatting secondary drive to use full disc encryption

Now, we are going to configure the second drive. This will destroy all data on this second drive. Then we will configure the second drive to serve as a `/home` partition using full disk encryption.

### destroy all data on the second drive, replace with a fresh ext4 partition

sudo apt install gparted

1.	Open gParted
2.	select second HDD
3.	delete any & all partitions
4.	create a new PRIMARY PARTITION using the ext4 file system. Optionally, label it.
5.	apply changes
6.	wait for gParted to finish, then close gParted

### install LUKS container in the second drive 

In the following, replace `{sdx}` with the name of your secondary drive. For example, when I did this, `{sdx}` was `sda1`.

As part of this setup you will be prompted for a passphrase. This is the *new* passphrase to use when encrypting the secondary drive. hint: consider using the same passphrase for both drives, unless you have high level of belief in your ability to remember passphrases.

```
sudo cryptsetup -y -v luksFormat /dev/{sdx}
sudo cryptsetup luksOpen /dev/{sdx} {sdx}_crypt
sudo mkfs.ext4 /dev/mapper/{sdx}_crypt
```

## automatically mount and decrypt the secondary drive on startup

There’s a way to automatically mount and decrypt your second drive on startup, when your computer prompts you for the primary hard drive decryption password.

### create keyfile for secondary drive

```
sudo dd if=/dev/urandom of=/root/.keyfile bs=1024 count=4
sudo chmod 0400 /root/.keyfile
sudo cryptsetup luksAddKey /dev/{sdx} /root/.keyfile
```

In the following, replace `{sdx_uuid}` with the uuid of your secondary drive. Discover this via `sudo blkid`. The value you want is the UUID of `/dev/{sdx}`, not `dev/mapper/{sdx}_crypt`. Make sure you copy the UUID, not the PARTUUID. Also, make sure you run `blkid` after you have run the above `cryptsetup` commands for the secondary disk: the reported UUID of the secondary disk is changed by running `cryptsetup`, so if you run `blkid` earlier it will report a different value, and this will not work.

Then append the following lines to `/etc/crypttab` (using e.g. `sudoedit /etc/crypttab`):

```
{sdx}_crypt UUID={sdx_uuid} /root/.keyfile luks,discard
```

Note: after you make a change to `/etc/crypttab`, you can run `sudo cryptdisks_start {sdx}_crypt` to test it out quickly.

### test that both primary and secondary drives are decrypted upon ubuntu login

Reboot ubuntu and log in.

When you choose “Other Locations” (in the file manager thing aka nautilus) the second drive should show up in the list and have a lock icon on it, but the icon should be unlocked.

#### troubleshooting

If your secondary encrypted drive was not automatically decrpypted, this may incidate that your `/etc/crypttab` file is misconfigured.

For more information, you can review system logs using `sudo journalctl`. The system logs may include an error message of the form: "Dependency failed for Cryptography Setup for `sda1_crypt`".

This may indicate that your `/etc/crypttab` is misconfigured. Re-run `sudo blkid` and double check that the UUID entry for `/dev/{sdx}` matches the line in your `/etc/crypttab`.

### move your `/home folder` into the secondary drive

Please ensure the above steps have all succeeded before proceeding.

Migrate data

```
sudo mkdir /mnt/tmp
sudo mount /dev/mapper/{sdx}_crypt /mnt/tmp
sudo rsync -avx /home/ /mnt/tmp
```

Check it: mount migrated data as `/home`, check everything is present:

```
sudo mount /dev/mapper/sd?X_crypt /home
cd /home
ls
```

note: if you find that you have a second `home` inside your `/home`, this indicates a mistake when `rsync`-ing across your files: the trailing `/` on the end of `/home/` is required when invoking `rsync`. To fix this, `rm -rf /mnt/tmp/home` then re-run the above `rsync` command, being careful to have EXACTLY the correct slashes in place.

Make change permanent: `sudoedit /etc/fstab`, append the following line to `/etc/fstab`:

```
/dev/mapper/{sdx}_crypt /home ext4 defaults 0 2
```

Reboot.

