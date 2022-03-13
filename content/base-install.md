# Install base system

Boot into the live environment using UEFI.


## Optional: Set keyboard layout

If you don't want to use the default US keyboard layout during the installation change the layout by listing the available layouts and choosing one of them (example with standard German layout):

```bash
ls /usr/share/kbd/keymaps/**/*.map.gz
loadkeys de-latin1
```


## Create partitions

Use [gdisk](https://wiki.archlinux.org/title/GPT_fdisk#Create_a_partition_table_and_partitions) to partition the disk.
Create at least:

- A 512MiB *EFI system partition* partition
  - If it does not already exists. In that case DO NOT format it with the `mkfs.fat` command below.
- A *Linux x86-64 root* partition
- Optionally a *Linux swap* partition
  - If hibernation is desired, its size should be at least the RAM size, e.g. 8192 MiB for 8GB RAM
  - Alternatively you can use either no swap at all or a swapfile (this is not advised with btrfs)

*Hint:* This can also be done graphically from an already installed distro or [gparted live iso](https://gparted.org/download.php).
In GParted, use *fat32* for the *EFI system parition* partition and set the *boot* and *esp* flags.


## Setup network

Connect Ethernet cable or use [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl) for WiFi (replace `<DEVICE>` with the device name you got from the first command):

```bash
iwctl device list
iwctl station <DEVICE> scan
iwctl station <DEVICE> get-networks
iwctl station <DEVICE> connect $SSID
```

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

Format the EFI partition (replace partition number accordingly):

```bash
mkfs.fat /dev/nvme0n1p1  # or /dev/sda1
```

I will leave you the choice between ext4 and Btrfs as root filesystem.
Ext4 is the traditional linux file system still used by some distributions.
It does its job without many bells and whistles.
Btrfs is a newer filesystem with more features like compression, reflinks, subvolumes and snapshotting.
It is becoming more popular in the distribution landscape and can be considered mature for our purposes.

It will not be possible to do the chapter [Optional: Setup Snapper](./snapper.md) if you choose ext4.


### Option 1: Ext4 as root filesystem

Format the root partition (replace partition number accordingly):

```bash
mkfs.ext4 /dev/nvme0n1p2  # or /dev/sda2
```

Mount the partitions:

```bash
mkdir -p /mnt/root/
mount /dev/nvme0n1p2 /mnt/root/  # or /dev/sda2
mkdir -p /mnt/root/efi/
mount /dev/nvme0n1p1 /mnt/root/efi/  # or /dev/sda1
```


### Option 2 (Advanced): Btrfs as root filesystem

We create the Btrfs subvolumes `@`, `@home` and `@var` for `/`, `/home/` and `/var/` respectively.
While it is not strictly necessary to use Btrfs per se, this makes it possible to snapshot them separately (see the [ArchWiki](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout) and the chapter [Optional: Setup Snapper](./snapper.md)).

```bash
mkfs.btrfs /dev/nvme0n1p2  # or /dev/sda2
mkdir -p /mnt/root/ /mnt/tmp/
mount -o compress=zstd /dev/nvme0n1p2 /mnt/tmp/  # or /dev/sda2
btrfs subvolume create /mnt/tmp/@
btrfs subvolume create /mnt/tmp/@home
btrfs subvolume create /mnt/tmp/@var
mount -o subvol=@ /dev/nvme0n1p2 /mnt/root/  # or /dev/sda2
mkdir -p /mnt/root/efi/ /mnt/root/home/ /mnt/root/var/
mount -o subvol=@home /dev/nvme0n1p2 /mnt/root/home/  # or /dev/sda2
mount -o subvol=@var /dev/nvme0n1p2 /mnt/root/var/  # or /dev/sda2
mount /dev/nvme0n1p1 /mnt/root/efi/  # or /dev/sda1
```


## Maybe: Prepare swap

If you have a swap partition do (replace partition number accordingly):

```bash
mkswap /dev/nvme0n1p3  # or /dev/sda3
swapon /dev/nvme0n1p3  # or /dev/sda3
```

If you instead want to use a swapfile instead do (adjust the count parameter to at least your actual RAM size if you want to use hibernation):

```bash
dd if=/dev/zero of=/mnt/root/swapfile count=8192 bs=1MiB
chmod 600 /mnt/root/swapfile
mkswap /mnt/root/swapfile
swapon /mnt/root/swapfile
```


## Install base system

Finally install Arch Linux:

```bash
pacstrap /mnt/root/ base linux linux-firmware btrfs-progs
genfstab -U /mnt/root/ >> /mnt/root/etc/fstab
```

---

Continue with [Setup base system](./base-setup.md).
