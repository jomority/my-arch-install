# Setup base system

Enter the base system installed in [Install base system](./base-install.md):

```bash
arch-chroot /mnt/root/
```

If your machine beeps annoyingly when pressing the tab key, disable the bell:

```bash
sed -i -E 's/^#?(set bell-style) .*$/\1 none/' /etc/inputrc
```


## Setup time

Setup timezone, hwclock and timesync.

```bash
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
systemctl enable systemd-timesyncd
```


## Setup locale

Enable locales:

```diff
--- /etc/locale.gen
+++ /etc/locale.gen
@@ -131,7 +131,7 @@
 #de_BE@euro ISO-8859-15  
 #de_CH.UTF-8 UTF-8  
 #de_CH ISO-8859-1  
-#de_DE.UTF-8 UTF-8  
+de_DE.UTF-8 UTF-8  
 #de_DE ISO-8859-1  
 #de_DE@euro ISO-8859-15  
 #de_IT.UTF-8 UTF-8  
@@ -175,7 +175,7 @@
 #en_SC.UTF-8 UTF-8  
 #en_SG.UTF-8 UTF-8  
 #en_SG ISO-8859-1  
-#en_US.UTF-8 UTF-8  
+en_US.UTF-8 UTF-8  
 #en_US ISO-8859-1  
 #en_ZA.UTF-8 UTF-8  
 #en_ZA ISO-8859-1
```

<details>
<summary>Command</summary>

```bash
sed -i \
	-e '/#de_DE.UTF-8/s/^#//' \
	-e '/#en_US.UTF-8/s/^#//' \
	/etc/locale.gen
```
</details>

Generate locales and configure system locales:

```bash
locale-gen
cat > /etc/locale.conf <<EOF
LANG=en_US.UTF-8
LC_NUMERIC=de_DE.UTF-8
LC_TIME=de_DE.UTF-8
LC_MONETARY=de_DE.UTF-8
LC_PAPER=de_DE.UTF-8
LC_MEASUREMENT=de_DE.UTF-8
EOF
```

### Optional: ISO 8601

If you want to use the [ISO 8601](https://xkcd.com/1179/) date format do:

```diff
--- /etc/locale.gen
+++ /etc/locale.gen
@@ -156,7 +156,7 @@
 #en_BW ISO-8859-1  
 #en_CA.UTF-8 UTF-8  
 #en_CA ISO-8859-1  
-#en_DK.UTF-8 UTF-8  
+en_DK.UTF-8 UTF-8  
 #en_DK ISO-8859-1  
 #en_GB.UTF-8 UTF-8  
 #en_GB ISO-8859-1
--- /etc/locale.conf
+++ /etc/locale.conf
@@ -1,6 +1,6 @@
 LANG=en_US.UTF-8
 LC_NUMERIC=de_DE.UTF-8
-LC_TIME=de_DE.UTF-8
+LC_TIME=en_DK.UTF-8
 LC_MONETARY=de_DE.UTF-8
 LC_PAPER=de_DE.UTF-8
 LC_MEASUREMENT=de_DE.UTF-8
```

<details>
<summary>Command</summary>

```bash
sed -i '/#en_DK.UTF-8/s/^#//' /etc/locale.gen
sed -i '/^LC_TIME=/s/=.*$/=en_DK.UTF-8/' /etc/locale.conf
```
</details>

And regenerate locales:

```bash
locale-gen
```


## Setup hostname

Replace `<HOSTNAME>` with your desired hostname.

```bash
hostname=<HOSTNAME>
echo $hostname > /etc/hostname
cat >> /etc/hosts <<EOF
127.0.0.1       localhost
127.0.1.1       $hostname.localdomain $hostname
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
EOF
```


## Ensure pacman keys are up to date

```bash
pacman -Syu archlinux-keyring
```


## Setup NetworkManager with systemd-resolved

```bash
yes | pacman -S --asdeps iptables-nft
pacman -S networkmanager systemd-resolvconf
exit
ln -sf /run/systemd/resolve/stub-resolv.conf /mnt/root/etc/resolv.conf
arch-chroot /mnt/root/
systemctl enable NetworkManager systemd-resolved
```

We need to briefly exit the chroot to link `/etc/resolv.conf`, because it is "over"-mounted in the chroot.


### Optional: firewall

If you want to use a firewall (most useful for mobile devices that visit untrusted networks):

```bash
pacman -S firewalld
systemctl enable firewalld
```


## Set root password

The root login can be disabled later when we have a user, but we need it for now.

```bash
passwd
```


## Optional: Enable resume (for hibernate)

It doesn't hurt to do this even if you don't plan on using hibernate.
Configure mkinitcpio:

```diff
--- /etc/mkinitcpio.conf
+++ /etc/mkinitcpio.conf
@@ -49,7 +49,7 @@
 #
 ##   NOTE: If you have /usr on a separate partition, you MUST include the
 #    usr, fsck and shutdown hooks.
-HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)
+HOOKS=(base systemd autodetect modconf block filesystems keyboard fsck)
 
 # COMPRESSION
 # Use this to compress the initramfs image. By default, zstd compression
```

<details>
<summary>Command</summary>

```bash
sed -i '/^HOOKS/s/udev/systemd/' /etc/mkinitcpio.conf
```
</details>

*Note:* This could also be done with the "resume" hook, see the [ArchWiki](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Configure_the_initramfs).

Regenerate initial ramdisk:

```bash
mkinitcpio -P
```

---

Continue with [Install bootloader](./bootloader.md)
