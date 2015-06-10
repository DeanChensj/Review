title: Compiling Linux Kernel
date: 2015-03-06 11:20:02
categories:
- Operating System
tags:
- Linux Kernel

---
To be prepared to make changes to the kernel, it's necessary to give it a try to compile the linux kernel successfully.

###Environment
* VMware Fusion 7
* Ubuntu 12.04.5
* Root permission

<!-- more -->


###Objective
Initially, the kernel version of the Ubuntu 12.04.5 is 3.13.0-32-generic according to 
```bash
uname -r
```
If you wanna check your ubuntu version, use
```bash
cat /etc/issue
```
Our goal is to update the version of kernel to newer one, in this case, 3.18.

###Compile the Linux Kernel
#####1. Download the kernel from [here](http://www.kernel.org).
#####2. Copy to a directory and extract.
```bash
cp linux-3.X.X.tar.xz /usr/src
cd /usr/src
xz -d linux-3.X.X.tar.xz
tar -xf linux-3.X.X.tar
cd linux-3.X.X
```

#####3. If not the first time to compile
```bash
make clean
make mrproper
```
#####4. Configuration
```bash
make menuconfig
```
There might be a tiny prob here saying
```bash
fatal error: curses.h: No such file or directory
```
Solution under Ubuntu:
```bash
sudo apt-get install libncurses5-dev libncursesw5-dev
```

#####5. Compile
```bash
make
make modules_install
make install
```
Suggestion here: enlarge the memory size of your virtual system, which may help reduce the compiling time dramatically.

It takes hours to compile with 1 core and 1-GB memory with GUI turned on...

#####6. Update GRUB
```bash
cd /boot
mkinitramfs 3.18.8 -o initrd.img-3.18.8
cp /usr/src/linux-3.18.8/arch/x86_64/boot/bzImage /boot/vmlinuz-3.18.8
ln -s /boot/System.map-3.18.8 /boot/System.map
sudo update-grub2
```

#####7. Reboot and Check!