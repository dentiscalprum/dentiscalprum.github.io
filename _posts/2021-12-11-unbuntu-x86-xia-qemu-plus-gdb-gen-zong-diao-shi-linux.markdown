---
title: "Ubuntu(x86) 下 qemu+gdb 跟踪调试 Linux"
date: 2021-12-11 17:45:39 +0800
layout: article
key: qemu_gdb_ubuntu
categories: 
---
本文将从两个方面来介绍，分别是配置环境和编译源码（安装 QEMU + gdb 及编译 Linux 源码及根文件系统），最后进行简单的跟踪调试。

___注 ：本文使用的 linux 内核版本为 5.15___


## 配置环境及编译源码

### 安装 gdb 和 QEMU

我安装的 Ubuntu 系统自带 gdb，因此只需要安装 QEMU，直接用 apt 进行安装

```bash
sudo apt install -y qemu-system-x86
```

### 编译调试版 linux 内核（5.15）

1. 安装依赖

   ```bash
   sudo apt install -y make build-essential libncurses5-dev flex bison libelf-dev libssl-dev
   ```

2. 从官网下载 linux 内核源码 [https://www.kernel.org/](https://www.kernel.org/)

3. 进入图形界面配置编译选项

   ```bash
   make menuconfig
   ```

   - 重要的配置项有:

     ```bash
     Kernel hacking  --->
       _*_ kernel debugging
       Compile-time checks and compiler options --->
		 [*] Compile the kernel with debug info
		 [*] Provide GDB scripts for kernel debugging
     ```

   - 关闭 **Randomize the address of the kernel image**, 否则将会导致打断点失败

     ```bash
     Processor type and features --->
		[] Randomize the address of the kernel image (KASLR)
     ```

4. 编译

   ```bash
   make -j$(nproc)
   ```

5. 在编译过程中可能会出现如下错误

 - 针对错误

   ```bash
   make[1]: *** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop.
   make: *** [Makefile:1868: certs] Error 2
   ```

   打开 ___.config___ 文件，将 ___CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem"___ 改为 ___CONFIG_SYSTEM_TRUSTED_KEYS=""___

- 和上述错误类似，针对错误

  ```bash
  make[1]: *** No rule to make target 'debian/canonical-revoked-certs.pem', needed by 'certs/x509_revocation_list'.  Stop.
  make: *** [Makefile:1868: certs] Error 2
  ```

  打开 ___.config___ 文件, 将 ___CONFIG_SYSTEM_REVOCATION_KEYS___ 置空, 即 ___CONFIG_SYSTEM_REVOCATION_KEYS=""___

- 针对 warning

  ```bash
  fs/jffs2/xattr.c: In function ‘jffs2_build_xattr_subsystem’:
  fs/jffs2/xattr.c:887:1: warning: the frame size of 1128 bytes is larger than 1024 bytes [-Wframe-l
  arger-than=]
    887 | }
        | ^
  ```

  配置编译选项 make menuconfig, 将 Warn for stack frames larger than 设为 2048

  ```bash
  Kernel hacking --->
	Compile-time checks and compiler options --->
	(2048) Warn for stack frames larger than
  ```

___通过以上步骤，就会生成  <code>bzImage</code> 和 <code>vmlinux</code>___

```bash
bzImage 位于 linux-5.15/arch/x86_64/boot
vmlinux 位于 linux-5.15
```

### busybox 安装最小文件系统

1. 下载源码 [https://busybox.net/](https://busybox.net/)

2. 进入 busybox 目录配置编译选项

   ```bash
   make defconfig
   make menuconfig
   ```

    进入图形界面配置编译选项:

   ```bash
   Settings --->
	[*] Build static binary (no shared libs)
   ```

3. 编译安装

   ```bash
   make -j$(nproc)
   make install
   ```

   - 注: 虽然有很多 warning ，影响不大
   - 最终会产生 _install 文件夹，这里是需要的文件

4. 生成 initrd

   - 首先复制_install文件夹中的内容

     ```bash
     mkdir ramdisk && cd ramdisk
     cp -r ../busybox-1.34.1/_install/* ./
     ```

   - 设置初始化进程init（建立一个软链接，一定不能直接复制过去）

     ```
     cd ramdisk
     ln -s /bin/busybox init
     ```

   - 设定一些程序运行所需要的文件夹

     ```bash
     mkdir -pv {bin,sbin,etc,proc,sys,usr/{bin,sbin},dev}
     ```

   - init程序首先会访问 etc/inittab 文件，因此，我们需要编写 inittab，指定开机需要启动的所有程序

     ```bash
     cd etc
     touch inittab
     chmod +x inittab
     vim inittab
     ```

     其中的内容为

     ```bash
     ::sysinit:/etc/init.d/rcS
     ::askfirst:-/bin/sh
     ::restart:/sbin/init
     ::ctrlaltdel:/sbin/reboot
     ::shutdown:/bin/umount -a -r
     ::shutdown:/sbin/swapoff -a
     ```

   - 从inittab文件中可以看出，首先执行的是/etc/init.d/rcS脚本，因此，我们生成初始化脚本

     ```bash
     mkdir init.d
     cd init.d
     touch rcS
     chmod +x rcS
     vim rcS
     ```

     rcS文件的内容如下所示

     ```bash
     #!/bin/sh

     mount proc
     mount -o remount,rw /
     mount -a
     clear
     echo "My Tiny Linux Start :D ......"
     ```

   - 在rcS脚本中，mount -a 是自动挂载 /etc/fstab 里面的东西，可以理解为挂在文件系统，因此我们还需要编写 fstab文件来设置我们的文件系统

     ```bash
     cd ramdisk/etc/
     vim fstab
     ```

     fstab文件内容如下

     ```bash
     # /etc/fstab
     proc            /proc        proc    defaults          0       0
     sysfs           /sys         sysfs   defaults          0       0
     devtmpfs        /dev         devtmpfs  defaults          0       0
     ```

   - 至此，我们完成了RAM DISK中相关文件的配置，可以压缩生成文件镜像了

     ```bash
     cd ramdisk
     find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../initramfs.img
     ```

   ___最后生成的<code>initramfs.img</code>就是我们的根文件系统___

## 跟踪调试

- QEMU 阻塞运行 Linux 虚拟机

  ```bash
  qemu-system-x86_64 -s -S -kernel linux-5.15/arch/x86_64/boot/bzImage -initrd initramfs.img -nographic -append "console=ttyS0"
  ```

  - -s 选项是 -gdb 的简写，会在本地:1234启动一个 GDB 服务
  - -S 暂停虚拟机，等待 gdb 执行 continue 指令

- gdb 连接 gdb server

  ```bash
  cd linux-5.15
  gdb vmlinux
  (gdb) target remote :1234
  ```

  连接到 GDB 服务

  ```cpp
  (gdb) b start_kernel
  (gdb) continue
  ```

  就停止到了 start_kernel 位置
## 参考博客
- [使用 VSCode + qemu 搭建 Linux 内核调试环境](https://howardlau.me/programming/debugging-linux-kernel-with-vscode-qemu.html)
- [QEMU+gdb调试Linux内核全过程](https://blog.csdn.net/jasonLee_lijiaqi/article/details/80967912?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-2.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-2.no_search_link)

