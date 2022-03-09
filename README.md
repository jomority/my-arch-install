# my-arch-install

This repo contains instructions on how I install my (private) Arch Linux systems.
It exists for:

- My poor memory that cannot remember everything I did in the past
- Some friends that like my support
- Anyone who wants some inspiration

The instructions are targeted at people who are somewhat proficient with linux.
If you don't understand some things, please refer to the [ArchWiki](https://wiki.archlinux.org/).
I apologize for any lack of clarity, typos and out-of-date information.
The main information source should always be the [official installation guide](https://wiki.archlinux.org/title/Installation_guide).

Feel free to submit pull requests, but please note that I will be very opinionated about what to include.
After all, this is how I set my private systems up.


## Table of contents

- [Install base system](./content/base-install.md)
- [Setup base system](./content/base-setup.md)
- [Install bootloader](./content/bootloader.md)
- [Install & configure basic things (user, mirrors, pacman, ...)](./content/basic.md)
- [Install Gnome & other GUI programs](./content/gui.md)
- [Optional: Setup Snapper](./content/snapper.md)


### Windows dual boot

If you want to dual boot with Windows, have a look [here](./content/windows.md).


## Options / Decisions

This list contains some choices I made for components, configurations or requirements.
Some of these things can be omitted (parentheses), others need to be replaced for a working system.
Some are easier to replace (e.g. timezone) than others (e.g. bootloader).

- German timezone
- A mixture of German & US locale with ISO 8601 dates
- (Hibernation (i.e. suspend-to-disk))
- NetworkManager with systemd-resolved
- (Btrfs & Snapper)
- UEFI
- EFI partition mounted on `/efi/`
- systemd-boot / GRUB
- Paru (AUR helper)
- Gnome & some Gnome programs
- Install packages non-explicitly (as dependency) if they are an optional dependency on another package and only installed because of that package
  - This will remove them if you remove the "required by" package using the `-s` flag on `pacman -R`
  - Note that this will make them show up in `pacman -Qdtt` but not in `pacman -Qdt`
  - If you don't want this behaviour leave out the `--asdeps` flag on the `pacman -S` and `paru -S` commands


## License

This whole repository is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
