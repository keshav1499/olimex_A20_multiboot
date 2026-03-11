# A20-OLinuXino-LIME2 Network Boot Bring-up Guide

This document describes how to configure **U-Boot, TFTP, and NFS
rootfs** so the\
A20-OLinuXino-LIME2 (Olimex Allwinner A20 ARM development board) boots
Linux over the network.

The final setup allows:

    U-Boot
       ↓
    TFTP kernel + DTB
       ↓
    bootz
       ↓
    NFS root filesystem

This is ideal for **kernel development and board bring-up** because you
never need to rewrite the SD card again.

------------------------------------------------------------------------

# 1. Host machine setup (Arch / CachyOS)

Install required packages:

    sudo pacman -S \
        git \
        arm-none-eabi-gcc \
        base-devel \
        tftp-hpa \
        nfs-utils \
        rsync

------------------------------------------------------------------------

# 2. Directory layout on host

Create the server directories:

    sudo mkdir -p /srv/{tftp,nfs}
    sudo mkdir -p /srv/nfs/lime2-rootfs

Recommended layout:

    /srv
     ├── tftp
     │   ├── zImage
     │   └── sun7i-a20-olinuxino-lime2.dtb
     │
     └── nfs
         └── lime2-rootfs

------------------------------------------------------------------------

# 3. Setup TFTP server

Start `in.tftpd`:

    sudo systemctl enable --now tftpd

Or run manually:

    sudo in.tftpd --listen --secure /srv/tftp

Verify server:

    ps aux | grep tftp

Expected:

    in.tftpd --listen --secure /srv/tftp

Ensure files are readable:

    sudo chmod 755 /srv/tftp
    sudo chmod 644 /srv/tftp/*

------------------------------------------------------------------------

# 4. Setup NFS rootfs

Export the rootfs:

    sudo nano /etc/exports

Add:

    /srv/nfs/lime2-rootfs 192.168.1.0/24(rw,sync,no_root_squash,no_subtree_check)

Reload exports:

    sudo exportfs -ra
    sudo systemctl enable --now nfs-server

Verify:

    showmount -e

------------------------------------------------------------------------

# 5. Copy root filesystem

Mount the rootfs from the SD card image:

    sudo mount /dev/mmcblk0p2 /mnt/rootfs

Copy preserving permissions:

    sudo rsync -aAXHv /mnt/rootfs/ /srv/nfs/lime2-rootfs/

This is critical because login will fail if permissions are not
preserved.

------------------------------------------------------------------------

# 6. Build U-Boot

Clone U-Boot:

    git clone https://source.denx.de/u-boot/u-boot.git
    cd u-boot

Use the following configuration.

------------------------------------------------------------------------

# Working U-Boot configuration

This configuration **fixes Ethernet bring-up** on the board.

    CONFIG_ARM=y
    CONFIG_ARCH_SUNXI=y
    CONFIG_DEFAULT_DEVICE_TREE="sun7i-a20-olinuxino-lime2"
    CONFIG_DRAM_CLK=384
    CONFIG_SPL=y
    CONFIG_MACH_SUN7I=y
    CONFIG_I2C1_ENABLE=y
    CONFIG_SPL_SPI_SUNXI=y
    CONFIG_AHCI=y
    # CONFIG_SYS_MALLOC_CLEAR_ON_INIT is not set
    CONFIG_SPL_I2C=y
    CONFIG_CMD_DFU=y
    CONFIG_CMD_USB_MASS_STORAGE=y
    CONFIG_SCSI_AHCI=y
    CONFIG_SYS_64BIT_LBA=y
    CONFIG_DFU_RAM=y
    CONFIG_FASTBOOT_CMD_OEM_FORMAT=y
    CONFIG_SYS_I2C_MVTWSI=y
    CONFIG_SYS_I2C_SLAVE=0x7f
    CONFIG_SYS_I2C_SPEED=400000

    CONFIG_ETH_DESIGNWARE=y
    CONFIG_RGMII=y
    CONFIG_MII=y
    CONFIG_SUN7I_GMAC=y

    CONFIG_PINCTRL=y
    CONFIG_PINCTRL_SUNXI=y
    CONFIG_DM_GPIO=y
    CONFIG_DM_ETH=y

    CONFIG_PHY_MICREL=y
    CONFIG_PHY_MICREL_KSZ90X1=y
    CONFIG_PHY_MICREL_KSZ8XXX=y

    CONFIG_GMAC_TX_DELAY=4

    CONFIG_AXP_ALDO3_VOLT=2800
    CONFIG_AXP_ALDO3_VOLT_SLOPE_08=y
    CONFIG_AXP_ALDO3_INRUSH_QUIRK=y
    CONFIG_AXP_ALDO4_VOLT=2800

    CONFIG_SCSI=y
    CONFIG_USB_EHCI_HCD=y
    CONFIG_USB_OHCI_HCD=y
    CONFIG_USB_MUSB_GADGET=y

------------------------------------------------------------------------

# Important Ethernet configuration

The Lime2 uses:

    Micrel KSZ9031 Gigabit PHY

Two settings are critical:

    CONFIG_PHY_MICREL=y
    CONFIG_GMAC_TX_DELAY=4

Without these, the Ethernet PHY link comes up but **ARP fails**.

Symptoms when broken:

    Speed: 100 full duplex
    ARP Retry count exceeded

With the correct config:

    host 192.168.1.50 is alive

------------------------------------------------------------------------

# 7. Build U-Boot

    make CROSS_COMPILE=arm-none-eabi- distclean
    make CROSS_COMPILE=arm-none-eabi- A20-OLinuXino-Lime2_defconfig
    make CROSS_COMPILE=arm-none-eabi-

Flash to SD card:

    sudo dd if=u-boot-sunxi-with-spl.bin of=/dev/mmcblk0 bs=1024 seek=8

------------------------------------------------------------------------

# 8. Network boot from U-Boot

Interrupt autoboot and configure network:

    setenv ipaddr 192.168.1.121
    setenv serverip 192.168.1.50
    setenv netmask 255.255.255.0

Test connectivity:

    ping 192.168.1.50

Expected:

    host 192.168.1.50 is alive

------------------------------------------------------------------------

# 9. Load kernel and DTB via TFTP

    tftp 0x42000000 zImage
    tftp 0x43000000 sun7i-a20-olinuxino-lime2.dtb

Typical transfer speed:

    4–5 MiB/s

------------------------------------------------------------------------

# 10. Boot kernel using NFS root

Set kernel arguments:

    setenv bootargs \
    console=ttyS0,115200 \
    root=/dev/nfs \
    nfsroot=192.168.1.50:/srv/nfs/lime2-rootfs,tcp \
    rw \
    ip=dhcp

Boot kernel:

    bootz 0x42000000 - 0x43000000

------------------------------------------------------------------------

# 11. Successful boot log

Example:

    Booting Linux on physical CPU 0x0
    Machine model: Olimex A20-OLinuXino-LIME2
    sun7i-dwmac ... configuring for phy/rgmii-id link mode

------------------------------------------------------------------------

# 12. Login

Default credentials (Arch ARM):

    user: alarm
    pass: alarm

or

    root

------------------------------------------------------------------------

# Final development workflow

After this setup:

    edit kernel
    make zImage
    reboot board

No SD card flashing required.

------------------------------------------------------------------------

# Debugging checklist

If networking fails in U-Boot:

  Symptom                             Cause
  ----------------------------------- ---------------------------
  ARP Retry exceeded                  wrong PHY delay
  TFTP request seen but no response   server permissions
  login incorrect                     rootfs permissions broken
  link LED blinking constantly        GMAC pinmux incorrect

------------------------------------------------------------------------

# Result

With the above configuration the board supports:

    U-Boot → TFTP kernel → NFS rootfs

which is the recommended workflow for **kernel bring-up and driver
development**.
