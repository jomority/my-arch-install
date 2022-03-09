# Install & configure basic things (user, mirrors, pacman, ...)

Prerequisites: You have done the steps in [Install bootloader](./bootloader.md) and rebooted into the Arch Linux system.


## Create user account

Replace `<USER>` with your desired user name.

```bash
user=<USER>
useradd -m $user
passwd $user
```

Install & configure `sudo` so we don't need the root account:

```bash
pacman -S sudo
sed -i '/wheel ALL=(ALL:ALL) ALL/s/^# *//' /etc/sudoers
visudo -c
usermod -aG wheel $user
```

Disable faillock (see [this issue](https://bugs.archlinux.org/task/67644)):

```bash
sed -i -E 's/^(# *)?(deny =).*$/\2 0/' /etc/security/faillock.conf
```

Logout and back in as `$user`.

```bash
logout  # Press Ctrl+D
```

After testing that sudo works, we can remove the password from the root account.
That will make login with root impossible.

```bash
sudo id
sudo passwd -l root
```


## Install essential packages

*Note:* These are packages that I consider essential. I am well aware that this is subjective.

```bash
sudo pacman -S \
	man-db \
	man-pages \
	vim \
	htop \
	ncdu \
	screen \
	openssh \
	nmap \
	bind \
	tcpdump \
	iperf3 \
	openbsd-netcat \
	fwupd \
	pacman-contrib \
	git \
	pkgstats
```

The last one, `pkgstats`, sends a weekly list of your installed packages (also AUR packages) to [pkgstats.archlinux.de](https://pkgstats.archlinux.de/packages).
If you are concerned about your privacy don't install it.


## Optional: SSH server

Disable root login:

```bash
sed -i -E 's/^#?(PermitRootLogin).*$/\1 no/' /etc/ssh/sshd_config
```

Disable Password login. You should use [SSH keys](https://wiki.archlinux.org/title/SSH_keys).

```bash
sed -i -E 's/^#?(PasswordAuthentication).*$/\1 no/' /etc/ssh/sshd_config
```

Enable & start:

```bash
sudo systemctl enable --now sshd.service
```


## Configure firewall zones

If you installed the [firewall](./base-setup.md#optional-firewall) it needs to know which network(s) is/are your home network and can be trusted.
Some open services are preconfigured.
Optionally, we will later add some ourself.

```bash
nmcli connection show
```

This will print out a list of all your networks.
The current active ones are green.
Choose the correct one and add it to the *home* zone:

```bash
nmcli connection modify "Wired connection 1" connection.zone home
```

This can be repeated with more networks.

You can also use other zones.
View all of them and their preconfigured rules using:

```bash
sudo firewall-cmd --list-all-zones
```


## kernel-modules-hook

This package ensures that if you update your kernel the modules for the running kernel remain at least until the next reboot.
This ensures, for example, that plug and play still works.

```bash
sudo pacman -S kernel-modules-hook
sudo systemctl daemon-reload
sudo systemctl enable linux-modules-cleanup.service
```


## Logrotate

Logrotate automatically rotates many logfiles under `/var/log/` so that they don't get too big.

```bash
sudo pacman -S logrotate
sudo systemctl enable --now logrotate.timer
```


## Trim for SSDs

If you have at least one SSD, you should enable the periodic trim service:

```bash
sudo systemctl enable --now fstrim.timer
```


## Reflector (automatic mirror selection)

To automatically filter and update the mirrorlist we use [reflector](https://wiki.archlinux.org/title/Reflector).
Feel free to change the filter options.

```bash
sudo pacman -S reflector
sudo tee -a /etc/xdg/reflector/reflector.conf >/dev/null <<EOF

# Overrides
--country Germany
--sort score
--age 12
--latest 200
--ipv6
EOF
sudo systemctl enable --now reflector.service reflector.timer
sudo sed -i -E 's;^#?(NoExtract.*)$;\1 etc/pacman.d/mirrorlist;' /etc/pacman.conf
```


## Configure pacman

Make pacman output more beautiful and the downloads faster:

```bash
sudo sed -i \
	-e '/^#Color/s/^#//' \
	-e '/^#VerbosePkgLists/s/^#//' \
	-e '/^#ParallelDownloads/s/^#//' \
	/etc/pacman.conf
```

Make makepkg faster by using all cores for building and compressing.

```bash
sudo sed -i -E \
	-e 's/^#?(MAKEFLAGS)=.*$/\1="-j$(nproc)"/' \
	-e '/^COMPRESSZST/s/\)$/ --threads=0)/' \
	/etc/makepkg.conf
```


### Optional: multilib repository

If you wan't to use wine and/or steam you need to enable the multilib repository:

```bash
sudo sed -i '/#\[multilib\]/{s/^#//;n;s/^#//}' /etc/pacman.conf
```


## Paru (AUR helper)

Install the `base-devel` group which is needed if we want to build (AUR) packages by ourself.
We install them as dependencies because we later install the [base-devel-meta](https://aur.archlinux.org/packages/base-devel-meta) AUR package which mimics what the [base](https://archlinux.org/packages/core/any/base/) package does for `base-devel`.

```bash
sudo pacman -S --asdeps base-devel
```

Build and install Paru:

```bash
git clone https://aur.archlinux.org/paru.git
cd paru/
makepkg -sri
cd ..
rm -r paru/
paru -S --asdeps bat
```

Install the mentioned `base-devel-meta` package:

```bash
paru -S base-devel-meta
```

## Install useful packages

*Note:* These are packages that I consider useful. I am well aware that this is subjective.

```bash
sudo pacman -S \
	neovim \
	tmux \
	progress \
	icdiff \
	ripgrep \
	fd \
	fzf \
	mosh \
	liboping \
	mtr \
	httpie \
	jq \
	whois \
	speedtest-cli \
	stress \
	pwgen \
	tree \
	up \
	downgrade \
	perl-file-mimeinfo \
	wl-clipboard \
	ntfs-3g \
	exfatprogs
```

What is the difference to the [essential packages](#install-essential-packages)?
I'm not completely sure, but there two distinctions:

- There are some AUR packages in the *useful packages*
- I wouldn't miss most of the *useful packages* on a server


## LTS kernel

It is useful to have a second kernel lying around if something breaks (not that it often happens).

```bash
paru -S linux-lts
```


### If bootloader GRUB

To boot the non-LTS kernel by default we need an additional package:

```bash
paru -S grub-linux-default-hook
sudo grub-mkconfig -o /boot/grub/grub.cfg
```


### If bootloader systemd-boot

We need to tell mkinitcpio to also create UKIs for the LTS kernel:

```bash
sed 's/linux/linux-lts/g; s/archlinux-lts/archlinux/g' /etc/mkinitcpio.d/linux.preset | sudo tee /etc/mkinitcpio.d/linux-lts.preset >/dev/null
sudo mkinitcpio -p linux-lts
```


## Optional: VPNs

If you want to use VPN services, install the packets supporting the corresponding protocols:

```bash
paru -S wireguard-tools
paru -S networkmanager-openvpn
paru -S networkmanager-openconnect
```


## Optional: Sensors

Is you want to be able to see you temperature, voltage and additional sensor data using the `sensors` command (also useful for some GUI programs) do:

```bash
paru -S lm_sensors
sudo sensors-detect --auto
```

If you miss some sensors, rerun `sensors-detect` without `--auto` and answer the questions differently (could be unsafe).


## Optional: Memtest

If you want a [Memtest](https://en.wikipedia.org/wiki/Memtest86) installed on your EFI partition do:

```bash
paru -S memtest86-efi
sudo memtest86-efi --install
```

Select GRUB or systemd-boot accordingly.

---

Continue with [Install Gnome & other GUI programs](./gui.md)
