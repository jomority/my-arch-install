# Optional: Setup Snapper

If you use Btrfs for your root partition you can make use of its snapshot functionality using [Snapper](https://wiki.archlinux.org/title/Snapper).

## Create subvolume for root snapshots

This ensures that the snapshots are saved outside of the snapshotted root subvolume (`@`), so that replacing the root subvolume is possible.
Read more about it in the [ArchWiki](https://wiki.archlinux.org/title/Snapper#Restoring_/_to_its_previous_snapshot).

```bash
sudo mkdir -p /mnt/tmp/ /.snapshots
sudo mount /dev/nvme0n1p2 /mnt/tmp/  # or /dev/sda2
sudo btrfs subvolume create /mnt/tmp/@snapshots

# TODO: update /etc/fstab
```


## Optional: Put everything that shouldn't be snapshotted on separate subvolumes

We already did that in [Install base system](./base-install.md#option-2-advanced-btrfs-as-root-filesystem) for `/var/`.
It may be useful to do it for other folders, for example `/srv/` or `/home/user/.cache/`.

Best to not move the folders on a running system.
Reboot into single user mode by changing the kernel command line in GRUB or systemd-boot.
Both bootloaders allow this with the *e* key from within the menu.
To get into the systemd-boot menu hold the *space* key at boot.
Append to the command line:

```bash
init=/bin/bash
```

Then create extra subvolumes (example for `/srv/`):

```bash
sudo mount /dev/nvme0n1p2 /mnt/tmp/  # or /dev/sda2
btrfs subvolume create /mnt/tmp/@srv
mv /srv/{,.}* /mnt/tmp/@srv/

# TODO: update /etc/fstab
```

For folders in `/home/` we don't really need a mounted `@...` subvolume, since we don't restore the whole subvolume (example for `/home/user/.cache/`):

```bash
btrfs subvolume create /home/user/.cache.tmp
mv /home/user/.cache/{,.}* /home/user/.cache.tmp/
rmdir /home/user/.cache/
mv /home/user/.cache.tmp/ /home/user/.cache
```

Finally reboot:

```bash
reboot
```


## Configure Snapper

Install:

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

Enable periodic snapshots and periodic cleanup:

```bash
sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
```


## Optional: Snapshots on pacman operations

If you want root snapshots taken after installing, removing or updating packages use [snap-pac](https://github.com/wesbarnett/snap-pac):

```bash
paru -S snap-pac
```

You may want limit the number of such snapshot with the `NUMBER_LIMIT` configuration option of snapper:

```bash
sudo vim /etc/snapper/config/root
```


## Optional: Bootloader entries for snapshots

TODO: explain

```bash
gpg --recv-keys EB4F9E5A60D32232BB52150C12C87A28FEAC6B20
paru -S snap-pac-grub
```

```bash
paru -S snapper-efi-entries
```

---

TODO: Recommendations for other programs
