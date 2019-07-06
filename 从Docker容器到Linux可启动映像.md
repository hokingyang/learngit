#从Docker容器到Linux可启动映像#

本文要谈谈容器和linux发行版之间的关系，之前网上并没有任何相关的文档，然而我认为这确实是深入了解系统内部知识所必需的。Dockerfile中最简单的一行，例如：From debian:latest，在容器映像里是如何实现的，或者说，是否可以创建一个可启动的linux映像，功能和容器一模一样？那么就需要深入容器root文件系统，看起来好像很简单只需要运行docker export，但是最终实现这个功能还是需要很多额外步骤的。。。

##原理##
首先，简单讨论一下枯燥的理论。安装完成的linux系统是什么样的？基本是上一个linux kernel二进制文件，初始化ramdisk二进制文件和用户态程序和库（一般是GNU核心应用和BootLoader）构成的。
【图一】

可以运行tree -L 1 / 查看root目录结构：
```
$ cat /etc/os-release | grep NAME
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
NAME="Debian GNU/Linux"

$ tree -L 1 /
/
├── bin
├── boot
├── data
├── dev
├── etc
├── home
├── initrd.img -> boot/initrd.img-4.9.0-9-amd64  # initial ramdisk
├── lib
├── lib64
├── media
├── mnt
├── opt
├── proc
├── root
├── run
├── sbin
├── srv
├── sys
├── tmp
├── usr
├── var
└── vmlinuz -> boot/vmlinuz-4.9.0-9-amd64        # kernel binary
```

现在集中看看Docker。Docker遵循OS层面虚拟化标准包装了容器，也就是容器运行重用了宿主机linux发行版的kernel，而用户态空间却是完全分隔的。
【图二】

查看一个启动的容器：
```
$ docker run -it debian:latest bash

root@62376e4c451b:/# cat /etc/os-release | grep NAME
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
NAME="Debian GNU/Linux"

root@62376e4c451b:/# apt-get update && apt-get install -y tree
root@62376e4c451b:/# tree -L 1
.
|-- bin
|-- boot
|-- dev
|-- etc
|-- home
|-- lib
|-- lib64
|-- media
|-- mnt
|-- opt
|-- proc
|-- root
|-- run
|-- sbin
|-- srv
|-- sys
|-- tmp
|-- usr
`--  var

19 directories, 0 files

root@62376e4c451b:/# tree -L 1 /boot
boot/

0 directories, 0 files
```
以上内容可以看到Debian的用户空间，但没有kernel&Co.。这不是唯一不同，Linux以PID1运行初始化进程，而Docker容器则是用shell或者用户定义可执行程序作为PID1进程。因此，需要把这些差异找出来使它越来越像一个完整的Debian安装。

##实践##
首先创建一个Dockerfile:
```
FROM debian:stretch
```
执行```docker build -t mydebian``` 开始创建，用wagoodman/dive: dive mydebian进入影像中查看。
【图三】

可以看出，尽管包含Debian全功能的用户空间，其映像总大小也就101MB；因为还是没有kernel，需要下载安装kernel二进制代码。在Dockerfile中添加：
```
ROM debian:stretch
RUN apt-get -y update
RUN apt-get -y install --no-install-recommends \
  linux-image-amd64
```

重建并检视新映像：
【图四】

看起来linux-image-amd64占用了232MB，其中24MB属于/boot目录，200MB属于/lib，继续深入。。。。
【图五】

从中看到/boot/vmlinuz-4.9.0-9-amd64只有4.2MB，初始化ramdisk /boot/initrd.img-4.9.0-9-amd64有16MB，其它大约200MB位于/lib/modules下的核心调用，包含各种驱动。

下一步可以启动初始化进程 - systemd:
```
FROM debian:stretch
RUN apt-get -y update
RUN apt-get -y install --no-install-recommends \
  linux-image-amd64
RUN apt-get -y install --no-install-recommends \
  systemd-sysv
```
重建并检视新映像：
【图六】

多了30MB多，快发现了。导出容器文件系统看看：
```$ CID=$(docker run -d mydebian /bin/true)
$ docker export -o linux.tar ${CID}

# List files in the archive:
$ tar -tf linux.tar | grep -E '^[^/]*/?$'
.dockerenv
bin/
boot/
dev/
etc/
home/
initrd.img
initrd.img.old
lib/
lib64/
media/
mnt/
opt/
proc/
root/
run/
sbin/
srv/
sys/
tmp/
usr/
var/
vmlinuz
vmlinuz.old
```
从tar包中生成一个可启动映像，以下命令行可以直接在linux宿主机上执行。因为我用MacOS，需要启动另外一个Debian容器作为创建主机。
```
$ docker run -it -v `pwd`:/os:rw            \
    --cap-add SYS_ADMIN --device /dev/loop0 \
    debian:stretch bash
```
首先创建一个足够大小的映像:
```
$ IMG_SIZE=$(expr 1024 \* 1024 \* 1024)
$ dd if=/dev/zero of=/os/linux.img bs=${IMG_SIZE} count=1
```
创建分区：
```
$ sfdisk /os/linux.img <<EOF
label: dos
label-id: 0x5d8b75fc
device: new.img
unit: sectors

linux.img1 : start=2048, size=2095104, type=83, bootable
EOF

Checking that no-one is using this disk right now ... OK

Disk /os/linux.img: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

>>> Script header accepted.
>>> Script header accepted.
>>> Script header accepted.
>>> Script header accepted.
>>> Created a new DOS disklabel with disk identifier 0x5d8b75fc.
/os/linux.img1: Created a new partition 1 of type 'Linux' and of size 1023 MiB.
/os/linux.img2: Done.

New situation:

Device         Boot Start     End Sectors  Size Id Type
/os/linux.img1 *     2048 2097151 2095104 1023M 83 Linux

The partition table has been altered.
Syncing disks.
```
挂载映像，用ext3文件系统格式化，拷入tar包内容：
```
$ OFFSET=$(expr 512 \* 2048)
$ losetup -o ${OFFSET} /dev/loop0 /os/linux.img
$ mkfs.ext3 /dev/loop0
$ mkdir /os/mnt
$ mount -t auto /dev/loop0 /os/mnt/
$ tar -xvf /os/linux.tar -C /os/mnt/
```
最后，安装BootLoader，卸载文件系统:
```
$ apt-get update -y
$ apt-get install -y extlinux

$ extlinux --install /os/mnt/boot/
$ cat > /os/mnt/boot/syslinux.cfg <<EOF
DEFAULT linux
  SAY Now booting the kernel from SYSLINUX...
 LABEL linux
  KERNEL /vmlinuz
  APPEND ro root=/dev/sda1 initrd=/initrd.img
EOF

$ dd if=/usr/lib/syslinux/mbr/mbr.bin of=/os/linux.img bs=440 count=1 conv=notrunc

$ umount /os/mnt
$ losetup -D
```
最终，在工作目录下生成了一个名为linux.img的映像。

##结论##
刚才生成了一个linux映像，可以到一个实际或者虚拟机上，例如，可以使用本映像启动一个QEMU虚机:
```
$ qemu-system-x86_64 -drive file=linux.img,index=0,media=disk,format=raw
```
【图七】
或者将映像转化成VDI格式的VirtualBox虚机：
```
$ VBoxManage convertfromraw --format vdi linux.img linux.vdi
```

##彩蛋:tiny Alpine Linux##
如果Debian 大约400MB大小太大，Alpine提供了一个功能类似但只有100MB左右的发行版:
```
FROM alpine:3.9.4
RUN apk update
RUN apk add linux-virt
RUN apk add openrc
```
【图八】

用以上方法可以自动生成Debian和Alpine发行版，可以在如下地址找到它们:[GitHub iximiuz/docker-to-linux](https://github.com/iximiuz/docker-to-linux)