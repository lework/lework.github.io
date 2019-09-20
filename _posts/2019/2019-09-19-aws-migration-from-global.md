---
layout: post
title: '将Debian AMI从全球AWS区域迁移到中国区域'
date: '2019-09-19 17:00:00'
category: aws
tags: aws
author: lework
---
* content
{:toc}

因为中国区域的`Debian`版本没有新版的,是不是很无语。云平台竟然连 linux 发行版的系统都没的。。。

## 大概的步骤

1. 首先,在`AWS`东京地区启动`Debian`实例,将适当大小的`EBS`卷挂载到该 Debian 实例上,然后使用`dd`命令将`Debian`实例的整个根卷作为文件保存到该`EBS`卷。

2. 之后,将文件复制到`cn-north-1`区域中的实例.在该`cn-north-1`区域的实例中,使用`dd`命令将该文件写入`EBS`卷,然后为该`EBS`卷创建快照,并使用该快照在`cn-north-1`区域中创建`AMI`。
3. 最后,用户可以使用该`AMI`启动`CentOS`实例,该实例与在东京地区运行的实例相同。




## 具体实施

1. 在海外创建一个 Debian 系统实例,本次选择东京地区

	![debian.png](/assets/images/aws/debian.png)

1. 创建一个挂载卷,用于存储系统镜像文件,并挂载到新建的实例上

    也可以在创建实例的时候一并建立  
    ![ebs_create.png](/assets/images/aws/ebs_create.png)

    将卷连接到实例中
    ![ebs_line.png](/assets/images/aws/ebs_line.png)

1. 连接实例后,可以看到系统中存在 2 个磁盘
   ![ebs_line.png](/assets/images/aws/disk.png)

    `/dev/xvda` 是系统盘, `/dev/xvdf`是我们挂载的数据磁盘

    格式化数据磁盘,并挂载到目录中

    ```bash
    mkfs.ext4 /dev/xvdf
    mount /dev/xvdf /mnt/
    ```

1. 使用`dd`命令将系统盘写入到数据磁盘目录文件中。

    ```bash
    dd if=/dev/xvda of=/mnt/root.img bs=1M
    ```

1. 在国内也创建一个 Amazon Linux 2 的实例,根磁盘需要设置 20g,用作存储 img,同样也挂载一块磁盘用于将 img 写入

    接着,将 img 传递到国内的实例中

    ```bash
    scp -i test.pem /mnt/root.img ec2-user@ec2-52-80-171-123.cn-north-1.compute.amazonaws.com.cn:/tmp/
    ```

1. 在传送完毕后,进入国内的实例中。

    磁盘信息

    ```bash
    fdisk -l
    Disk /dev/nvme1n1: 20 GiB, 21474836480 bytes, 41943040 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: dos
    Disk identifier: 0xf6897bce

    Device         Boot Start      End  Sectors Size Id Type
    /dev/nvme1n1p1 *     2048 16777215 16775168   8G 83 Linux

    Disk /dev/nvme0n1: 20 GiB, 21474836480 bytes, 41943040 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disklabel type: gpt
    Disk identifier: 33E98A7E-CCDF-4AF7-8A35-DA18E704CDD4
    
    Device           Start      End  Sectors Size Type
    /dev/nvme0n1p1    4096 41943006 41938911  20G Linux filesystem
    /dev/nvme0n1p128  2048     4095     2048   1M BIOS boot
    ```
    
    `/dev/nvme0n1` 是系统盘, `/dev/nvme1n1` 是挂载的数据盘
    
    使用 dd 命令,将 img 文件写入到`/dev/nvme1n1`数据盘中。
    
    ```bash
    dd if=/tmp/root.img of=/dev/nvme1n1 bs=1M oflag=direct
    ```
    
    接着，我们挂载数据盘，并删除系统用户的`authorized_keys`
    
    ```bash
    mount /dev/nvme1n1p1 /tmp/debian
    rm -f /tmp/debian/root/.ssh/authorized_keys
    umount /tmp/debian
    ```

1. 卸载完磁盘后, 在 aws 网页控制台上将 EBS 打上快照

	![ebs_create_snapshot.png](/assets/images/aws/ebs_create_snapshot.png)

1. 创建完快照后, 使用这个快照创建 ami 映像
   ![create_ami.png](/assets/images/aws/create_ami.png)
   创建完映像之后，我们就可以用这个映像来创建 debian ec2 了