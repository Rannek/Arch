# Every Arch user has an install guide. This is mine.
This guide is tested on archlinux-2023.12.01-x86_64.iso

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
> Now this is the most important line! Replace UUID with ctrl + U
```
GRUB_CMDLINE_LINUX="cryptdevice=UUID:cryptroot root=/dev/mapper/vgname-root rootfstype=btrfs rootflags=subvol=@"
```
```
grub-install /dev/sda
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
