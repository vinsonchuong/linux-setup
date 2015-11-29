# Laptop Setup Notes

## BIOS Setup
* Disable any unused features and enable any useful features
  * Boot with UEFI only and without CSM support
  * Disable secure boot
* Ensure that USB drives are part of the boot order

## Writing Installation Media
* Download Arch Linux to a USB drive via
  ```sh
  bin/download_arch /dev/sdb
  ```

## Booting Installation Media
* Ensure internet connectivity is available via ethernet and DHCP. The installer will connect automatically.
* Boot from the USB installation media

## Disk Partitioning
* The following partition scheme will be produced, assuming an SSD with a capacity of at least 256GB:
  Label | Mount | Type | Size    
  ----- | ----- | ---- | --------
  boot  | /boot | vfat | 512MiB
  root  | /     | ext4 | 16GiB
  var   | /var  | ext4 | 128GiB
  home  | /home | ext4 | > 90GiB
* Wipe the installed SSD (assumed to be named `/dev/sda`) via
  ```sh
  sgdisk --zap-all /dev/sda
  ```
* Generate the partition table as follows (`_` denotes carriage return)
  ```sh
  gdisk /dev/sda
  n _ _ +512M ef00
  n _ _ +16G _
  n _ _ +128G _
  n _ _ _ _
  w
  ```
* Format the new partitions:
  ```sh
  mkfs.vfat /dev/sda1
  mkfs.ext4 /dev/sda2
  mkfs.ext4 /dev/sda3
  mkfs.ext4 /dev/sda4
  ```
* Label the partitions:
  ```sh
  echo 'mtools_skip_check=1' > ~/.mtoolsrc && mlabel -i /dev/sda1 boot
  e2label /dev/sda2 root
  e2label /dev/sda3 var
  e2label /dev/sda4 home
  ```
* Make `/boot` bootable via UEFI:
  ```sh
  efibootmgr -d /dev/sda -p 1 -c -L 'Arch Linux' -l '/vmlinuz-linux' -u 'root=/dev/sda2 rw initrd=/intel-ucode.img initrd=/initramfs-linux.img'
  INSTALLER=$(efibootmgr | awk '/BootCurrent/ {print $2}')
  OS=$(efibootmgr | awk '/Arch Linux/ {print $1}' | sed 's/^Boot\([^*]*\).*$/\1/')
  efibootmgr -O
  efibootmgr -o $INSTALLER,$OS
  ```
* Set reserved blocks to 0 on ext4 partitions:
  ```sh
  tune2fs -m 0.0 /dev/sda2
  tune2fs -m 0.0 /dev/sda3
  tune2fs -m 0.0 /dev/sda4
  ```
* Mount the partitions
  ```sh
  mount -o defaults,noatime,discard /dev/sda2 /mnt
  mkdir /mnt/boot && mount -o defaults,noatime,discard /dev/sda1 /mnt/boot
  mkdir /mnt/var && mount -o defaults,noatime,discard /dev/sda3 /mnt/var
  mkdir /mnt/home && mount -o defaults,noatime,discard /dev/sda4 /mnt/home
  ```
* Generate the `fstab` via
  ```sh
  genfstab -L -p /mnt | sed 's/rw[^\t]*/defaults,noatime,discard/' >> /mnt/etc/fstab
  ```

## Installing the Base System
* Install needed packages:
```sh
tee -a /etc/pacman.conf <<'EOF'

[infinality-bundle]
Server = http://bohoomil.com/repo/$arch

[infinality-bundle-fonts]
Server = http://bohoomil.com/repo/fonts
EOF
pacman-key -r 962DDE58 && pacman-key --lsign-key 962DDE58
pacman -Syy

pacstrap /mnt base base-devel \
  intel-ucode xf86-input-synaptics xf86-input-wacom xf86-video-intel libva-intel-driver libvdpau-va-gl alsa-utils \
  haveged tlp acpi_call acpid ethtool iw lsb-release smartmontools wpa_supplicant dnsmasq nftables \
  xorg-server xorg-server-utils xorg-apps \
  infinality-bundle ibfonts-meta-extended \
  lightdm lightdm-gtk3-greeter accountsservice \
  i3 gnome-icon-theme \
  gstreamer gst-libav gst-plugins-base gst-plugins-good gst-vaapi \
  atool zip unzip p7zip unrar \
  mediainfo \
  transmission-cli \
  jre7-openjdk cmake ctags \
  namcap pkgbuild-introspection burp \
  zsh zsh-completions git hub openssh rxvt-unicode tmux vim neovim python-neovim xclip firefox chromium flashplugin gimp pass di colordiff the_silver_searcher \
  zathura zathura-djvu zathura-pdf-mupdf zathura-ps \
  feh mpv beets python2-pylast python2-requests imagemagick \
  postgresql postgresql-old-upgrade nodejs npm \
  libvirt virt-manager qemu dmidecode ebtables
```
* chroot into the system and set basic settings:
  ```sh
  arch-chroot /mnt /bin/bash

  systemd-firstboot --locale=en_US.UTF-8 --timezone=America/Los_Angeles --hostname=laptop
  locale-gen
  hwclock --systohc --utc
  sed -i '/127\.0\.0\.1/s/$/ laptop/' /etc/hosts

  export LANG=en_US.UTF-8
  hostname laptop
  ```
* Install AUR packages:
  ```sh
  bash <(curl aur.sh) --asroot --noconfirm -si aura-bin 
  rm -rf aura-bin
  aura --noconfirm -Aya hostsblock gtk-theme-numix-solarized \
    dmenu-xft-height google-musicmanager xidel jq tmux-solarized-git flavoured \
    virt-viewer insync bats-git
  ```
* Copy configuration files:
  ```sh
  cd ~
  hub clone vinsonchuong/etcfiles
  cp -rf etcfiles/* /etc

  pacman-key -r 962DDE58 && pacman-key --lsign-key 962DDE58
  pacman -Syy
  ```
* Enable system services:
  ```sh
  systemctl enable systemd-networkd systemd-networkd-wait-online \
    systemd-resolved systemd-timesyncd nftables haveged tlp tlp-sleep \
    lightdm hostsblock.timer postgresql libvirtd
  ```
* Setup PostgreSQL:
  ```sh
  bin/setup_postgres
  ```
* Setup Networking
  ```sh
  ETHERNET_INTERFACE=$(find /sys/class/net -name 'en*' -printf '%f\n' | head -1)
  tee "/etc/systemd/network/$INTERFACE.network" <<EOF
  [Match]
  Name=$INTERFACE

  [Network]
  DHCP=yes
  EOF

  WIFI_INTERFACE=$(find /sys/class/net -name 'wl*' -printf '%f\n' | head -1)
  bin/setup_wifi "$WIFI_INTERFACE"
  cat <<EOF >> "/etc/wpa_supplicant/wpa_supplicant-$INTERFACE.conf"
  network={
    ssid="home"
    bssid=90:f6:52:e5:6e:d4
    psk=c2634ea070eda68dc2fc3d0e9b0078de35cf5470c7a23258fe74cc9bd1c1bd98
  }
  network={
    ssid="CalVisitor"
    key_mgmt=NONE
  }
  EOF
  wpa_passphrase 'Pivotal Guest' 'makeithappen' >> "/etc/wpa_supplicant/wpa_supplicant-$INTERFACE.conf"
  ```
* Setup libvirt
  ```sh
  groupadd libvirt

  virt-install --connect 'qemu:///system'
    -n 'windows' --ram 4096 --cpu 'host' --vcpus 2 --disk 'size=40' --graphics 'vnc' \
    --os-variant 'win2k12r2' --clock 'offset=localtime' \
    --cdrom '/home/vinsonchuong/downloads/en_windows_server_2012_r2_with_update_x64_dvd_4065220.iso'
  virsh -c 'qemu:///system' start 'windows' && virt-viewer -c 'qemu:///system' 'windows'
  ```
* Setup user account
  ```sh
  bin/mksudoer vinsonchuong
  ```
* Shutdown
  ```sh
  exit
  systemctl poweroff
  ```
* Remove the USB installation media and restart.

## User-Level Setup
```sh
ssh-keygen -t ecdsa -b 521 -C "$(whoami)@$(hostname)-$(date -I)" && ssh-add
gpg --gen-key
pass init 'vinsonchuong@gmail.com'
```

## Notes
* User-Level Setup
  * Firefox Configuration
    * https://wiki.archlinux.org/index.php/Firefox_tweaks
* `chmod 0600` files containing wireless passwords?
* Add i3 shortcuts for `dm-tool` commands

## Pages to Read
* Check on Wacom tablet drivers
* https://wiki.archlinux.org/index.php/Improve_pacman_performance
* https://wiki.archlinux.org/index.php/Mirrors#Sorting_mirrors
* https://wiki.archlinux.org/index.php/Improve_boot_performance#Readahead
* https://wiki.archlinux.org/index.php/Systemd-nspawn
* https://wiki.archlinux.org/index.php/Systemd-networkd
* Add wifi hotspots without root
* https://wiki.archlinux.org/index.php/Power_saving#Kernel_parameters
* https://wiki.archlinux.org/index.php/Systemd/cron_functionality
* http://www.thinkwiki.org/wiki/How_to_reduce_power_consumption
* https://wiki.ubuntu.com/Kernel/PowerManagement/PowerSavingTweaks
* https://wiki.archlinux.org/index.php/List_of_applications
* https://wiki.archlinux.org/index.php/Category:X_Server
* https://wiki.archlinux.org/index.php/Polkit#Authentication_agents
* https://wiki.archlinux.org/index.php/Desktop_notifications
* https://wiki.archlinux.org/index.php/Display_Power_Management_Signaling
* https://wiki.archlinux.org/index.php/General_recommendations
* http://vincent.jousse.org/tech/archlinux-retina-hidpi-macbookpro-xmonad/
* https://www.archlinux.org/news/reorganization-of-vim-packages/
* https://wiki.archlinux.org/index.php/LightDM#Enabling_autologin
* https://wiki.archlinux.org/index.php/File_systems#Create_a_filesystem - Mounting ISOs without root permissions
* https://wiki.archlinux.org/index.php/Category:System_administration
* https://wiki.archlinux.org/index.php/System_maintenance
* https://wiki.archlinux.org/index.php/Desktop_environment - Custom environments
* https://wiki.archlinux.org/index.php/Systemd/User#Xorg_as_a_systemd_user_service
* https://wiki.archlinux.org/index.php/Infinality-bundle%2Bfonts
* LightDM dm-tool for switching sessions: https://wiki.archlinux.org/index.php/LightDM
* https://wiki.archlinux.org/index.php/Fprint
* https://wiki.archlinux.org/index.php/Webcam_Setup
* http://support.lenovo.com/us/en/products/laptops-and-netbooks/thinkpad-x-series-laptops/thinkpad-x1-carbon-type-20a7-20a8/downloads/DS039783
* https://wiki.archlinux.org/index.php/Microcode#Enabling_Intel_Microcode_Updates
* https://wiki.archlinux.org/index.php/Nftables
* https://wiki.archlinux.org/index.php/Core_utilities
* https://wiki.archlinux.org/index.php/Man_page#Colored_man_pages
* https://wiki.archlinux.org/index.php/Zsh#Prompts
* https://wiki.archlinux.org/index.php/Tmux
* https://wiki.archlinux.org/index.php/Bash#Tips_and_tricks
* http://grml.org/zsh/
* https://wiki.archlinux.org/index.php/Maximizing_performance
* https://wiki.archlinux.org/index.php/CUPS
* https://launchpad.net/~thefanclub/+archive/ubuntu/grive-tools

http://jbt.github.io/markdown-editor/#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
