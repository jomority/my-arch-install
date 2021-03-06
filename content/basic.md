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
usermod -aG wheel $user
EDITOR=vim visudo
```

```diff
--- /etc/sudoers
+++ /etc/sudoers
@@ -79,7 +79,7 @@
 root ALL=(ALL:ALL) ALL
 
 ## Uncomment to allow members of group wheel to execute any command
-# %wheel ALL=(ALL:ALL) ALL
+%wheel ALL=(ALL:ALL) ALL
 
 ## Same thing without a password
 # %wheel ALL=(ALL:ALL) NOPASSWD: ALL
```

<details>
<summary>Command</summary>

```bash
sed -i '/wheel ALL=(ALL:ALL) ALL/s/^# *//' /etc/sudoers
visudo -c
```
</details>

Disable faillock (see [this issue](https://bugs.archlinux.org/task/67644)):

```diff
--- /etc/security/faillock.conf
+++ /etc/security/faillock.conf
@@ -29,7 +29,7 @@
 # Deny access if the number of consecutive authentication failures
 # for this user during the recent interval exceeds n tries.
 # The default is 3.
-# deny = 3
+deny = 0
 #
 # The length of the interval during which the consecutive
 # authentication failures must happen for the user account
```

<details>
<summary>Command</summary>

```bash
sed -i -E 's/^(# *)?(deny =).*$/\2 0/' /etc/security/faillock.conf
```
</details>

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


## Maybe: Connect to WiFi

If you used WiFi previously, you need to connect to it again.
This time use [NetworkManager](https://wiki.archlinux.org/title/NetworkManager#nmcli_examples):

```bash
nmcli device wifi list
nmcli device wifi connect <SSID_OR_BSSID> password <PASSWORD>
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

```diff
--- /etc/ssh/sshd_config
+++ /etc/ssh/sshd_config
@@ -29,7 +29,7 @@
 # Authentication:
 
 #LoginGraceTime 2m
-#PermitRootLogin prohibit-password
+PermitRootLogin no
 #StrictModes yes
 #MaxAuthTries 6
 #MaxSessions 10
```

<details>
<summary>Command</summary>

```bash
sudo sed -i -E 's/^#?(PermitRootLogin).*$/\1 no/' /etc/ssh/sshd_config
```
</details>

Disable Password login. You should use [SSH keys](https://wiki.archlinux.org/title/SSH_keys).

```diff
--- /etc/ssh/sshd_config
+++ /etc/ssh/sshd_config
@@ -54,7 +54,7 @@
 #IgnoreRhosts yes
 
 # To disable tunneled clear text passwords, change to no here!
-#PasswordAuthentication yes
+PasswordAuthentication no
 #PermitEmptyPasswords no
 
 # Change to no to disable s/key passwords
```

<details>
<summary>Command</summary>

```bash
sudo sed -i -E 's/^#?(PasswordAuthentication).*$/\1 no/' /etc/ssh/sshd_config
```
</details>

Enable & start:

```bash
sudo systemctl enable --now sshd.service
```


## Maybe: Configure firewall zones

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
sudo systemctl enable linux-modules-cleanup.service
```


## Logrotate

Logrotate automatically rotates many logfiles under `/var/log/` so that they don't get too big.

```bash
sudo pacman -S logrotate
sudo systemctl enable --now logrotate.timer
```


## Maybe: Trim for SSDs

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
```

```diff
--- /etc/pacman.conf
+++ /etc/pacman.conf
@@ -26,7 +26,7 @@
 #IgnoreGroup =
 
 #NoUpgrade   =
-#NoExtract   =
+NoExtract   = etc/pacman.d/mirrorlist
 
 # Misc options
 #UseSyslog
```

<details>
<summary>Command</summary>

```bash
sudo sed -i -E 's;^#?(NoExtract.*)$;\1 etc/pacman.d/mirrorlist;' /etc/pacman.conf
```
</details>


## Configure pacman

Make pacman output more beautiful and the downloads faster:

```diff
--- /etc/pacman.conf
+++ /etc/pacman.conf
@@ -30,11 +30,11 @@
 
 # Misc options
 #UseSyslog
-#Color
+Color
 #NoProgressBar
 CheckSpace
-#VerbosePkgLists
-#ParallelDownloads = 5
+VerbosePkgLists
+ParallelDownloads = 5
 
 # By default, pacman accepts packages signed by keys that its local keyring
 # trusts (see pacman-key and its man page), as well as unsigned packages.
```

<details>
<summary>Command</summary>

```bash
sudo sed -i \
	-e '/^#Color/s/^#//' \
	-e '/^#VerbosePkgLists/s/^#//' \
	-e '/^#ParallelDownloads/s/^#//' \
	/etc/pacman.conf
```
</details>

Make makepkg faster by using all cores for building and compressing.

```diff
--- /etc/makepkg.conf
+++ /etc/makepkg.conf
@@ -46,7 +46,7 @@
 LTOFLAGS="-flto=auto"
 #RUSTFLAGS="-C opt-level=2"
 #-- Make Flags: change this for DistCC/SMP systems
-#MAKEFLAGS="-j2"
+MAKEFLAGS="-j$(nproc)"
 #-- Debugging flags
 DEBUG_CFLAGS="-g"
 DEBUG_CXXFLAGS="$DEBUG_CFLAGS"
@@ -137,7 +137,7 @@
 COMPRESSGZ=(gzip -c -f -n)
 COMPRESSBZ2=(bzip2 -c -f)
 COMPRESSXZ=(xz -c -z -)
-COMPRESSZST=(zstd -c -z -q -)
+COMPRESSZST=(zstd -c -z -q - --threads=0)
 COMPRESSLRZ=(lrzip -q)
 COMPRESSLZO=(lzop -q)
 COMPRESSZ=(compress -c -f)
```

<details>
<summary>Command</summary>

```bash
sudo sed -i -E \
	-e 's/^#?(MAKEFLAGS)=.*$/\1="-j$(nproc)"/' \
	-e '/^COMPRESSZST/s/\)$/ --threads=0)/' \
	/etc/makepkg.conf
```
</details>


### Optional: multilib repository

If you wan't to use wine and/or steam you need to enable the multilib repository:

```diff
--- /etc/pacman.conf
+++ /etc/pacman.conf
@@ -90,8 +90,8 @@
 #[multilib-testing]
 #Include = /etc/pacman.d/mirrorlist
 
-#[multilib]
-#Include = /etc/pacman.d/mirrorlist
+[multilib]
+Include = /etc/pacman.d/mirrorlist
 
 # An example of a custom package repository.  See the pacman manpage for
 # tips on creating your own repositories.
```

<details>
<summary>Command</summary>

```bash
sudo sed -i '/#\[multilib\]/{s/^#//;n;s/^#//}' /etc/pacman.conf
```
</details>

Refresh package database:

```bash
sudo pacman -Syu
```


## Paru (AUR helper)

Install the `base-devel` group which is needed if we want to build (AUR) packages by ourself.
We install them as dependencies because we later install the [base-devel-meta](https://aur.archlinux.org/packages/base-devel-meta) AUR package which mimics what the [base](https://archlinux.org/packages/core/any/base/) package does for `base-devel`.

```bash
sudo pacman -S --asdeps --needed --noconfirm base-devel
```

Build and install Paru:

```bash
git clone https://aur.archlinux.org/paru.git
cd paru/
makepkg -sri --noconfirm
cd ..
rm -rf paru/
paru -S --asdeps bat
```

Install the mentioned `base-devel-meta` package:

```bash
paru -S base-devel-meta
```

## Install useful packages

*Note:* These are packages that I consider useful. I am well aware that this is subjective.

```bash
paru -S \
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
I'm not completely sure, but there are two distinctions:

- There are some AUR packages in the *useful packages*
- I wouldn't miss most of the *useful packages* on a server


## LTS kernel

It is useful to have a second kernel lying around if something breaks (not that it often happens).

```bash
paru -S linux-lts
```


### Maybe: If bootloader GRUB

To boot the non-LTS kernel by default we need an additional package:

```bash
paru -S grub-linux-default-hook
sudo grub-mkconfig -o /boot/grub/grub.cfg
```


### Maybe: If bootloader systemd-boot

We need to tell mkinitcpio to also create UKIs for the LTS kernel:

```diff
--- /etc/mkinitcpio.d/linux-lts.preset
+++ /etc/mkinitcpio.d/linux-lts.preset
@@ -2,13 +2,16 @@
 
 ALL_config="/etc/mkinitcpio.conf"
 ALL_kver="/boot/vmlinuz-linux-lts"
+ALL_microcode=(/boot/*-ucode.img)
 
 PRESETS=('default' 'fallback')
 
 #default_config="/etc/mkinitcpio.conf"
 default_image="/boot/initramfs-linux-lts.img"
-#default_options=""
+default_efi_image="/efi/EFI/Linux/archlinux-linux-lts.efi"
+default_options=" --splash /usr/share/systemd/bootctl/splash-arch.bmp"
 
 #fallback_config="/etc/mkinitcpio.conf"
 fallback_image="/boot/initramfs-linux-lts-fallback.img"
-fallback_options="-S autodetect"
+fallback_efi_image="/efi/EFI/Linux/archlinux-linux-lts-fallback.efi"
+fallback_options="-S autodetect --splash /usr/share/systemd/bootctl/splash-arch.bmp"
```

<details>
<summary>Command</summary>

```bash
sed 's/linux/linux-lts/g; s/archlinux-lts/archlinux/g' /etc/mkinitcpio.d/linux.preset | \
	sudo tee /etc/mkinitcpio.d/linux-lts.preset >/dev/null
```
</details>

Regenerate initial ramdisk:

```bash
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

Everything should be detected correctly. Select GRUB (`3`) or systemd-boot (`4`) accordingly.

---

Continue with [Install Gnome & other GUI programs](./gui.md)
