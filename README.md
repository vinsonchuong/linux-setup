# Linux Setup Notes

## BIOS Settings
* Boot with UEFI only and without CSM support
* Disable secure boot
* Ensure that USB drives are part of the boot order

## Writing Installation Media
* Download Arch Linux to a USB drive via
  ```sh
  bin/download_arch /dev/sdb
  ```

## Boot Installation Media
* Boot from the USB installation media

## Networking
* If Ethernet is connected, the installer will connect automatically.
* If only WiFi is available, connect as follows:
  ```sh
  WIFI_INTERFACE=$(find /sys/class/net -name 'wl*' -printf '%f\n')
  iw link set $WIFI_INTERFACE up
  wpa_supplicant -B -i $WIFI_INTERFACE -c <(wpa_passphrase SSID PASSPHRASE)
  ```

## Updating System Clock
* Run
  ```sh
  timedatectl set-ntp true
  ```

## Disk Partitioning
* The following partition scheme will be produced, assuming an SSD with a capacity of at least 1TB:
  Label | Mount | Type | Size
  ----- | ----- | ---- | --------
  boot  | /boot | vfat | 512MiB
  root  | /     | ext4 | ~928GiB
* Determine the device name for the main SSD by running
  ```sh
  fdisk -l
  ```
* Wipe the installed SSD (assumed to be named `/dev/nvme0n1`) via
  ```sh
  sgdisk --zap-all /dev/nvme0n1
  ```
* Generate the partition table as follows (`_` denotes carriage return)
  ```sh
  gdisk /dev/nvme0n1
  n _ _ +512M ef00
  n _ _ _ _
  w
  ```
* Format the new partitions:
  ```sh
  mkfs.vfat /dev/nvme0n1p1
  mkfs.ext4 /dev/nvme0n1p2
  ```
* Label the partitions:
  ```sh
  echo 'mtools_skip_check=1' > ~/.mtoolsrc && mlabel -i /dev/nvme0n1p1 ::boot
  e2label /dev/nvme0n1p2 root
  ```
* Make `/boot` bootable via UEFI:
  ```sh
  efibootmgr -d /dev/nvme0n1 -p 1 -c -L 'Arch Linux' -l '/vmlinuz-linux' -u 'root=/dev/nvme0n1p2 rw initrd=/amd-ucode.img initrd=/initramfs-linux.img'
  ```
* Set reserved blocks to 0 on ext4 partitions:
  ```sh
  tune2fs -m 0.0 /dev/nvme0n1p2
  ```
* Mount the partitions
  ```sh
  mount -o defaults,noatime,discard /dev/nvme0n1p2 /mnt
  mkdir /mnt/boot && mount -o defaults,noatime,discard /dev/nvme0n1p1 /mnt/boot
  ```
* Mark the EFI partition
  ```sh
  mkdir /mnt/boot/EFI
  ```

## Installing the Base System
* Install needed packages:
```sh
echo 'Server = https://mirrors.kernel.org/archlinux/$repo/os/$arch' > /etc/pacman.d/mirrorlist
pacstrap /mnt base base-devel \
  efibootmgr amd-ucode fwupd \
  mesa vulkan-radeon libva-mesa-driver mesa-vdpau \
  greetd sway xorg-xwayland waybar otf-font-awesome wofi \

  lm_sensors \
  pipewire pipewire-alsa pipewire-pulse alsa-utils pavucontrol \

  alacritty tmux zsh zsh-completions \

  noto-fonts noto-fonts-extra noto-fonts-cjk noto-fonts-emoji \

  man-db man-pages \
  firefox-developer-edition \
  atool bzip2 cpio gzip lhasa lzop p7zip tar unace unrar unzip xz zip \
  git hub github-cli \
  pass wl-clipboard \

  pacman-contrib \
  dosfstools openssh \
  iw ethtool lsb-release \
  wpa_supplicant nftables wireless_tools \
  di colordiff \
  the_silver_searcher \
  atool bzip2 cpio gzip lha xz lzop p7zip tar unace unrar zip unzip \
  neovim python-neovim xclip \
  zathura zathura-djvu zathura-pdf-mupdf zathura-ps zathura-cb \
  feh \
  mpv \
  mediainfo \
  gphoto2 \
  darktable \
  i3 gnome-icon-theme ttf-dejavu \
  chromium \
  gimp \
  pass \
  docker \
  nodejs npm yarn \
  jdk10-openjdk \
  git hub \
  certbot \
  jq
```
* Generate the `fstab` via
  ```sh
  genfstab -L -p /mnt | sed 's/rw[^\t]*/defaults,noatime,discard/' >> /mnt/etc/fstab
  ```
* chroot into the system and set basic settings:
  ```sh
  arch-chroot /mnt /bin/bash

  sed -i '/^#en_US\.UTF-8 UTF-8/s/#//' /etc/locale.gen
  locale-gen
  systemd-firstboot --locale=en_US.UTF-8 --timezone=America/Los_Angeles --hostname=laptop
  hwclock --systohc --utc
  echo 'Server = https://mirrors.kernel.org/archlinux/$repo/os/$arch' > /etc/pacman.d/mirrorlist

  systemctl enable systemd-timesyncd

  git clone https://github.com/vinsonchuong/linux-setup /root/linux-setup
  ```
* Setup user account
  ```sh
  /root/laptop/bin/mksudoer vinsonchuong
  ```
* Install AUR packages:
  ```sh
  su - vinsonchuong
  bash <(curl aur.sh) -si --noconfirm aura-bin
  rm -rf aura-bin
  sudo aura --noconfirm -Aya \
    wev \
    gtk3-theme-numix-solarized papirus-icon-theme \
    tmux-solarized16 \



    flavoured \
    fonts-meta-extended-lt \
    google-musicmanager qtwebkit-bin \
    insync \
    hostsblock \
    dmenu-height \
    gitaur bats-git cloudfoundry-cli heroku-toolbelt \
    python2-neovim-git \
    stepmania-git antimicro
  sudo aura -Oj
  sudo paccache -r
  sudo paccache -ruk0
  exit
  ```
* Setup login manager:
  ```sh
  cat <<EOF >> /etc/greetd/config.toml
  [initial_session]
  command = "sway"
  user = "vinsonchuong"
  EOF
  systemctl enable greetd
  ```
* Configure Fonts:
  ```sh
  ln -s /etc/fonts/conf.avail/10-sub-pixel-rgb.conf /etc/fonts/conf.d
  ln -s /etc/fonts/conf.avail/11-lcdfilter-default.conf /etc/fonts/conf.d
  ln -s /etc/fonts/conf.avail/30-infinality-aliases.conf /etc/fonts/conf.d
  ```
* Set monitor DPI:
  ```sh
  cat <<EOF >> /etc/X11/xorg.conf.d/10-monitor.conf
  Section "Monitor"
    Identifier "eDP1"
    DisplaySize 310 174
  EndSection

  Section "Monitor"
    Identifier "DP2"
    DisplaySize 697 392
  EndSection
  EOF
  ```
* Set keyboard and trackpad settings:
  ```sh
  cat <<EOF >> /etc/X11/xorg.conf.d/10-keyboard.conf
Section "InputClass"
	Identifier "AT Translated Set 2 keyboard"
	MatchProduct "AT Translated Set 2 keyboard"
	Option "XkbOptions" "caps:super,altwin:prtsc_rwin,ctrl:swap_lalt_lctl"
EndSection

Section "InputClass"
	Identifier "Topre Corporation HHKB Professional"
	MatchProduct "Topre Corporation HHKB Professional"
	Option "XkbOptions" "ctrl:swap_lwin_lctl"
EndSection
  EOF

  cat <<EOF >> /etc/X11/xorg.conf.d/10-trackpad.conf
Section "InputClass"
	Identifier "Trackpad"
	MatchIsTouchpad "on"
	Driver "libinput"
	Option "Tapping" "on"
EndSection
  EOF
  ```
* Setup Networking
  ```sh
  WIFI_INTERFACE=$(find /sys/class/net -name 'wl*' -printf '%f\n' | head -1)

  systemctl enable systemd-networkd systemd-networkd-wait-online nftables "wpa_supplicant@$WIFI_INTERFACE" dnsmasq

  cat <<EOF >> /etc/systemd/network/wifi.network
  [Match]
  Name=wl*

  [Network]
  DHCP=ipv4
  IPForward=1
  EOF

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

  cat <<EOF >> /etc/dnsmasq.conf
server=1.1.1.1
server=1.0.0.1
  EOF
  ```
* Configure Host Blacklist:
  ```sh
  cat <<EOF >> /var/lib/hostsblock/hostsblock.conf
blocklists=(
  'http://support.it-mate.co.uk/downloads/HOSTS.txt'
  'http://winhelp2002.mvps.org/hosts.zip'
  'http://pgl.yoyo.org/as/serverlist.php?hostformat=hosts&mimetype=plaintext'
  'http://hosts-file.net/download/hosts.zip'
  'http://www.malwaredomainlist.com/hostslist/hosts.txt'
  'http://hosts-file.net/ad_servers.txt'
  'http://hosts-file.net/hphosts-partial.asp'
  'http://hostsfile.org/Downloads/BadHosts.unx.zip'
  'http://hostsfile.mine.nu/Hosts.zip'
  'http://sysctl.org/cameleon/hosts'
)
  EOF

  mkdir /etc/hosts.d
  ln -s /var/lib/hostsblock/hosts.block /etc/hosts.d
  echo 'hostsdir=/etc/hosts.d' >> /etc/dnsmasq.conf

  systemctl enable hostsblock.timer
  ```
* Setup libvirt
  ```sh
  systemctl enable libvirtd

  virt-install --connect 'qemu:///system'
    -n 'windows' --ram 4096 --cpu 'host' --vcpus 2 --disk 'size=40' --graphics 'vnc' \
    --os-variant 'win2k12r2' --clock 'offset=localtime' \
    --cdrom '/home/vinsonchuong/downloads/en_windows_server_2012_r2_with_update_x64_dvd_4065220.iso'
  virsh -c 'qemu:///system' start 'windows' && virt-viewer -c 'qemu:///system' 'windows'
  ```
* Setup Docker
  ```sh
  cat <<EOF >> /etc/nftables.conf
  table inet filter {
    chain input {
      type filter hook input priority 0;
      ct state {established, related} accept
      ct state invalid drop
      iifname lo accept
      ip protocol icmp accept
      ip6 nexthdr icmpv6 accept
      tcp dport ssh accept
      reject with icmpx type port-unreachable
    }

    chain forward {
      type filter hook forward priority 0;
    }

    chain output {
      type filter hook output priority 0;
    }
  }

  table ip nat {
    chain prerouting {
      type nat hook prerouting priority 0;
    }

    chain postrouting {
      type nat hook postrouting priority 0;
      oifname "wlp2s0" masquerade
    }
  }
  EOF

  mkdir /etc/systemd/system/docker.service.d
  cat <<EOF > /etc/systemd/system/docker.service.d/noiptables.conf
  [Service]
  ExecStart=
  ExecStart=/usr/bin/dockerd -H fd:// --iptables=false
  EOF

  systemctl daemon-reload
  systemctl enable docker
  usermod -a -G docker vinsonchuong
  ```
* Shutdown
  ```sh
  rm -rf /root/laptop/{*,.*}
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
