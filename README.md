# Laptop Setup Notes

## BIOS Setup
* Disable any unused features and enable any useful features
  * Boot with UEFI only and without CSM support
  * Disable secure boot
* Enable Thunderbolt BIOS Assist Mode
* Ensure that USB drives are part of the boot order

## Writing Installation Media
* Download Arch Linux to a USB drive via
  ```sh
  bin/download_arch /dev/sdb
  ```

## Booting Installation Media
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
  root  | /     | ext4 | 16GiB
  var   | /var  | ext4 | 512GiB
  home  | /home | ext4 | ~ 400GiB
* Determine the device name for the main SSD by running
  ```sh
  lsblk
  ```
* Wipe the installed SSD (assumed to be named `/dev/nvme0n1`) via
  ```sh
  sgdisk --zap-all /dev/nvme0n1
  ```
* Generate the partition table as follows (`_` denotes carriage return)
  ```sh
  gdisk /dev/nvme0n1
  n _ _ +512M ef00
  n _ _ +16G _
  n _ _ +512G _
  n _ _ _ _
  w
  ```
* Format the new partitions:
  ```sh
  mkfs.vfat /dev/nvme0n1p1
  mkfs.ext4 /dev/nvme0n1p2
  mkfs.ext4 /dev/nvme0n1p3
  mkfs.ext4 /dev/nvme0n1p4
  ```
* Label the partitions:
  ```sh
  echo 'mtools_skip_check=1' > ~/.mtoolsrc && mlabel -i /dev/nvme0n1p1 ::boot
  e2label /dev/nvme0n1p2 root
  e2label /dev/nvme0n1p3 var
  e2label /dev/nvme0n1p4 home
  ```
* Make `/boot` bootable via UEFI:
  ```sh
  efibootmgr -d /dev/nvme0n1 -p 1 -c -L 'Arch Linux' -l '/vmlinuz-linux' -u 'root=/dev/nvme0n1p2 rw initrd=/intel-ucode.img initrd=/initramfs-linux.img'
  ```
* Set reserved blocks to 0 on ext4 partitions:
  ```sh
  tune2fs -m 0.0 /dev/nvme0n1p2
  tune2fs -m 0.0 /dev/nvme0n1p3
  tune2fs -m 0.0 /dev/nvme0n1p4
  ```
* Mount the partitions
  ```sh
  mount -o defaults,noatime,discard /dev/nvme0n1p2 /mnt
  mkdir /mnt/boot && mount -o defaults,noatime,discard /dev/nvme0n1p1 /mnt/boot
  mkdir /mnt/var && mount -o defaults,noatime,discard /dev/nvme0n1p3 /mnt/var
  mkdir /mnt/home && mount -o defaults,noatime,discard /dev/nvme0n1p4 /mnt/home
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
  pacman-contrib \
  efibootmgr intel-ucode fwupd \
  dosfstools haveged openssh \
  tlp acpi_call iw ethtool lsb-release smartmontools x86_energy_perf_policy \
  iw wpa_supplicant dnsmasq nftables \
  xf86-input-libinput xf86-input-wacom \
  xf86-video-intel libva-intel-driver libvdpau-va-gl \
  xorg-server xorg-apps \
  alsa-utils pulseaudio pulseaudio-alsa \
  gstreamer gst-libav gst-plugins-base gst-plugins-good gstreamer-vaapi \
  lightdm lightdm-gtk-greeter accountsservice \
  di colordiff \
  the_silver_searcher \
  atool bzip2 cpio gzip lha xz lzop p7zip tar unace unrar zip unzip \
  zsh zsh-completions \
  neovim python-neovim xclip \
  zathura zathura-djvu zathura-pdf-mupdf zathura-ps zathura-cb \
  feh \
  mpv \
  beets python-beautifulsoup4 python-pylast python-requests imagemagick python-xdg \
  mediainfo \
  gphoto2 \
  darktable \
  i3 gnome-icon-theme ttf-dejavu \
  rxvt-unicode \
  tmux \
  firefox firefox-developer-edition \
  chromium \
  flashplugin \
  gimp \
  pass \
  virt-manager libvirt ebtables dnsmasq bridge-utils gnu-netcat qemu dmidecode \
  docker \
  cmake ctags gconf \
  python python-pip \
  ruby ruby-bundler ruby-rdoc \
  nodejs npm yarn \
  jdk10-openjdk \
  git hub \
  certbot \
  postgresql postgresql-old-upgrade \
  jq
```
* Generate the `fstab` via
  ```sh
  genfstab -L -p /mnt | sed 's/rw[^\t]*/defaults,noatime,discard/' >> /mnt/etc/fstab
  ```
* chroot into the system and set basic settings:
  ```sh
  arch-chroot /mnt /bin/bash

  systemd-firstboot --locale=en_US.UTF-8 --timezone=America/Los_Angeles --hostname=laptop
  sed -i '/^#en_US\.UTF-8 UTF-8/s/#//' /etc/locale.gen
  locale-gen
  hwclock --systohc --utc
  echo 'Server = https://mirrors.kernel.org/archlinux/$repo/os/$arch' > /etc/pacman.d/mirrorlist

  systemctl enable systemd-timesyncd haveged tlp tlp-sleep

  git clone https://github.com/vinsonchuong/laptop /root/laptop
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
    fonts-meta-extended-lt \
    google-musicmanager qtwebkit-bin \
    insync \
    hostsblock \
    gtk-theme-numix-solarized tmux-solarized16 flavoured \
    dmenu-height \
    gitaur bats-git cloudfoundry-cli heroku-toolbelt \
    stepmania-git antimicro
  sudo aura -Oj
  sudo paccache -r
  sudo paccache -ruk0
  exit
  ```
* Setup login manager:
  ```sh
  sed 's/#autologin-user=/autologin-user=vinsonchuong/' -i /etc/lightdm/lightdm.conf
  groupadd -r autologin
  gpasswd -a vinsonchuong autologin
  systemctl enable lightdm
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
* Setup PostgreSQL:
  ```sh
  systemctl enable postgresql

	su postgres -c 'initdb -D /var/lib/postgres/data'

  mkdir /run/postgresql
  chown postgres:postgres /run/postgresql
  su postgres -c 'pg_ctl -s -D /var/lib/postgres/data start -w -t 120'

	for user in $(groupmems -g users -l)
	do
		su postgres -c "createuser -dw '${user}'"
	done

  su postgres -c 'pg_ctl -s -D /var/lib/postgres/data stop -m fast'
  rm -rf /run/postgresql
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
