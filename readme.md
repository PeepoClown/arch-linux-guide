# Arch linux installation and configuration guide
## Intro
In this article, I describe the steps that I did during my arch installation(made with ❤️‍🩹). It's based on the guide
from the [official wiki](https://wiki.archlinux.org/title/Installation_guide). The latest version of system you can
download [here](https://archlinux.org/download/). Then you need to write this image to a flash drive, and boot on it.

## Check network connection
First you need to do after booting is checking your internet connection.

To get list of network interfaces use `ip link show` command.

Then if interface is down, you can up it like this `ip link set ${interface} up`.

Now, when interface successfully up, you need to dynamically configure you host or connect to wi-fi.
Wifi-menu have some ui:) And then check connection by ping google.
```shell
# For wired
dhcpcd
# For wireless
wifi-menu
# Check
ping -c 3 8.8.8.8
```

? If you want to get your private/public ip you can use this commands:
```shell
# For private
ip adress show | grep inet | grep ${interface}
# For public
wget -O - -q ifconfig.me/ip && echo ''
```

## Disk partitioning
I've been describing the classic way of disk management with root, boot, swap and home.

You can get list of disks configuration by command `fdisk -l`. As a partition tool I use `cfdisk`.
You need to pass for it argument a disk name, `/dev/sda` by default. Then you need to create partitions using GUI,
save them and quit.

### Partitions
| Partition  | Size  | Type             | Label |
|------------|-------|------------------|-------|
| /dev/sda1  | 50G   | Linux Filesystem | root  |
| /dev/sda2  | 100M  | EFI System       | efi   |
| /dev/sda3  | 16G   | Linux swap       | swap  |
| /dev/sda4  | other | Linux Filesystem | home  |

### Format partitions and mount
Root `/` partition:
```shell
mkfs.ext4 /dev/sda1 -L "root"
mount /dev/sda1 /mnt
```

EFI `/boot/efi` partition:
```shell
mkfs.vfat /dev/sda2
mkdir -p /mnt/boot/efi
mount /dev/sda2 /mnt/boot/efi
```

Swap partition:
```shell
mkswap /dev/sda3
swapon /dev/sda3
```

Home `/home` partition:
```shell
mkfs.ext4 /dev/sda4 -L "home"
mkdir /mnt/home
mount /dev/sda4 /mnt/home
```

## System installation
Now we're going to install kernel, base system package and VIM to root mount `/mnt`.

?You can select your installation mirror server by editing of file `/etc/pacman.d/mirrorList`. For Russia, I've
recommended to use [Yandex mirror](http://mirror.yandex.ru/archlinux/). You need to put it on head of file.

Let's install packages and generate the partition table:
```shell
pacstrap -i /mnt base base-devel linux linux-firmware vim
genfstab -U /mnt >> /mnt/etc/fstab
```

## System setup
After installation, you need to log in on root user. Use command `arch-chroot /mnt`.

### Locale and timezone
If you need specific keyboard uncomment locales in file `/etc/locale.gen`. English locale `en_US.UTF-8 UTF-8` and
Russian `ru_RU.UTF-8 UTF-8`. Then use this commands(example for Cyrillic):
```shell
locale-gen
loadkeys ru
setfont cyr-sun16
```
For right timezone set you need to create sym link of timezone file in `/usr/share/zoneinfo` to `/etc/localtime`
directory. Then sync hardware clock. Example for Moscow timezone:
```shell
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
hwclock --systohc
```

### System host(s)
To provide your own host edit `/etc/hostname` file. You need to add you hostname to it. Then edit `/etc/hosts` file
like this:
```text
127.0.0.1   localhost
::1         localhost
127.0.1.1   ${hostname}.localdomain	${hostname}
```

### Configure root and non-root users
For reset of root password you need use this command `passwd` and type new password two times. Let's create own user:
```shell
useradd -m -g users -G wheel -s /bin/bash ${username}
passwd ${username}
```
Then edit `/etc/sudoers` to allow your user execute any command. Uncomment line `# %wheel ALL=(ALL) ALL`.
Also add `Defaults pwfeedback` to defaults, to set stars when you type a password.

## Install grub
Let's install and configure bootloader - grub, and os-prober for multi system usage.
```shell
pacman -S grub os-prober efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```

### Install microcode
Install intel microcode for cpu microcode updates `pacman -S intel-ucode`. Then add `initrd /intel-ucode.img` in above
`initrd /initramfs-linux.img` in `/boot/loader/entries/arch.conf`.

### Configure networking
Now we need to install network manager `pacman -S (dhcpcd or wifimenu) networkmanager`. To enable them use command
`systemctl enable NetworkManager`

For now, installation and setup process is ending. Let's reboot system.
```shell
exit
umount -R /mnt
reboot
```

## System configure
Log in as user that you add before.

### Xorg
```shell
sudo pacman -S xorg xorg-xinit xorg-server
```

### Pulseaudio
```shell
sudo pacman -S pulseaudio pulseaudio-alsa
```

### Sddm and Plasma
```shell
sudo pacman -S sddm plasma
sudo systemctl enable sddm
```

### ?Ndidia driver
```shell
sudo pacman -S nvidia-lts nvidia-settings
```
Then use command `sudo nvidia-xconfig` to configure.

### Aur manager(yaourt)
```shell
sudo pacman -S --needed base-devel git wget yajl
cd /tmp
git clone https://aur.archlinux.org/package-query.git pq && cd ./pq
makepkg -si
cd ..
git clone https://aur.archlinux.org/yaourt.git yaourt && cd ./yaourt
makepkg -si
cd ..
rm -rf yaourt/ pq/
yaourt -S pamac-aur
```

## Shell configuration
Now you need to reboot you system and log back in non-root user.

I like to use zsh with plugin `oh-my-zsh`. Let's install them with plugins for completion and highlighting:
```shell
sudo pacman -Syu zsh
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
git clone https://github.com/zsh-users/zsh-autosuggestions.git $ZSH_CUSTOM/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
```

Add plugins `zsh-autosuggestions` and `zsh-syntax-highlighting` to plugin list in zsh resource file. Edit `~/.zshrc` :
```shell
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
```

?Optionally you can install some fonts for your terminal like `Powerlevel9k`:
```shell
sudo git clone https://github.com/bhilburn/powerlevel9k.git /usr/share/oh-my-zsh/themes/powerlevel9k 
pamac-aur -S nerd-fonts-hack
```

Now we need to make zsh - default shell. Use this commands:
```shell
sudo chsh -l
sudo chsh -s /usr/bin/zsh
```

I provide my `.zshrc` in [configuration file](config/zsh.cfg).

## Configure KDE
To configure plasma I've used guides from this [youtube channel](https://www.youtube.com/c/LinuxScoop).

# Good luck! :)
