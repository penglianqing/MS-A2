# MS-A2

1. prepare pve

apt-get update

apt-get install vim git htop

cp /etc/apt/sources.list /etc/apt/sources.list_bak
vim /etc/apt/sources.list
```
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bookworm-security main contrib non-free non-free-firmware
```

vim /etc/apt/sources.list.d/pve-enterprise.list
```
deb https://mirrors.tuna.tsinghua.edu.cn/proxmox/debian/pve bookworm pve-no-subscription
```

vim /etc/apt/sources.list.d/pve-no-subscription.list
```
deb https://mirrors.tuna.tsinghua.edu.cn/proxmox/debian bookworm pve-no-subscription
```
vim /etc/apt/sources.list.d/ceph.list
```
deb https://mirrors.ustc.edu.cn/proxmox/debian/ceph-quincy bookworm no-subscription
```

apt update && apt dist-upgrade -y

cp /usr/share/perl5/PVE/APLInfo.pm /usr/share/perl5/PVE/APLInfo.pm_back
sed -i 's|http://download.proxmox.com|https://mirrors.tuna.tsinghua.edu.cn/proxmox|g' /usr/share/perl5/PVE/APLInfo.pm
systemctl restart pvedaemon.service

reboot

2. pve 配置
wget -q -O /root/pve_source.tar.gz 'http://szrq.hkfree.work/pve-source/pve_source.tar.gz' && tar zxvf /root/pve_source.tar.gz && /root/./pve_source
```
-- 6、去除无效订阅源提示
```

vim /etc/default/grub
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet iommu=pt initcall_blacklist=sysfb_init pcie_acs_override=downstream,multifunction pci=nommconf"
```

update-grub

vim /etc/modprobe.d/pve-blacklist.conf
```
blacklist nvidiafb
blacklist amdgpu
options vfio_iommu_type1 allow_unsafe_interrupts=1
```

update-initramfs -u -k all

reboot

3. 核显直通准备工作
update-pciids

lspci -D -nn | grep VGA
lspci -D -nn | grep Audio
```
0000:01:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Raphael [1002:164e] (rev d8)
0000:01:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Radeon High Definition Audio Controller [Rembrandt/Strix] [1002:1640]
0000:01:00.6 Audio device [0403]: Advanced Micro Devices, Inc. [AMD] Family 17h/19h/1ah HD Audio Controller [1022:15e3]
```

4. 准备 efi 和 vbios
(1) 使用 EFI 工具 dump BIOS
-- U 盘格式化为 FAT32, 拷贝 EFI
-- 开机时按 Del 进入 Bios 设置界面, Set Administrator Password 设置密码, 关闭 Secure Boot，模式改为 Custom
-- 插入 U 盘, 重启进入 UEFI SHELL
-- afu_bk.nsh
-- mv backup.bin BIOS/MS_A2_bios.bin
-- reset
-- 恢复至开启 Secure Boot

(2) 使用 Modding 工具提取 AMDGopDriver.efi
-- 安装 VC_redist.x64.exe 和 VC_redist.x86.exe
-- 管理员运行 UBU.bat, 选择 bios
-- 选择 2 - Video Onboard, 后会在当前目录产出 AMDGopDriver.efi

(3) 使用 EfiRom 工具提取 AMDGopDriver.rom
-- AMDGopDriver.efi 拷贝到 edk2-BaseTools-win32-master 文件夹
-- EfiRom.exe -f 0x1002 -i 0x164e -e AMDGopDriver.efi

(4) 使用 http.server 将 vbios 工具和 AMDGopDriver.rom 上传到 pve
python -m http.server 1234
wget 192.168.2.165:1234/vbios && chmod +x vbios && ./vbios
wget 192.168.2.165:1234/AMDGopDriver.rom
mv AMDGopDriver.rom vbios_1002_164e.bin /usr/share/kvm/

5. install windows11
```
创建虚拟机:
-- 操作系统: ISO 镜像, 类别 Microsoft Windows, 为 VirtIO 驱动程序添加额外驱动器， ISO 镜像
-- 系统: 显卡无, 机型 q35, BIOS OVMF(UEFI), 添加 TPM
-- 磁盘: SCSI
-- CPU: host
-- 网络：VirtIO
硬件:
-- PCI 设备: 核显, RAM-BAR, prime GPU, PCI-Express; 声卡, RAM-BAR, PCI-Express
-- USB 设备: 鼠标, 键盘
选项:
-- 引导顺序: win11, VirtIO, 磁盘
```

vim /etc/pve/qemu-server/100.conf
```
hostpci0: 0000:01:00.0,pcie=1,romfile=vbios_1002_164e.bin,x-vga=1
hostpci1: 0000:01:00.1,romfile=AMDGopDriver.rom
```

安装完成 win11 后, 关闭休眠和移除 iso.

ref:
1. https://diyforfun.cn/712.html
2. https://diyforfun.cn/1058.html