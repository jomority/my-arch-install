# Install Gnome & other GUI programs

Prerequisites: You have done the steps in [Install & configure basic things (user, mirrors, pacman, ...)](./basic.md), at least created a user account and installed an AUR helper.


## Maybe: NVIDIA driver

If you have a NVIDIA GPU install the proprietary driver:

```bash
paru -S nvidia
```

To be able to run a Wayland session, you need to enable DRM KMS (see the [ArchWiki](https://wiki.archlinux.org/title/GDM#Wayland_and_the_proprietary_NVIDIA_driver)) by extending the kernel command line.
Depending on your bootloader execute the commands in one of the following sections.


### Maybe: If bootloader GRUB

```bash
sed -i '/GRUB_CMDLINE_LINUX_DEFAULT/s/"$/ nvidia-drm.modeset=1"/' /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg
```


### Maybe: If bootloader systemd-boot

```bash
echo -n " nvidia-drm.modeset=1" | sudo tee -a /etc/kernel/cmdline >/dev/null
sudo mkinitcpio -P
```


## Fonts

Install some font packages:

```bash
paru -S noto-fonts \
	noto-fonts-emoji \
	ttf-dejavu \
	adobe-source-han-sans-otc-fonts
```


## Install basic Gnome

```bash
paru -S --asdeps xdg-desktop-portal-gnome \
	power-profiles-daemon \
	wireplumber
paru -S gdm \
	gnome-control-center \
	gnome-tweaks \
	dconf-editor \
	gnome-keyring \
	gnome-backgrounds \
	gnome-themes-extra \
	qgnomeplatform \
	qgnomeplatform-qt6 \
	gnome-shell-extension-appindicator \
	xdg-desktop-portal \
	xdg-user-dirs
```


### Optional: Install extended Gnome features

- `gnome-remote-desktop`: VNC/RDP remote desktop
- `rygel`: DLNA/UPnP-AV MediaServer

```bash
paru -S --asdeps gnome-remote-desktop \
	rygel
```


#### Maybe: If firewall installed

You probably want these servers at least to be reachable in your *home* networks.

```bash
sudo firewall-cmd --add-service=vnc-server --add-service=rdp --zone=home --permanent
sudo firewall-cmd --add-service=vnc-server --add-service=rdp --zone=home
```

Unfortunately, I haven't found a rule for rygel yet :(
Just using port 5900/udp doesn't work.
Probably some more ports need to be opened.


## Install (more or less) essential Gnome programs

Feel free to omit programs or add other `gnome-*` programs.

```bash
paru -S nautilus \
	ffmpegthumbnailer \
	file-roller \
	gnome-terminal \
	gedit \
	eog \
	cheese \
	gnome-characters \
	gnome-clocks \
	gnome-calendar \
	gnome-contacts \
	gnome-maps
paru -S --asdeps gvfs-mtp gvfs-smb
```

When you get asked to choose between `jack2` and `pipewire-jack` I recommend `pipewire-jack`.
Especially if you want to use [PipeWire for audio](#optional-advanced-handle-audio-with-pipewire) anyway.


## Audio management software

```bash
paru -S alsa-utils pavucontrol
sudo alsactl store
```


### Optional (Advanced): Handle audio with PipeWire

If you want to use PipeWire to handle audio instead of PulseAudio:

```bash
paru -S --asdeps rtkit pipewire-alsa
paru -Rns pulseaudio-alsa
paru -S pipewire-jack pipewire-pulse
```

This asks to remove `pulseaudio` and `pulseaudio-bluetooth`.
Confirm it.

You can also install one ore more of the following patch bays:

```bash
paru -S patchmatrix
paru -S patchage
paru -S carla
paru -S helvum
```


## Hardware video acceleration

Especially useful for mobile devices to save power.
See the [ArchWiki](https://wiki.archlinux.org/title/Hardware_video_acceleration) for more information and older hardware.

```bash
paru -S gstreamer-vaapi
```

Depending on the GPU in your system execute the commands in one of the following sections.


### Maybe: Intel (i)GPU

```bash
paru -S --asdeps intel-media-driver
```


### Maybe: AMD (i)GPU

```bash
paru -S --asdeps libva-mesa-driver mesa-vdpau
echo "VDPAU_DRIVER=radeonsi" | sudo tee -a /etc/environment >/dev/null
```


### Maybe: Nvidia GPU

```bash
paru -S --asdeps libva-vdpau-driver
echo "VDPAU_DRIVER=nvidia" | sudo tee -a /etc/environment >/dev/null
```


## Enable & Reboot

```bash
sudo systemctl enable gdm
reboot
```

You should be greeted by a graphical login after the reboot.
Login and continue inside the graphical terminal.

Set you keyboard layout in the Gnome settings if you want.


## Printing

Install and run CUPS:

```bash
paru -S cups cups-pdf
sudo usermod -aG sys $USER
echo a4 | sudo tee -a /etc/papersize >/dev/null
sudo systemctl enable --now cups avahi-daemon
```

You can now add your printers in the Gnome settings.


## Optional: SMB (Windows) file sharing

Following these steps will enable a samba server, so that you can easily share folders with the network from the file manager like in Windows.

*Warning:* If you activate this you should have installed a [firewall](./base-setup.md#optional-firewall).
We will only open the ports in the (hopefully secure) *home* zone.

The `smbpasswd` command will ask for a password.
It can be different from the normal login password and will be used for authentication for non-public shares.
The command has to be executed for each user that wants to add their own shares.

Change the `NET` variable to the IP range of you home network.
You can also define multiple allowed ranges with a separated comma, e.g. `NET="192.168.0.0/255.255.255.0, 192.168.178.0/255.255.255.0"`

```bash
paru -S nautilus-share
sudo mkdir /var/lib/samba/usershares
sudo groupadd -r sambashare
sudo chown root:sambashare /var/lib/samba/usershares
sudo chmod 1770 /var/lib/samba/usershares
sudo usermod -aG sambashare $USER
NET="192.168.0.0/255.255.255.0"
sudo tee /etc/samba/smb.conf >/dev/null <<EOF
[global]
	workgroup = WORKGROUP
	server string = $HOST
	server role = standalone server
	hosts allow = $NET
	logging = systemd
	max log size = 50
	dns proxy = no
	load printers = no
	printing = bsd
	printcap name = /dev/null
	disable spoolss = yes
	show add printer wizard = no
	usershare path = /var/lib/samba/usershares
	usershare max shares = 100
	usershare allow guests = yes
	usershare owner only = yes
[homes]
	comment = Home Directories
	browseable = no
	writable = yes
[printers]
	comment = All Printers
	path = /usr/spool/samba
	browseable = no
	guest ok = no
	writable = no
	printable = yes
EOF
sudo smbpasswd -a $USER
sudo systemctl enable --now smb.service nmb.service
sudo firewall-cmd --add-service=samba --zone=home --permanent
sudo firewall-cmd --add-service=samba --zone=home
```

---

If you use Btrfs for your root partition you may continue with [Optional: Setup Snapper](./snapper.md).

TODO: Recommendations for other programs
