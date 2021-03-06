# Install bootloader

Prerequisites: You entered (chroot) the system installed in [Install base system](./base-install.md) and have done the steps in [Setup base system](./base-setup.md)

I will leave you the choice between GRUB and systemd-boot.
GRUB supports more scenarios and still is the default in most other distributions, therefore it is easier to troubleshoot using search engines.
Systemd-boot is simpler and more modern, but can't do some special things, for example boot a Windows system from another **physical** drive.
It is possible that I won't "support" GRUB in the future anymore.

*Hint:* Don't forget the [Cleanup & Reboot](#cleanup--reboot) step at the bottom.


## Common steps

Install the respective microcode package for your processor:

```bash
pacman -S intel-ucode
pacman -S amd-ucode
```

Install `efibootmgr`:

```bash
pacman -S efibootmgr
```


## Option 1: GRUB


### Install GRUB

You can replace the `bootloader-id` with anything you want.
It will show up in the BIOS/UEFI menu.

```bash
pacman -S grub
grub-install --target=x86_64-efi --efi-directory=/efi/ --bootloader-id=archlinux
```


### Optional: Support for booting other OSes

If you want GRUB to detect OSes on other partitions and drives do:

```bash
pacman -S --asdeps os-prober
```

```diff
--- /etc/default/grub
+++ /etc/default/grub
@@ -60,4 +60,4 @@
 # documentation on GRUB_DISABLE_OS_PROBER, if still want to enable this
 # functionality install os-prober and uncomment to detect and include other
 # operating systems.
-#GRUB_DISABLE_OS_PROBER=false
+GRUB_DISABLE_OS_PROBER=false
```

<details>
<summary>Command</summary>

```bash
sed -i -E 's/^#?(GRUB_DISABLE_OS_PROBER)=.*$/\1=false/' /etc/default/grub
```
</details>


### Optional: Hibernation support

If you want hibernation to work GRUB has to tell the kernel at boot where to find the swap space.


#### Swap partition

If you use a swap partition you just need to find out its UUID and add it to the kernel cmdline (replace `<SWAP_UUID>`):

```bash
findmnt -st swap -no UUID
```

```diff
--- /etc/default/grub
+++ /etc/default/grub
@@ -3,7 +3,7 @@
 GRUB_DEFAULT=0
 GRUB_TIMEOUT=5
 GRUB_DISTRIBUTOR="Arch"
-GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
+GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet resume=UUID=<SWAP_UUID>"
 GRUB_CMDLINE_LINUX=""
 
 # Preload both GPT and MBR modules so that they are not missed
```

<details>
<summary>Command</summary>

```bash
swap_uuid=$(findmnt -st swap -no UUID)
sed -i '/GRUB_CMDLINE_LINUX_DEFAULT/s/"$/ resume=UUID='$swap_uuid'"/' /etc/default/grub
```
</details>


#### Swap file

If you use a swap file you need to find out the UUID of its containing partition, the start offset and add that to the kernel cmdline (replace `<SWAP_UUID>` and `<SWAP_OFFSET>`):

```bash
findmnt -no UUID -T /swapfile
filefrag -v /swapfile | awk '$1 == "0:" { $0=$4; sub(/\.*$/, ""); print }'
```

```diff
--- /etc/default/grub
+++ /etc/default/grub
@@ -3,7 +3,7 @@
 GRUB_DEFAULT=0
 GRUB_TIMEOUT=5
 GRUB_DISTRIBUTOR="Arch"
-GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
+GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet resume=UUID=<SWAP_UUID> resume_offset=<SWAP_OFFSET>"
 GRUB_CMDLINE_LINUX=""
 
 # Preload both GPT and MBR modules so that they are not missed
```

<details>
<summary>Command</summary>

```bash
root_uuid=$(findmnt -no UUID -T /swapfile)
swap_offset=$(filefrag -v /swapfile | awk '$1 == "0:" { $0=$4; sub(/\.*$/, ""); print }')
sed -i '/GRUB_CMDLINE_LINUX_DEFAULT/s/"$/ resume=UUID='$root_uuid' resume_offset='$swap_offset'"/' /etc/default/grub
```
</details>


### Generate GRUB config

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```


## Option 2 (Advanced): systemd-boot


### Install systemd-boot

```bash
bootctl install
systemctl enable systemd-boot-update.service
```


### Assemble cmdline

The kernel needs to know on which partition it will find the rootfs.
The other parameters reduce verbosity and flickering at boot.

```bash
root_uuid=$(findmnt -no UUID -T /)
echo -n "root=UUID=$root_uuid quiet bgrt_disable" > /etc/kernel/cmdline
```


#### Maybe: If using Btrfs for the root partition

The kernel needs to know on which subvolume it will find the rootfs.

```bash
echo -n " rootflags=subvol=@" >> /etc/kernel/cmdline
```


#### Optional: Hibernation support

If you want hibernation to work systemd-boot has to tell the kernel at boot where to find the swap space.


##### Swap partition

If you use a swap partition you just need to find out its UUID and add it to the kernel cmdline:

```bash
swap_uuid=$(findmnt -st swap -no UUID)
echo -n " resume=UUID=$swap_uuid" >> /etc/kernel/cmdline
```


##### Swap file

If you use a swap file you need to find out the UUID of its containing partition, the start offset and add that to the kernel cmdline:

```bash
root_uuid=$(findmnt -no UUID -T /swapfile)
swap_offset=$(filefrag -v /swapfile | awk '$1 == "0:" { $0=$4; sub(/\.*$/, ""); print }')
echo -n " resume=UUID=$root_uuid resume_offset=$swap_offset" >> /etc/kernel/cmdline
```


### Generate UKI

We use [Mkinitcpio](https://wiki.archlinux.org/title/Mkinitcpio) so that besides creating the initial ramdisk, it will also create [unified kernel images](https://wiki.archlinux.org/title/Unified_kernel_image) (UKI), so that systemd-boot can automatically boot them.

Configure mkinitcpio:

```diff
--- /etc/mkinitcpio.d/linux.preset
+++ /etc/mkinitcpio.d/linux.preset
@@ -2,13 +2,16 @@
 
 ALL_config="/etc/mkinitcpio.conf"
 ALL_kver="/boot/vmlinuz-linux"
+ALL_microcode=(/boot/*-ucode.img)
 
 PRESETS=('default' 'fallback')
 
 #default_config="/etc/mkinitcpio.conf"
 default_image="/boot/initramfs-linux.img"
-#default_options=""
+default_efi_image="/efi/EFI/Linux/archlinux-linux.efi"
+default_options=" --splash /usr/share/systemd/bootctl/splash-arch.bmp"
 
 #fallback_config="/etc/mkinitcpio.conf"
 fallback_image="/boot/initramfs-linux-fallback.img"
-fallback_options="-S autodetect"
+fallback_efi_image="/efi/EFI/Linux/archlinux-linux-fallback.efi"
+fallback_options="-S autodetect --splash /usr/share/systemd/bootctl/splash-arch.bmp"
```

<details>
<summary>Command</summary>

```bash
sed -i -E \
	-e '/^ALL_kver=/a ALL_microcode=(/boot/*-ucode.img)' \
	-e '/^default_image=/a default_efi_image="/efi/EFI/Linux/archlinux-linux.efi"' \
	-e 's;^#?(default_options=".*)"$;\1 --splash /usr/share/systemd/bootctl/splash-arch.bmp";' \
	-e '/^fallback_image=/a fallback_efi_image="/efi/EFI/Linux/archlinux-linux-fallback.efi"' \
	-e 's;^#?(fallback_options=".*)"$;\1 --splash /usr/share/systemd/bootctl/splash-arch.bmp";' \
	/etc/mkinitcpio.d/linux.preset
```
</details>

Generate UKI and set default boot entry:

```bash
mkinitcpio -p linux
bootctl set-default archlinux-linux.efi
```


## Cleanup & Reboot

The Arch Linux system should now be bootable.
Clean everything up and reboot.
Make sure you boot in the new Arch Linux.
You can remove the archiso (USB/CD/...).

```bash
exit
swapoff -a
umount -R /mnt/root/ /mnt/tmp/
reboot
```

If everything works the system will start and you can login as root.

Don't forget to change your keyboard layout again if you did this previously using the `loadkeys` command.


### Maybe: Troubleshooting

On some systems the NVRAM entries are not stored persistently, probably due to bugs in the UEFI implementation.
Arch Linux will then probably not boot, because the BIOS/UEFI does not now about it.

On one system it helped to force the `-e 3` argument of `efibootmgr` by creating an override script:

```bash
cat > /usr/local/bin/efibootmgr <<EOF
#!/bin/sh

/usr/bin/efibootmgr -e 3 "$@"
EOF
chmod +x /usr/local/bin/efibootmgr
```

And the running the `grub-install` or `bootctl install` command again (see above).

On another system it magically worked after installing a Fedora on another partition.

---

Continue with [Install & configure basic things (user, mirrors, pacman, ...)](./basic.md)
