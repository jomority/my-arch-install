# Optional: Setup Snapper

If you use Btrfs for your root partition you can make use of its snapshot functionality using [Snapper](https://wiki.archlinux.org/title/Snapper).


## Install & Configure Snapper

```bash
paru -S snapper
sudo snapper -c root create-config /
sudo snapper -c home create-config /home
```

Adjust settings, especially the `TIMELINE_LIMIT_*` ones, which specify how much snapshots to keep:

```bash
sudo vim /etc/snapper/configs/root
sudo vim /etc/snapper/configs/home
```

Make `/home/.snapshots` readable by everyone, so that users can look into and read their own snapshots:

```bash
sudo chmod o+rx /home/.snapshots
```


## Create subvolume for root snapshots

This ensures that the snapshots are saved outside of the snapshotted root subvolume (`@`), so that replacing the root subvolume is possible.
Read more about it in the [ArchWiki](https://wiki.archlinux.org/title/Snapper#Restoring_/_to_its_previous_snapshot).

```bash
sudo mkdir -p /mnt/tmp/
sudo mount /dev/nvme0n1p2 /mnt/tmp/  # or /dev/sda2
sudo btrfs subvolume create /mnt/tmp/@snapshots
chown --reference=/.snapshots/ /mnt/tmp/@snapshots
chmod --reference=/.snapshots/ /mnt/tmp/@snapshots
sudo btrfs subvolume delete /.snapshots/
sudo mkdir /.snapshots/
sudo mount -o subvol=@snapshots /dev/nvme0n1p2 /.snapshots/  # or /dev/sda2
```

Make the mount persistent in the `/etc/fstab`:

```diff
--- /etc/fstab
+++ /etc/fstab
@@ -17,3 +17,6 @@
 # /dev/nvme0n1p3
 UUID=900ca734-b8ab-48a3-9a60-6f8ddcc52905	none      	swap      	defaults  	0 0
 
+# /dev/nvme0n1p2
+UUID=df62f6a7-40e5-4eee-a839-313e202aebb9	/.snapshots	btrfs     	rw,relatime,compress=zstd:3,space_cache=v2,subvolid=261,subvol=@snapshots	0 0
+
```

<details>
<summary>Command</summary>

```bash
paru -S --needed arch-install-scripts
genfstab -U / | grep -B 1 -A 1 '\s/.snapshots\s' | sudo tee -a /etc/fstab >/dev/null
```
</details>


## Optional: Create subvolumes for things that shouldn't be included in snapshots

We already did that in [Install base system](./base-install.md#option-2-advanced-btrfs-as-root-filesystem) for `/var/`.
It may be useful to do it for other folders.
The following examples are for `/srv/` or `/home/user/.cache/`.


### Reboot in single-user mode

Best to not move the folders on a running system.
To enter single-user mode a root password is needed.
Set one with:

```bash
sudo passwd root
```

Reboot into single user mode by changing the kernel command line in GRUB or systemd-boot.
Both bootloaders allow this with the *e* key from within the menu.
To get into the systemd-boot menu hold the *space* key at boot.
Append to the command line:

```bash
systemd.unit=rescue.target
```

Enter the root passwort you just set.
You can directly unset the password again to not forget it later:

```bash
passwd -l root
```

### Create and mount subvolumes (folders in /)

For folders in `/` we create a special `@...` subvolume that we mount into the root filesystem.
This ensures that you can boot into a snapshot and these folders are preserved in the current state and not read-only.
Example for `/srv/`:

```bash
mount /dev/nvme0n1p2 /mnt/tmp/  # or /dev/sda2
btrfs subvolume create /mnt/tmp/@srv
chown --reference=/srv /mnt/tmp/@srv
chmod --reference=/srv /mnt/tmp/@srv
mv /srv/{,.}* /mnt/tmp/@srv/
mount -o subvol=@srv /dev/nvme0n1p2 /srv/  # or /dev/sda2
```

Make the mount persistent in the `/etc/fstab`:

```diff
--- /etc/fstab
+++ /etc/fstab
@@ -20,3 +20,6 @@
 # /dev/nvme0n1p2
 UUID=df62f6a7-40e5-4eee-a839-313e202aebb9	/.snapshots	btrfs     	rw,relatime,compress=zstd:3,space_cache=v2,subvolid=261,subvol=@snapshots	0 0
 
+# /dev/nvme0n1p2
+UUID=df62f6a7-40e5-4eee-a839-313e202aebb9	/srv      	btrfs     	rw,relatime,compress=zstd:3,space_cache=v2,subvolid=262,subvol=@srv	0 0
+
```

<details>
<summary>Command</summary>

```bash
paru -S --needed arch-install-scripts
genfstab -U / | grep -B 1 -A 1 '\s/srv\s' >> /etc/fstab
```
</details>


### Create subvolumes (folders in /home/)

For folders in `/home/` we don't really need a mounted `@...` subvolume, since we don't restore the whole subvolume.
Example for `/home/user/.cache/`:

```bash
btrfs subvolume create /home/user/.cache.tmp
chown --reference=/home/user/.cache /home/user/.cache.tmp
chmod --reference=/home/user/.cache /home/user/.cache.tmp
mv /home/user/.cache/{,.}* /home/user/.cache.tmp/
rmdir /home/user/.cache/
mv /home/user/.cache.tmp/ /home/user/.cache
```


### Reboot

```bash
passwd -l root
reboot
```


## Arm automatic snapshots

Enable periodic snapshots and periodic cleanup:

```bash
sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
```


### Optional: Snapshots on pacman operations

If you want root snapshots taken after installing, removing or updating packages use [snap-pac](https://github.com/wesbarnett/snap-pac):

```bash
paru -S snap-pac
```

You may want limit the number of such snapshot with the `NUMBER_LIMIT` configuration option of snapper:

```bash
sudo vim /etc/snapper/config/root
```


## Optional: Bootloader entries for snapshots

This makes it possible to boot directly into snapshots from the bootloader.
This can probably be very useful if the system won't boot after some broken update (didn't happen to me yet).
From the working system it is then easier to solve the problem and/or restore the root subvolume.


### Maybe: If bootloader GRUB

```bash
gpg --recv-keys EB4F9E5A60D32232BB52150C12C87A28FEAC6B20
paru -S snap-pac-grub
```


### Maybe: If bootloader systemd-boot

TODO: This package is not yet in the AUR.

```bash
paru -S snapper-efi-entries
```

---

TODO: Recommendations for other programs
