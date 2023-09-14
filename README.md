# NixOS running on RK3588/RK3588s

> :warning: Work in progress, use at your own risk...

A minimal flake to run NixOS on RK3588/RK3588s based SBCs.

![](_img/nixos-on-orangepi5.webp)
![](_img/nixos-orangepi5plus.webp)
![](_img/nixos-rock5a.webp)

Default user: `rk`, default password: `rk3588`

## Boards

| Singal Board Computer | Boot from SD card  | Boot from SSD      |
| --------------------- | ------------------ | ------------------ |
| Orange Pi 5           | :heavy_check_mark: | :heavy_check_mark: |
| Orange Pi 5 Plus      | :heavy_check_mark: | :no_entry_sign:    |
| Rock 5A               | :heavy_check_mark: | :no_entry_sign:    |

## TODO

- [ ] build u-boot with nix
- [x] support boot from emmc/ssd
- [ ] verify all the hardware features available by RK3588/RK3588s
  - [x] ethernet (rj45)
  - wifi/bluetooth
  - audio
  - [x] gpio
  - [x] uart/ttl
  - gpu(mali-g610-firmware + [panfrok/mesa](https://gitlab.com/panfork/mesa))
  - npu
  - ...

## Flash into SD card

### 1. Flash U-Boot to SPI flash

You should get the uboot from the vendor and flash it to the SPI flash before doing anything NixOS

1. [Armbian on Orange Pi 5 / Orange Pi 5 Plus](https://www.armbian.com/orange-pi-5/) as an example:
   1. download the image and flash it to a sd card first
   2. boot the board with the sd card, and then run `sudo armbian-install` to flash the uboot to the SPI flash(maybe named as `MTD devices`)

For Rock 5A, we've bundled the uboot into the sdImage, so you don't need to flash it into the SPI flash manually.

### 2. Flash NixOS to SD card

Build an sdImage by `nix build`, and then flash it to a sd card using `dd`(please replace `/dev/sdX` with the correct device name of your sd card)):

> Instead of build from source, you can also download the prebuilt image from [Releases](https://github.com/ryan4yin/nixos-rk3588/releases).

```shell
# for orange pi 5 plus
nix build .#sdImage-opi5plus
zstdcat result/sd-image/orangepi5plus-sd-image-*.img.zst | sudo dd status=progress bs=4M of=/dev/sdX

# for orange pi 5
nix build .#sdImage-opi5
zstdcat result/sd-image/orangepi5-sd-image-*.img.zst | sudo dd status=progress bs=4M of=/dev/sdX
```


For Rock 5A, it requires a little more work to flash the image to the sd card:

```shell
nix build .#sdImage-rock5a
zstd -d result/sd-image/rock5a-sd-image-*.img.zst -o rock5a.img

# increase img's file size
dd if=/dev/zero bs=1M count=16 >> rock5a.img
sudo losetup --find --partscan rock5a.img

nix shell nixpkgs#parted
## rock 5a's u-boot require to use gpt partition table, and the root partition must be the first partition!
## so we need to remove all the partitions on the sd card first
## and then recreate the root partition with the same start sector as the original partition 2
START=$(sudo fdisk -l /dev/loop0 | grep /dev/loop0p2 | awk '{print $2}')
sudo parted /dev/loop0 rm 1
sudo parted /dev/loop0 rm 2
sudo parted /dev/loop0 mkpart primary ext4 ${START}s 100%

# check rootfs's status, it's broken.
sudo fsck /dev/loop0p1

# umount the image file
sudo losetup -d /dev/loop0


# flash the image to the sd card
sudo dd status=progress bs=4M if=rock5a.img of=/dev/sdX
```

1. insert the sd card to the board, and power on
2. resize the root partition to the full size of the sd card.
3. then having fun with NixOS

Once the system is booted, you can use `nixos-rebuild` to update the system.

## Flash into SSD/eMMC

To flash the image into the board's eMMC / SSD, you need to flash the image into the board and start into NixOS first.

Then, use the following command to flash the image into the board's SSD / eMMC:

```bash
# upload the sdImage to the NixOS system on the board
scp result/sd-image/orangepi5-sd-image-*.img.zst  rk@<ip-of-your-board>:~/

# login to the board via ssh or serial port
ssh rk@<ip-of-your-board>

# check all the block devices
# you should see nvme0n1(SSD)
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
mtdblock0    31:0    0    16M  0 disk
zram0       254:0    0     0B  0 disk
nvme0n1     259:0    0 238.5G  0 disk
......

# flash the image into the board's SSD
zstdcat orangepi5-sd-image-*.img.zst | sudo dd bs=4M status=progress of=/dev/nvme0n1
```

After the flash is complete, remove the SD card and reboot, you should see NixOS booting from SSD / eMMC.

## Debug via serial port(UART)

See [Debug.md](./Debug.md)

## Custom Deployment

You can use this flake as an input to build your own configuration.
Here is an example configuration that you can use as a starting point: [Demo - Deployment](./demo)

## How this flake works

A complete Linux system typically consists of five components: U-Boot, the kernel, device trees, firmwares, and the root file system (rootfs).

Among these, U-Boot, the kernel, device trees, and firmwares are hardware-related and require customization for different SBCs.
On the other hand, the majority of content in the rootfs is hardware-independent and can be shared across different SBCs.

Hence, the fundamental approach here is to use the hardware-specific components(U-Boot, kernel, and device trees, firmwares) provided by the vendor(orangepi/rockpi/...), and combine them with the NixOS rootfs to build a comprehensive system.

Regarding RK3588/RK3588s, a significant amount of work has been done by Armbian on their kernel, and device tree.
Therefore, by integrating these components from Armbian with the NixOS rootfs, we can create a complete NixOS system.

The primary steps involved are:

1. Build U-Boot using this Flake.
   - Since no customization is required for U-Boot, it's also possible to directly use the precompiled U-Boot from Armbian or the hardware vendor.
2. Build the NixOS rootfs using this flake, leveraging the kernel and device tree provided by Armbian.
   - To make all the hardware features available, we need to add its firmwares to the rootfs. Since there is no customization required for the firmwares too, we can directly use the precompiled firmwares from Armbian & Vendor.

Related Armbian projects:

- <https://github.com/armbian/build>
- <https://github.com/armbian/linux-rockchip>

## References

- [K900/nix](https://gitlab.com/K900/nix)
- [aciceri/rock5b-nixos](https://github.com/aciceri/rock5b-nixos)
- [nabam/nixos-rockchip](https://github.com/nabam/nixos-rockchip)
- [fb87/nixos-orangepi-5x](https://github.com/fb87/nixos-orangepi-5x)

And I also got a lot of help in the [NixOS on ARM Matrix group](https://matrix.to/#/#nixos-on-arm:nixos.org)!
