<p align="center">
  <img src="https://raw.githubusercontent.com/Rannek/arch-guide/main/logo.png" alt="Arch Guide Logo"/>
</p>


# Every Arch user has an install guide. This is mine.
This guide is tested on archlinux-2023.12.01-x86_64.iso
## Enhanced Security with LUKS

- **LUKS (Linux Unified Key Setup)**: Using LUKS for disk encryption is particularly beneficial if you're using a laptop. It encrypts your entire drive, meaning your data is secure even if your computer is lost or stolen.

- **Data Protection**: With LUKS, there's no need to overwrite your data before selling or disposing of your computer. The encryption ensures that your data remains inaccessible without the correct passphrase.


## Flexible File System with Btrfs

- **Snapshots with Btrfs**: Btrfs (B-tree File System) allows you to create snapshots of your system. This feature is very good for backing up and restoring your system state, making system updates and changes less risky.

- **Advanced Features**: Btrfs supports features like volume management, error detection, and self-repair capabilities, adding layers of robustness to your system.



## Why Choose This Guide?

- **Tailored for Arch Linux**: Arch Linux is known for its simplicity, efficiency, and customization capabilities. This guide leverages these strengths to provide you with a powerful and personalized computing experience.

- **Step-by-Step Instructions**: Whether you're an old Linux user or new to Arch, this guide walks you through every step of the installation and setup process.

### Let's start!

### Fdisk Configuration
```
fdisk /dev/sda
```

### Boot Partition
```
n
p
1
[Press Enter]
+512M
t
0c (W95 FAT 32 LBA)
a
```

### Root Partition
```
n
[Press Enter]
[Press Enter]
[Press Enter]
[Press Enter]
w
```

### Formatting Partitions
```
mkfs.fat -F32 /dev/sda1
cryptsetup luksFormat /dev/sda2
cryptsetup open /dev/sda2 cryptroot
```

### Creating LVM (Logical Volume Management)
> [!TIP]
> You can use `lvs` command to list all logical volumes.
> 
You need to start by initializing physical storage devices (like hard drives or partitions) as Physical Volumes. This is done using the pvcreate command.

Volume Group (VG) with vgcreate: Once you have one or more Physical Volumes, you can create a Volume Group. This is a pool of storage made from the Physical Volumes, and it's created using the vgcreate command.

Logical Volume (LV) with lvcreate: Finally, within the Volume Group, you can create Logical Volumes. These are the volumes that your operating system and applications will use, and they are created with the lvcreate command.


- **Physical Volume Create**:
  ```
  pvcreate /dev/mapper/cryptroot
  ```
- **Volume Group Create**:
  ```
  vgcreate vgname /dev/mapper/cryptroot
  ```
- **Logical Volume Create**:
  ```
  lvcreate -l 100%FREE vgname -n root
  ```

### Creating Btrfs Partition
```
mkfs.btrfs /dev/mapper/vgname-root
```

### Creating Btrfs Subvolumes @ and @home
```
mount /dev/mapper/vgname-root /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
umount /mnt
```

### Mount Btrfs Subvolumes and Boot
> [!TIP]
> Alternatively you can just use
> `mount -o subvol=@ /dev/mapper/vgname-root /mnt` and configure it later.
>
> Also `mount -o subvol=@home /dev/mapper/vgname-root /mnt/home`
```
mount -o subvol=@,rw,noatime,autodefrag,ssd,compress=zstd /dev/mapper/vgname-root /mnt
mkdir /mnt/home
mount -o subvol=@home,rw,noatime,autodefrag,ssd,compress=zstd /dev/mapper/vgname-root /mnt/home
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

### Install Base System
```
pacstrap /mnt base linux linux-firmware btrfs-progs base-devel nano
```

### Fstab
```
genfstab -U /mnt >> /mnt/etc/fstab
```

### Configuring System
```
arch-chroot /mnt
pacman -Syy lvm2
nano /etc/mkinitcpio.conf
HOOKS=(encrypt btrfs lvm2) <- Add these to the uncommented hooks line.
mkinitcpio -P
```

### Bootloader
> [!TIP]
> With this command and using ctrl + K and ctrl + U with nano you can easily copy the correct UUID. Go to the bottom of the file and copy it (ctrl+k) then paste where the UUID needed. If you are installing on SSH you can copy paste it more easily.
```
pacman -S grub
grub-install /dev/sda
blkid -s UUID -o value /dev/sda2 >> /etc/default/grub

nano /etc/default/grub
```
> now scroll down the document and copy the UUID by pressing ctrl + K
```
GRUB_ENABLE_CRYPTODISK=y (Uncomment this)
```
> [!TIP]
> GRUB_TIMEOUT=0 (Default is 5, but for faster boot you can change it to 0)

> [!CAUTION]
> Now this is the most important line!
```
GRUB_CMDLINE_LINUX="cryptdevice=UUID=[PASTE UUID HERE]:cryptroot root=/dev/mapper/vgname-root rootfstype=btrfs rootflags=subvol=@"
```

For example: GRUB_CMDLINE_LINUX="cryptdevice=UUID=0d02ca7d-b4bd-47a8-8df8-70c972be025f:cryptroot root=/dev/mapper/vgname-root rootfstype=btrfs rootflags=subvol=@"
```
grub-mkconfig -o /boot/grub/grub.cfg
passwd
```

### Network
```
Pacman -S networkmanager
systemctl enable NetworkManager
```

### Users + Sudo
```
useradd -m test
passwd test
pacman -S sudo
EDITOR=nano visudo
test ALL=(ALL) ALL <-- Add this to the end of the file.

# For testing
su - test
sudo ls /root
```

### KDE + SDDM
> [!TIP]
> If you want a minimal installation, you can just install plasma-desktop.
![KDE Desktop](https://raw.githubusercontent.com/Rannek/arch-guide/main/KDE.png)
```

pacman -S xorg
pacman -S plasma kde-applications
pacman -S sddm
systemctl enable sddm
```

### Hungarian Keyboard
```
nano /etc/vconsole.conf
KEYMAP=hu
```

### Timeshift
```
btrfs subvolume list /
pacman -S timeshift
systemctl enable --now cronie
systemctl start cronie
```
