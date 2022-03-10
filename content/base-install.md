# Install base system

Boot into the live environment using UEFI.


## Setup disk

Use [gdisk](https://wiki.archlinux.org/title/GPT_fdisk#Create_a_partition_table_and_partitions) to partition the disk.
Create at least:

- A 512MiB *EFI system partition* partition
  - If it does not already exists. In that case DO NOT format it with the `mkfs.fat` command below.
- A *Linux x86-64 root* partition
- Optionally a *Linux swap* partition
  - If hibernation is desired, its size should be at least the RAM size, e.g. 8192 MiB for 8GB RAM
  - Alternatively you can use either no swap at all or a swapfile (this is not advised with btrfs)

Format the partitions (replace partition numbers accordingly):

```bash
mkfs.fat /dev/nvme0n1p1  # or /dev/sda1
mkfs.btrfs /dev/nvme0n1p2  # or /dev/sda2
mkswap /dev/nvme0n1p3  # or /dev/sda3
```

You can also use ext4 (or any other filesystem) instead of btrfs for the root partition if you feel less adventurous.
However it is then not possible to use snapshots in the [later chapters](./snapper.md).

*Hint:* This can also be done graphically from an already installed distro or [gparted live iso](https://gparted.org/download.php).
In GParted, use *fat32* for the *EFI system parition* partition and set the *boot* and *esp* flags.


## Setup network

Connect Ethernet cable or use [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl) for WiFi.

Test connection:

```bash
ping archlinux.org
```


## Setup time

Change the timezone accordingly.

```bash
timedatectl set-ntp 1
timedatectl set-timezone Europe/Berlin
```


## Mount and prepare filesystem

We create the btrfs subvolumes `@`, `@home` and `@var` for `/`, `/home/` and `/var/` respectively.
While this is not strictly necessary this makes it possible to snapshot them separately (see the [ArchWiki](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout) and the [later chapter about snapper](./snapper.md))

```bash
mkdir -p /mnt/root/ /mnt/tmp/
mount /dev/nvme0n1p2 /mnt/tmp/  # or /dev/sda2
btrfs subvolume create /mnt/tmp/@
btrfs subvolume create /mnt/tmp/@home
btrfs subvolume create /mnt/tmp/@var
mount -o compress=zstd,subvol=@ /dev/nvme0n1p2 /mnt/root/  # or /dev/sda2
mkdir -p /mnt/root/efi/ /mnt/root/home/ /mnt/root/var/
mount -o compress=zstd,subvol=@home /dev/nvme0n1p2 /mnt/root/home/  # or /dev/sda2
mount -o compress=zstd,subvol=@var /dev/nvme0n1p2 /mnt/root/var/  # or /dev/sda2
mount /dev/nvme0n1p1 /mnt/root/efi/  # or /dev/sda1
```

If you don't use btrfs just do:

```bash
mkdir -p /mnt/root/
mount /dev/nvme0n1p2 /mnt/root/  # or /dev/sda2
mkdir -p /mnt/root/efi/
mount /dev/nvme0n1p1 /mnt/root/efi/  # or /dev/sda1
```

If you have a swap partition do (replace partition number accordingly):

```bash
swapon /dev/nvme0n1p3  # or /dev/sda3
```

If you instead want to use a swapfile do (adjust the count parameter to at least your actual RAM size if you want to use hibernation):

```bash
dd if=/dev/zero of=/mnt/root/swapfile count=8192 bs=1MiB
chmod 600 /mnt/root/swapfile
mkswap /mnt/root/swapfile
swapon /mnt/root/swapfile
```


## Install base system

To use a fast mirror near to you for package download during the installation, edit the mirrorlist and put the server on top:

```bash
vim /etc/pacman.d/mirrorlist
# Use `dd` and `p` to cut and paste a line, `:wq` to save and quit
```

Finally install Arch Linux:

```bash
pacstrap /mnt/root/ base linux linux-firmware btrfs-progs
genfstab -U /mnt/root/ >> /mnt/root/etc/fstab
```

Then repeat the mirrorlist edit for your installed system:

```bash
vim /mnt/root/etc/pacman.d/mirrorlist
# Use `dd` and `p` to cut and paste a line, `:wq` to save and quit
```

---

Continue with [Setup base system](./base-setup.md).
