# BusyBox

本仓库 Fork 自 https://github.com/mirror/busybox

## 什么是 BusyBox

BusyBox 将众多常见 UNIX 工具的微型版本整合到一个单一的小型可执行文件中。它为通常在 bzip2、coreutils、dhcp、diffutils、e2fsprogs、file、findutils、gawk、grep、inetutils、less、modutils、net-tools、procps、sed、shadow、sysklogd、sysvinit、tar、util-linux 和 vim 中找到的大多数工具提供了极简替代方案。BusyBox 中的工具通常比其功能齐全的对应工具选项更少；然而，所包含的选项提供了预期的功能，并且行为与它们的大型对应工具非常相似。

---

## 编译 BusyBox

### 1. 获取源码

```bash
git clone https://github.com/mirror/busybox.git
cd busybox
```

### 2. 环境准备

编译 BusyBox 需要基本的构建工具链（gcc、make、ncurses-dev 用于 menuconfig）：

```bash
# Ubuntu/Debian
sudo apt install build-essential libncurses5-dev bison flex

# 如果是 ARM64 交叉编译，还需安装交叉工具链
sudo apt install gcc-aarch64-linux-gnu
```

### 3. 配置（三种方式）

与 Linux 内核构建类似，BusyBox 使用 Kconfig 配置系统：

```bash
# 方式一：图形化配置（推荐，需要 ncurses）
make menuconfig

# 方式二：文本交互式配置
make config

# 方式三：基于预设 defconfig 配置
make defconfig              # 启用所有通用功能（最大配置）
make allnoconfig             # 全部禁用，从零开始只加需要的
make allyesconfig            # 启用绝对所有功能（含调试）
```

配置关键选项说明：
- `CONFIG_PREFIX`: 安装路径（默认 `./_install`）
- `CONFIG_CROSS_COMPILER_PREFIX`: 交叉编译工具链前缀
- `CONFIG_STATIC`: 静态链接编译（initramfs 场景必须启用）

**configs/** 目录下提供了平台预设配置，例如：
- `android_defconfig` - Android 环境
- `freebsd_defconfig` - FreeBSD 环境
- `cygwin_defconfig` - Cygwin 环境

### 4. 编译与安装


**本地编译完整示例：**

```bash
make defconfig
make menuconfig          # 根据需要调整
make -j$(nproc)
make CONFIG_PREFIX=./rootfs install
```

**ARM64 交叉编译完整示例：**

```bash
# 导出交叉编译环境变量
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

# 创建最小化配置
make allnoconfig
make menuconfig          # 按需要勾选 applet

# 编译时指定交叉编译器
make -j$(nproc) CROSS_COMPILE=aarch64-linux-gnu-

#编译产物验证  
file busybox  
#应输出: busybox: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, for GNU/Linux 3.7.0, stripped  

**静态编译（initramfs 必需！！！）:**

```bash
# 在 menuconfig 中设置：
#   Busybox Settings → Build Options → Build BusyBox as a static binary (no shared libs) → [*] yes
# 或直接修改 .config：
sed -i 's/.*CONFIG_STATIC.*/CONFIG_STATIC=y/' .config
make -j$(nproc)
```

---

## 安装与使用

### 创建 initramfs 目录结构

```bash
mkdir -p ./initramfs/{bin,sbin,etc/init.d,dev,proc,sys,tmp,root}
```  

### 安装 busbox  

```bash
make CONFIG_PREFIX=./initramfs install
```  

## 创建 /init 脚本(仓库中默认已创建, 无需重复)  

```bash
  cat > ./initramfs/init << 'EOF'
  #!/bin/sh

  mount -t proc none /proc
  mount -t sysfs none /sys
  mount -t devtmpfs none /dev

  echo "Welcome to BusyBox initramfs"

  exec setsid cttyhack sh
  EOF

  chmod +x ./initramfs/init  
```  

## 创建必要的设备节点  

```bash
  sudo mknod -m 622 ./initramfs/dev/console c 5 1
  sudo mknod -m 666 ./initramfs/dev/null c 1 3
  sudo mknod -m 666 ./initramfs/dev/tty c 5 0
```  

## 打包为 initramfs cpio  

```bash  
  cd initramfs
  find . -not -name 'initramfs.cpio.gz' | cpio -o -H newc | gzip > ./initramfs.cpio.gz
```

# U-Boot 启动示例(以飞腾 E2000D 为例)  

使用 MDFT-E2KD-CPUEVK-A0-C 板卡进行启动，详情如下：  

```bash  
#加载系统镜像  
tftpboot 0x90200000 Image; tftpboot 0x92000000 initramfs.cpio.gz; tftpboot 0x90000000 e2000d-power-board.dtb  

#设置启动参数
setenv bootargs console=ttyAMA1,115200 earlycon=pl011,0x2800d000  

#启动镜像  
booti 0x90200000 0x92000000:{initramfs_size} 0x90000000
```  

## 当前版本状态

### 常用命令速查

#### 系统信息

| 命令 | 用法示例 | 说明 |
|------|---------|------|
| `uname` | `uname -a` | 查看内核版本、架构信息 |
| `hostname` | `hostname myboard` | 查看/设置主机名 |
| `uptime` | `uptime` | 查看系统运行时间及负载 |
| `dmesg` | `dmesg \| tail -20` | 查看内核日志 |
| `free` | `free -m` | 查看内存使用情况 |
| `iostat` | `iostat -k 2 3` | 查看 CPU 和磁盘 I/O 统计 |
| `cat /proc/cpuinfo` | `grep -c processor /proc/cpuinfo` | 查看 CPU 信息/核心数 |
| `cat /proc/partitions` | `cat /proc/partitions` | 查看分区及大小 |
| `lspci` | `lspci`；`lspci -k` | 查看 PCI 总线设备 |
| `lsusb` | `lsusb`；`lsusb -t` | 查看 USB 总线设备（树形） |
| `lsscsi` | `lsscsi` | 查看 SCSI 设备列表 |
| `acpid` | `acpid -d` | ACPI 事件守护进程 |

#### 进程管理

| 命令 | 用法示例 | 说明 |
|------|---------|------|
| `ps` | `ps aux` | 查看进程列表 |
| `top` | `top` | 实时进程监控（需在 menuconfig 中勾选） |
| `kill` | `kill -9 1234`；`killall process_name` | 终止进程 |
| `pidof` | `pidof sshd` | 按名称查找进程 PID |
| `nice` | `nice -n 10 command` | 调整进程优先级 |

#### 文件操作

| 命令 | 用法示例 | 说明 |
|------|---------|------|
| `ls` | `ls -la /` | 列出目录内容 |
| `cp` | `cp -a /src /dst` | 复制文件/目录 |
| `mv` | `mv /old /new` | 移动/重命名 |
| `rm` | `rm -rf /tmp/*` | 删除文件/目录 |
| `cat` | `cat /etc/inittab` | 查看文件内容 |
| `head`/`tail` | `tail -f /var/log/messages` | 查看文件头/尾部 |
| `du` | `du -sh /usr` | 查看目录占用空间 |
| `df` | `df -h` | 查看磁盘使用情况 |
| `stat` | `stat /bin/busybox` | 查看文件详细属性 |
| `find` | `find / -name "*.conf"` | 搜索文件 |
| `grep` | `grep -r "error" /var/log/` | 内容搜索 |
| `sed` | `sed 's/old/new/g' file` | 文本替换 |
| `awk` | `awk '{print $1}' file` | 文本处理 |
| `vi` | `vi /etc/init.d/rcS` | 编辑文件 |
| `tar` | `tar -xzf archive.tar.gz` | 打包/解包 |
| `gzip`/`gunzip` | `gzip file`；`gunzip file.gz` | 压缩/解压 |

#### 磁盘与文件系统

| 命令 | 用法示例 | 说明 |
|------|---------|------|
| `mount` | `mount -t ext4 /dev/sda1 /mnt` | 挂载文件系统 |
| `umount` | `umount /mnt` | 卸载文件系统 |
| `blkid` | `blkid` | 查看块设备 UUID/FSTYPE |
| `blockdev` | `blockdev --getsize64 /dev/sda` | 查询块设备大小 |
| `fdisk` | `fdisk /dev/sda` | 磁盘分区工具 |
| `mkfs.ext2` | `mkfs.ext2 /dev/sda1` | 格式化 ext2 文件系统 |
| `mkfs.vfat` | `mkfs.vfat /dev/sda1` | 格式化 FAT/VFAT 文件系统 |
| `fsck` | `fsck /dev/sda1` | 文件系统检查修复 |
| `losetup` | `losetup -f`；`losetup /dev/loop0 file.img` | 管理 loop 设备 |
| `sync` | `sync` | 同步缓存数据到磁盘 |
| `swapon`/`swapoff` | `swapon /swapfile` | 启用/禁用交换空间 |
| `hdparm` | `hdparm -I /dev/sda` | 查看/设置硬盘参数 |
| `eject` | `eject /dev/cdrom` | 弹出光驱 |
| `mt` | `mt -f /dev/st0 status` | 磁带设备控制 |
| `partx` | `partx /dev/sda` | 通知内核分区表变化 |

#### 网络

| 命令 | 用法示例 | 说明 |
|------|---------|------|
| `ifconfig` | `ifconfig eth0 192.168.1.100 netmask 255.255.255.0 up` | 配置网络接口 |
| `route` | `route add default gw 192.168.1.1` | 管理路由表 |
| `ping` | `ping -c 4 8.8.8.8` | 测试网络连通性 |
| `wget` | `wget http://example.com/file` | 下载文件 |
| `tftp` | `tftp -g -r file 192.168.1.1` | TFTP 下载文件 |
| `telnet` | `telnet 192.168.1.1` | 远程登录 |
| `nc` | `nc -l -p 1234` | 网络调试瑞士军刀 |
| `netstat` | `netstat -tlnp` | 查看网络连接状态 |
| `nslookup` | `nslookup www.example.com` | DNS 查询（需编译时启用） |
| `traceroute` | `traceroute 8.8.8.8` | 路由路径追踪 |
| `ntpd` | `ntpd -p pool.ntp.org` | NTP 时间同步 |
| `httpd` | `httpd -p 80 -h /var/www` | 轻量级 HTTP 服务器 |
| `ftpd` | `tcpsvd 0 21 ftpd -w /tmp` | 轻量级 FTP 服务器 |

#### 系统管理

| 命令 | 用法示例 | 说明 |
|------|---------|------|
| `init` | `/sbin/init` | 系统初始化进程 |
| `reboot`/`halt`/`poweroff` | `reboot` | 重启/关机 |
| `date` | `date -s "2025-01-01 12:00:00"` | 查看/设置系统时间 |
| `hwclock` | `hwclock -w` | 同步系统时间到硬件时钟 |
| `modprobe` | `modprobe module_name` | 加载内核模块 |
| `lsmod` | `lsmod` | 查看已加载的内核模块 |
| `rmmod` | `rmmod module_name` | 卸载内核模块 |
| `mdev` | `mdev -s` | 设备节点管理（devtmpfs 替代方案） |
| `sysctl` | `sysctl -w vm.swappiness=10` | 内核参数调整 |
| `chmod` | `chmod +x script.sh` | 修改文件权限 |
| `chown` | `chown root:root file` | 修改文件所有者 |
| `passwd` | `passwd root` | 修改密码 |
| `adduser`/`deluser` | `adduser username` | 添加/删除用户 |
| `crontab` | `crontab -e` | 定时任务管理 |
| `hexdump` | `hexdump -C file` | 十六进制查看文件 |
| `setserial` | `setserial /dev/ttyS0` | 查看/设置串口参数 |
| `readahead` | `readahead /lib/*` | 预读文件到页缓存 |
| `usleep` | `usleep 500000` | 微秒级休眠 |
| `i2cdetect` | `i2cdetect -l`；`i2cdetect -y 0` | 探测 I2C 总线及设备 |
| `i2cget` | `i2cget -y 0 0x50 0x00` | 读取 I2C 设备寄存器 |
| `i2cset` | `i2cset -y 0 0x50 0x00 0xff` | 写入 I2C 设备寄存器 |
| `i2cdump` | `i2cdump -y 0 0x50` | 导出 I2C 设备所有寄存器 |
| `i2ctransfer` | `i2ctransfer -y 0 w1@0x50 0x00 r1` | I2C 底层传输操作 |
---

### BusyBox 不支持的命令及替代方案

| 不支持的命令 | 替代方案 | 用法示例 |
|-------------|---------|---------|
| `lsblk` | `blkid`、`cat /proc/partitions`、`blockdev` | `blkid`；`cat /proc/partitions`；`blockdev --getsize64 /dev/mmcblk0` |
| `lscpu` | `cat /proc/cpuinfo` | `cat /proc/cpuinfo`；`grep -c processor /proc/cpuinfo`（核心数） |
| `ip addr` | `ifconfig` | `ifconfig eth0`；`ifconfig eth0 192.168.1.100 netmask 255.255.255.0 up` |
| `ip route` | `route` | `route`；`route add default gw 192.168.1.1` |
| `ip link` | `ifconfig` | `ifconfig eth0 up`；`ifconfig eth0 down` |
| `ss -tlnp` | `netstat` | `netstat -tlnp`；`netstat -an` |
| `journalctl` | `dmesg`、`cat /var/log/messages`、`logread` | `dmesg`；`cat /var/log/messages`；`logread` |
| `systemctl start/stop` | `/etc/init.d/` 脚本 | `/etc/init.d/sshd start`；`/etc/init.d/sshd stop` |
| `timedatectl` | `date`、`hwclock` | `date -s "2025-01-01 12:00:00"`；`hwclock -w`（写入硬件时钟） |
| `hostnamectl` | `hostname` | `hostname myboard`（临时）；写入 `/etc/hostname`（永久） |
| `htop` | `top` | `top`（BusyBox 内置，实时进程监控） |
| `pam-auth-update` | 不支持 PAM | BusyBox 不包含 PAM 认证框架，无法替代 |
| `apt`/`dpkg` | 不支持包管理 | 嵌入式环境通过交叉编译预置软件 |
| `lvm` | 不支持 | 需要从 util-linux 单独移植 |
| `mdadm` | 不支持 | 软件 RAID 管理需额外移植 |
| `iperf3` | 不支持 | 需交叉编译独立二进制 |
| `strace` | 不支持 | 需交叉编译独立二进制 |
| `perf` | 不支持 | 需配合内核从 Linux 源码编译 |
| `tracepath` | `traceroute` | `traceroute 8.8.8.8` |
| `nslookup` | `nslookup`（需编译时启用） | `nslookup www.example.com`（在 menuconfig → Networking Utilities 中启用） |    

## 其他  
  
  若在构建时出错 **error: 'IFLA_CAN_TERMINATION' undeclared(first use in this function)**，有两种方案解决：  

  - 方案一：提升编译器版本，实测会在 **gcc 7.5.x** 上出现  
  - 方案二：在 **.config** 文件中手动禁用 **CONFIG_FEATURE_IP_LINK_CAN** 选项或者直接将其删除















