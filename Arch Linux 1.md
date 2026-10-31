# My Arch Linux VM Install

**Author:** Natalie **Host:** Windows PC, Oracle VirtualBox **ISO:** `archlinux-2026.10.01-x86_64.iso` (from archlinux.org/download) **Final result:** Arch Linux with the LXQt desktop, zsh, SSH, a colored terminal, custom aliases, two sudo users (me and codi), and a boot straight into the GUI.

This is the document I wrote for myself so I can rebuild this VM quickly. It is not a copy of the Arch wiki. It shows what I did, every command with a note on what it does, the problems I ran into, and how I fixed them. I followed the Arch Installation Guide wiki by hand and did not use `archinstall`.

---


## 1. Creating the VM in VirtualBox

|Setting|What I used|
|---|---|
|Name|Arch Linux|
|Type / Version|Linux / Arch Linux (64-bit)|
|Base memory|4096 MB|
|Processors|2|
|Disk|about 26 GB, dynamically allocated (Pre-Allocate unchecked)|
|Enable EFI|Checked|
|Skip Unattended Installation|Checked|
|Video memory|128 MB|
|Graphics controller|VMSVGA|
|Optical drive|Arch ISO attached|

**Problem I hit:** The wizard summary said "Proceed with Unattended Installation: true." Arch is not supported by that feature, and I wanted to install by hand anyway. **How I fixed it:** I clicked Back and checked **Skip Unattended Installation**. **Next time:** Read the summary screen before clicking Finish.

## 2. Booting the ISO and checking the environment

I booted the first menu entry, **Arch Linux install medium (x86_64, UEFI)**.

**Problem I hit:** I could not click inside the VM window. The live ISO is text-only, so the mouse does nothing there. If VirtualBox captures the mouse, the **right Ctrl** key (the Host Key) gives it back to Windows.

```bash
cat /sys/firmware/efi/fw_platform_size   # printed 64, so I booted in UEFI mode
ping -c 3 archlinux.org                  # internet works (VirtualBox NAT needed no setup)
timedatectl                              # clock is synchronized
lsblk                                    # list disks so I can find the right one 
```

The disk was named sda*

## 3. Partitioning the disk

```bash
fdisk /dev/sda   # open fdisk on my virtual disk
```

Inside fdisk:

|Keys|What it does|
|---|---|
|`g`|New GPT partition table|
|`n`, Enter, Enter, `+1G`|Partition 1, 1 GB|
|`t`, `1`|Set type to EFI System|
|`n`, Enter, Enter, `+2G`|Partition 2, 2 GB|
|`t`, `2`, `19`|Set type to Linux swap|
|`n`, Enter, Enter, Enter|Partition 3, the rest of the disk|
|`p`|Print the table to check it|
|`w`|Write and exit (nothing is saved until this)|

My layout:

|Partition|Size|Type|Used for|
|---|---|---|---|
|sda1|1 GB|EFI System|/boot|
|sda2|2 GB|Linux swap|swap|
|sda3|about 23.4 GB|Linux filesystem|/|

If I make a mistake before `w`, I type `q` to quit without saving and start over.

## 4. Formatting and mounting

```bash
mkfs.fat -F 32 /dev/sda1            # FAT32 for the EFI partition
mkswap /dev/sda2                    # prepare the swap partition
mkfs.ext4 /dev/sda3                 # ext4 for the root partition

mount /dev/sda3 /mnt                # mount root first
mount --mkdir /dev/sda1 /mnt/boot   # mount EFI at /mnt/boot (creates the folder)
swapon /dev/sda2                    # turn swap on
lsblk                               # check that the mount points look right
```

## 5. Installing the base system

```bash
pacstrap -K /mnt base linux linux-firmware nano networkmanager grub efibootmgr sudo
```

- `base linux linux-firmware`: the minimum bootable system
- `nano`: a simple text editor
- `networkmanager`: so networking works after the first reboot
- `grub efibootmgr`: the bootloader and the tool that registers it with UEFI
- `sudo`: needed for my sudo users later

I put the networking, bootloader, and sudo packages in this one command so I would not have to install them later.

```bash
genfstab -U /mnt >> /mnt/etc/fstab   # save the partition layout so it mounts at boot
cat /mnt/etc/fstab                   # should list /, /boot, and swap
arch-chroot /mnt                     # step inside the new system
```

## 6. Configuring the system (inside the chroot)

```bash
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime   # timezone: Central
hwclock --systohc                                           # sync the hardware clock

nano /etc/locale.gen                  # remove the # in front of: en_US.UTF-8 UTF-8
locale-gen                            # generate the locale
echo "LANG=en_US.UTF-8" > /etc/locale.conf

echo "archvm" > /etc/hostname         # hostname
passwd                                # root password

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB   # install GRUB to the EFI partition
grub-mkconfig -o /boot/grub/grub.cfg                                          # generate the GRUB config

systemctl enable NetworkManager       # start networking at boot (capital N and M)
```

I also added a safety net because VirtualBox sometimes forgets the GRUB entry and drops to a UEFI shell:

```bash
mkdir -p /boot/EFI/BOOT
cp /boot/EFI/GRUB/grubx64.efi /boot/EFI/BOOT/BOOTX64.EFI   # fallback boot file
```

Then I left the chroot and shut down:

```bash
exit            # leave the chroot
umount -R /mnt  # unmount everything cleanly
poweroff        # shut down the VM
```

**Problem I hit:** When I went to remove the ISO, I clicked the button that deletes the optical drive. VirtualBox warned that I would not be able to insert disks or Guest Additions without it. **How I fixed it:** I clicked **Cancel**. The correct way is Settings > Storage > click the ISO under Controller: IDE > click the disc icon in the Attributes panel > **Remove Disk from Virtual Drive**. That ejects the disc and keeps the drive. I did it with the VM powered off.

## 7. First boot and creating users

I booted into `archvm login:` and logged in as `root`.

```bash
ping -c 3 archlinux.org   # NetworkManager brought the network up on its own

useradd -m -G wheel -s /bin/bash natalie   # -m makes a home folder, wheel group will get sudo
passwd natalie
useradd -m -G wheel -s /bin/bash codi
passwd codi

EDITOR=nano visudo   # uncomment the line: %wheel ALL=(ALL:ALL) ALL
```

I used `visudo` because it checks the file for mistakes before saving, which is safer than editing `/etc/sudoers` directly.

**Problem I hit:** My password was rejected, and then `sudo whoami` kept saying "Sorry, try again." **What I figured out:**

- Plain `whoami` printing `natalie` is correct, because it shows the current user. The real sudo test is `sudo whoami`, which should print `root`.
- "Sorry, try again" means the password was wrong. It also means sudo is working, because a user who was not allowed to use sudo would see a different message.
- Arch also locks an account for 10 minutes after 3 failed attempts (faillock). During that time even the correct password fails, and resetting the password does not clear the lock. This was important to note as I kept locking myself out and was wondering why my passwords were never going through.

**How I fixed it** (as root):

```bash
faillock --user natalie --reset   # clear the lockout
faillock --user codi --reset
passwd natalie                    # set a simple password, typed slowly with the top-row number keys
```

After that, `su - natalie` followed by `sudo whoami` printed `root`. **Next time:** Use a simple password first and change it later.

## 8. Shell, SSH, and colors

```bash
pacman -S zsh zsh-completions openssh   # install zsh, its completions, and SSH
chsh -s /usr/bin/zsh natalie            # make zsh my login shell
chsh -s /usr/bin/zsh codi               # and codi's
systemctl enable --now sshd             # start SSH now AND at every boot
systemctl status sshd                   # should say active (running)
```

**Problem I hit:** I ran `systemctl enable sshd` first, and `systemctl status sshd` said inactive. **What I learned:** `enable` only makes a service start at the _next boot_. `start` starts it right now. `enable --now` does both.

The first time zsh opens it shows a setup wizard. I pressed `q` to skip it, because I wrote my own config.

My `~/.zshrc`:

```zsh
autoload -Uz colors && colors            # turn on color support
autoload -Uz compinit && compinit        # turn on tab completion
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f$ ' # green user@host, blue current folder

alias ls='ls --color=auto'       # colored file listing
alias ll='ls -lah --color=auto'  # long listing, hidden files, readable sizes
alias grep='grep --color=auto'   # colored search matches
alias ..='cd ..'                 # go up one directory
alias update='sudo pacman -Syu'  # full system update
alias ports='ss -tulpn'          # show listening ports
```

```bash
source ~/.zshrc                              # load the config now
cp /home/natalie/.zshrc /home/codi/.zshrc    # (as root) same setup for codi
chown codi:codi /home/codi/.zshrc            # give codi ownership of the file
```

## 9. Desktop environment and booting into the GUI

```bash
pacman -S xorg-server lxqt sddm virtualbox-guest-utils xdg-utils ttf-dejavu
systemctl enable sddm          # graphical login at boot, so the system boots into the GUI
systemctl enable vboxservice   # VirtualBox guest utilities (screen resize, mouse, clipboard)
reboot
```

- `xorg-server`: the display server
- `lxqt`: the desktop (a package group, so I pressed Enter to accept everything)
- `sddm`: the graphical login screen
- `virtualbox-guest-utils`: screen resizing, mouse integration, and clipboard
- `ttf-dejavu`: fonts, so the desktop is readable

**Problem I hit:** After the reboot I saw "Arch Linux, Advanced options for Arch Linux, UEFI Firmware Settings" and thought LXQt was missing. **What it was:** That is the **GRUB boot menu**, which appears before Linux loads. I pressed Enter on **Arch Linux**, and the SDDM login screen came up next. The LXQt session picker is on the SDDM screen. **Next time:** GRUB chooses the operating system, and SDDM chooses the desktop session.

## 10. Verifying everything works

```bash
echo $SHELL                 # /usr/bin/zsh
systemctl get-default       # graphical.target
systemctl is-active sshd    # active
```


## 11. Choices I made (required vs. optional)

|Choice|Required or optional?|What I picked and why|
|---|---|---|
|Manual install, not `archinstall`|Required by the assignment|Followed the Arch wiki by hand|
|UEFI (EFI enabled in VirtualBox)|Had to pick one|UEFI is the modern way, and the wiki's GRUB steps assume it|
|GPT partition table|Required with UEFI|`g` in fdisk|
|2 GB swap partition|Optional|Added a little safety for a small VM|
|EFI mounted at `/mnt/boot`|Optional (`/efi` also works)|Simpler GRUB setup|
|ext4 for root|Optional|Default, stable, simple|
|GRUB bootloader|Optional (systemd-boot also works)|Most common and best documented|
|NetworkManager|Needed some networking|Easy to enable at boot|
|EFI fallback copy (`BOOTX64.EFI`)|Optional|Safety net for VirtualBox forgetting the GRUB entry|
|Desktop: LXQt|Assignment minimum|Lightweight, uses little disk space|
|Display manager: SDDM|Needed for a graphical login|Works well with LXQt|
|Shell: zsh|Required (anything but bash)|It is highly backward-compatible with bash, so my existing bash knowledge still works|
|SSH: openssh|Required|`systemctl enable --now sshd`|
|Users: natalie and codi in `wheel`|Required|`wheel` gets sudo through `visudo`|
|Timezone and locale|Personal choice|America/Chicago, en_US.UTF-8|
|Hostname: `archvm`|Personal choice|Easy to recognize at the login prompt|

## 12. Requirements checklist

|#|Requirement|Done|
|---|---|---|
|1|Desktop environment (LXQt)|Yes|
|2|User accounts for me and codi with sudo|Yes|
|3|Shell other than bash (zsh)|Yes|
|4|SSH installed and running|Yes|
|5|Terminal color coding|Yes|
|6|Boots to the GUI|Yes|
|7|Aliases in `.zshrc`|Yes|

## 13. What I would do differently next time

1. Pick the right ISO for my CPU first.
2. Check **Skip Unattended Installation** while creating the VM.
3. Use a simple password at first and change it once everything works.
4. Copy the EFI fallback file before the first reboot.
5. Use `systemctl enable --now` so services start immediately.
6. Take screenshots during the install, not only at the end.
7. Take a VirtualBox snapshot once the system works, so I can restore it quickly if I break something.
8. Shut down from inside the VM (`sudo poweroff` or the desktop's Shutdown) instead of closing the window.

## 14. What I learned

- How a Linux system fits together: partitions, filesystems, the bootloader, systemd, users, and services.
- That the live ISO and the installed system are separate, and that `arch-chroot` is how I work "inside" the new one.
- That an error message can look scary but have a simple cause. "Sorry, try again" looked like a broken account, but it was a typo and a temporary lockout.
- That writing notes while installing made recovering from mistakes much faster.