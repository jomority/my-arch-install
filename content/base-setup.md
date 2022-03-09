# Setup base system

Enter the base system installed in [Install base system](./base-install.md):

```bash
arch-chroot /mnt/root/
```

If your machine beeps annoyingly when pressing the tab key, disable the bell:

```bash
sed -i -E 's/^#?(set bell-style) .*$/\1 none/' /etc/inputrc
```


## Setup time and locale

Change the timezone and locales accordingly.

```bash
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
hwclock --systohc
systemctl enable systemd-timesyncd
sed -i \
	-e '/#de_DE.UTF-8/s/^#//' \
	-e '/#en_US.UTF-8/s/^#//' \
	/etc/locale.gen
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

```bash
sed -i '/#en_DK.UTF-8/s/^#//' /etc/locale.gen
locale-gen
sed -i '/^LC_TIME=/s/=.*$/=en_DK.UTF-8/' /etc/locale.conf
```


## Setup hostname

Replace `<HOSTNAME>` with your desired hostname.

```bash
hostname=<HOSTNAME>
echo "$hostname" > /etc/hostname
cat >> /etc/hosts <<EOF
127.0.0.1       localhost
127.0.1.1       $hostname.localdomain $hostname
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
EOF
```


## Setup NetworkManager with systemd-resolved

```bash
pacman -S networkmanager systemd-resolvconf
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
systemctl enable NetworkManager systemd-resolved
```

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

```bash
sed -i '/^HOOKS/s/udev/systemd/' /etc/mkinitcpio.conf
mkinitcpio -P
```

*Note:* This would also be possible with the "resume" hook, see the [ArchWiki](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Configure_the_initramfs).

---

Continue with [Install bootloader](./bootloader.md)
