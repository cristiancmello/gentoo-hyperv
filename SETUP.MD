
# SETUP HYPER-V VM

```sh
# Set VM Name, Switch Name, and Installation Media Path.
$VMName = 'hyperv-gentoo'
$Switch = 'Default Switch'
$InstallMedia = 'D:\Linux\install-amd64-minimal-20221113T170154Z.iso'

# Create New Virtual Machine
New-VM -Name $VMName -MemoryStartupBytes 8GB -Generation 2 -NewVHDPath "D:\VMs\$VMName\$VMName.vhdx" -NewVHDSizeBytes 40GB -Path "D:\VMs\$VMName" -SwitchName $Switch

Set-VMProcessor $VMName -Count 8 -ExposeVirtualizationExtensions $false

# Add DVD Drive to Virtual Machine
Add-VMScsiController -VMName $VMName
Add-VMDvdDrive -VMName $VMName -ControllerNumber 1 -ControllerLocation 0 -Path $InstallMedia

# Mount Installation Media
$DVDDrive = Get-VMDvdDrive -VMName $VMName

# Configure Virtual Machine to Boot from DVD
Set-VMFirmware -VMName $VMName -FirstBootDevice $DVDDrive -EnableSecureBoot Off

Set-VMNetworkAdapter -VMName $VMName -StaticMacAddress '00155DEC6314'
Start-VM -VMName $VMName
```

## Get Network Adapter Info

```sh
Get-VMNetworkAdapter -VMName $VMName
```

## Get IP Address

```sh
# ALTERNATIVA A
Get-VM | ?{$_.State -eq "Running"} | select -ExpandProperty networkadapters | select vmname, macaddress, switchname, ipaddresses | ft -wrap -autosize

# ALTERNATIVA B
Get-VM -Name $VMName | Select -ExpandProperty Networkadapters | Select vmname, ipaddress
```

## Attach Serial Console

```sh
Get-VMComPort -VMName $VMName
Set-VMComPort -VMName $VMName -Number 2 -Path "\\.\pipe\$VMName-com1"
Get-VMComPort -VMName $VMName
```

## Setup Kernel and OS

```sh
# ---[HYP A='ROOT-HYPERV-GRAPHCONSOLEFROMWIN10']
5                                       # Keyboard Layout ABNT-2
passwd                                  # escolha 'root'
rc-service sshd start
ip addr show eth0
ip route get 1.2.3.4 | awk '{print $7}' # alternativa futura para automatizar
# ---------------------------
# ---[HYP A.B='gpt-efi-espon-swap-fs-format-gentoo-mount']
parted -js /dev/sda print        # exibir particoes em formato JSON
parted -s  /dev/sda print        # exibir particoes em formato TABELA SEM INTERACAO COM USUARIO (-s)
parted -s  /dev/sda mktable GPT  # Criando tabela de particao GPT
parted -s  /dev/sda mkpart primary fat32 1M 250M
parted -s  /dev/sda mkpart primary linux-swap 250M 4G
parted -s  /dev/sda mkpart primary ext4 4G 100%
parted -s  /dev/sda set 1 esp on
mkfs.vfat -F 32 /dev/sda1
mkfs.ext4 /dev/sda3
mkswap /dev/sda2
swapon /dev/sda2
mount /dev/sda3 /mnt/gentoo
# ---------------------------
# ---[HYP A.B.C='']
ntpd -qg
cd /mnt/gentoo
wget http://gentoo.c3sl.ufpr.br/releases/amd64/autobuilds/20221113T170154Z/stage3-amd64-openrc-20221113T170154Z.tar.xz
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
blkid
nano -w /mnt/gentoo/etc/fstab
/dev/sda1   /boot        vfat    defaults,noatime     0 2
/dev/sda2   none         swap    sw                   0 0
/dev/sda3   /            ext4    noatime              0 1
/dev/cdrom  /mnt/cdrom   auto    noauto,user          0 0

nano -w /mnt/gentoo/etc/portage/make.conf
COMMON_FLAGS="-march=tigerlake -O2 -pipe"
MAKEOPTS="-j8 --load-average=9"
# ---------------------------
# ---[HYP A.B.C.D='']
mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
mkdir --parents /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
mount /dev/sda1 /boot
emerge-webrsync
eselect profile list
eselect profile set 1
emerge --verbose --update --deep --newuse @world
emerge app-portage/cpuid2cpuflags
nano -w /etc/portage/make.conf
ACCEPT_LICENSE="-* @FREE @BINARY-REDISTRIBUTABLE"

echo "Brazil/East" > /etc/timezone
emerge --config sys-libs/timezone-data
nano -w /etc/locale.gen
en_US ISO-8859-1
en_US.UTF-8 UTF-8

locale-gen
eselect locale list
eselect locale set 6 # [6]   en_US.utf8
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
emerge sys-kernel/gentoo-sources
eselect kernel list  # [1]   linux-5.15.75-gentoo
eselect kernel set 1
ls -l /usr/src/linux
emerge sys-apps/pciutils
cd /usr/src/linux
# +++++[SAVING...PRE-MAKE-KERNELCONFIG]
# +++++++++++++++++++++++++++++++++++
# +++++[RESTORE FROM PRE-MAKE-KERNELCONFIG]
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
cd /usr/src/linux
# +++++++++++++++++++++++++++++++++++
#make mrproper                               # Clean Up
#make tinyconfig                             # Tiny Config
make menuconfig
function should_built_tiny_kernel() {
EXPECTED_ENABLED_CONFIGS=16
KERNEL_ENABLED_CONFIGS_SIZE=$(cat .config | grep -Eio "\
CONFIG_64BIT=y|\
CONFIG_MODULES=y|\
...
" | wc -l)
[ "$KERNEL_ENABLED_CONFIGS_SIZE" -eq "$EXPECTED_ENABLED_CONFIGS" ] && printf "KERNEL ENABLED CONFIGS: "; echo "PASSED" || echo "FAILED"
EXPECTED_DISABLED_CONFIGS=3
KERNEL_DISABLED_CONFIGS_SIZE=$(cat .config | grep -Eio "\
CONFIG_EXPERT=n|CONFIG_EXPERT is not set|\
...
" | wc -l)
[ "$KERNEL_DISABLED_CONFIGS_SIZE" -eq "$EXPECTED_DISABLED_CONFIGS" ] && printf "KERNEL DISABLED CONFIGS: "; echo "PASSED" || echo "FAILED"
}
should_built_tiny_kernel
echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
echo 'VIDEO_CARDS="fbdev"' >> /etc/portage/make.conf
emerge --update --newuse --deep x11-base/xorg-drivers
make -j8 --load-average=9 && make modules_install
cd /usr/src/linux/tools/hv/
make install
cd ../..
make install
emerge sys-kernel/dracut
dracut --kver=5.15.75-gentoo
ls /boot/initramfs*
nano -w /etc/conf.d/hostname
hostname="hyperv-gentoo"

nano -w /etc/security/passwdqc.conf
enforce=none

passwd # user: root; pass: root
emerge app-admin/sysklogd sys-apps/mlocate
rc-update add sysklogd default
rc-update add sshd default
nano -w /etc/inittab
# SERIAL CONSOLES
s0:12345:respawn:/sbin/agetty 9600 ttyS0 vt100
s1:12345:respawn:/sbin/agetty 9600 ttyS1 vt100

emerge net-misc/chrony net-misc/dhcpcd
rc-update add chronyd default
emerge --verbose sys-boot/grub
grub-install --target=x86_64-efi --efi-directory=/boot
grub-mkconfig -o /boot/grub/grub.cfg
exit
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot

# POST-INSTALLATIONS
emerge app-editors/vim app-admin/sudo

# EXPERIMENTOS
# Desabilitar conexão USB
CONFIG_USB_SUPPORT=n
```

## Hyper-V Kernel Customization

* A partir da Config padrão de `make menuconfig` do Gentoo, juntei as configs do Hyper-V
da discussão de Wei Liu e da doc [Gentoo - Hyper-V](https://wiki.gentoo.org/wiki/Hyper-V)

* Configuração completa funcional se encontra [aqui](.config.gentoo.hyperv)

## Hyper-V Linux Menu-Config by Liu

* Segundo Wei Liu <liuwe@microsoft.com> em https://lore.kernel.org/lkml/20191008161001.lwiduq4mlyjumw5b@debian/T/. Recusado por Masahiro Yamada <yamada.masahiro@socionext.com>

```makefile
CONFIG_NET=y
CONFIG_NET_CORE=y
CONFIG_NETDEVICES=y
CONFIG_BLOCK=y
CONFIG_BLK_DEV=y
CONFIG_NETWORK_FILESYSTEMS=y
CONFIG_INET=y
CONFIG_TTY=y
CONFIG_SERIAL_8250=y
CONFIG_SERIAL_8250_CONSOLE=y
CONFIG_IP_PNP=y
CONFIG_IP_PNP_DHCP=y
CONFIG_BINFMT_ELF=y
CONFIG_PCI=y
CONFIG_PCI_MSI=y
CONFIG_DEBUG_KERNEL=y
CONFIG_VIRTUALIZATION=y
CONFIG_HYPERVISOR_GUEST=y
CONFIG_PARAVIRT=y
CONFIG_ACPI=y
CONFIG_HYPERV=y
CONFIG_VSOCKETS=y
CONFIG_HYPERV_VSOCKETS=y
CONFIG_64BIT=y
CONFIG_PCI_HYPERV=y
CONFIG_PCI_HYPERV_INTERFACE=y
CONFIG_HYPERV_STORAGE=y
CONFIG_HYPERV_NET=y
CONFIG_HYPERV_KEYBOARD=y
CONFIG_FB=y
CONFIG_FB_HYPERV=y
CONFIG_HID_HYPERV_MOUSE=y
CONFIG_HYPERV_TIMER=y
CONFIG_CONNECTOR=y
CONFIG_HYPERV_UTILS=y
CONFIG_HYPERV_BALLOON=y
CONFIG_HYPERV_IOMMU=y
```

* EXPANDINDO A CONFIG COM HYPER-V [PASSED]

```makefile
CONFIG_MODULES=y
CONFIG_PRINTK=y
CONFIG_BLK_DEV_INITRD=y
CONFIG_RD_GZIP=y
CONFIG_PROC_FS=y
CONFIG_MAGIC_SYSRQ=y
CONFIG_DEVTMPFS=y
CONFIG_DEVTMPFS_MOUNT=y
CONFIG_SMP=y
CONFIG_PARAVIRT_SPINLOCKS=y
CONFIG_SCSI_FC_ATTRS=y
CONFIG_DRM=y
CONFIG_DRM_FBDEV_EMULATION=y
```

## Linux Tiny QEMU

* Ref: https://z49x2vmq.github.io/2020/12/24/linux-tiny-qemu/

## GENTOO SETUP HYPOTHESIS

Dado um fluxo de comandos, podemos criar uma árvore

```
|- hyp 'format-disk'
  |- hyp 'gpt+efipart+efepartenabled+linuxswappart+ext4fsrootpart'
    |- parted -s /dev/sda mktable GPT
    |- parted -s /dev/sda mkpart primary fat32 1M 250M
    |- parted -s /dev/sda mkpart primary linux-swap 250M 4G
    |- parted -s /dev/sda mkpart primary ext4 4G 100%
    |- parted -s /dev/sda set 1 esp on
```
