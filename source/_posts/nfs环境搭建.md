---
title: nfs环境搭建
date: 2022-05-31 19:44:12
categories: 内核
tags: [环境搭建,文件系统,nfs]
---

# nfs环境搭建

我启动两台qemu虚拟机（linux kernel），一台作为服务器端，一台作为客户端搭建了NFS环境，并尝试进行了使用。

1. 在服务器端和客户端操作：安装nfs-server包（同时包括了客户端软件和服务器端软件）

   ```shell
   apt-get install nfs-kernel-server -y
   ```
   
2. 编辑服务器端的/etc/exports配置文件

   ```shell
   # vim /etc/exports,加入以下内容
   /tmp/ *(rw,no_root_squash,fsid=0)
   /tmp/s_test/ *(rw,no_root_squash,fsid=1)
   ```

3. 在服务器端操作：将设备格式化为特定的文件系统格式

   ```shell
   mkfs.ext4 -b 4096 -F /dev/sda
   ```

4. 在服务器端操作：创建共享文件夹

   ```shell
   mkdir /tmp/s_test
   ```

5. 在服务器端操作：挂载文件系统

   ```shell
   mount -t ext4 /dev/sda /tmp/s_test
   ```

6. 在服务器端操作：设置最多可以打开的文件描述符数量

   ```shell
   ulimit -n 65535
   ```

7. 在服务器端操作：重新共享所有目录（它使  /var/lib/nfs/xtab  和 /etc/exports 同步。 它將 /etc/exports中已删除的条目从 /var/lib/nfs/xtab 中删除，將内核共享表中任何不再有效的条目移除）

   ```shell
   exportfs -r
   ```

8. 在服务器端操作：关闭防火墙

   ```shell
   systemctl stop firewalld
   setenforce 0
   ```

9. 在服务器端操作：重启服务

   ```shell
   systemctl restart nfs-server.service
   systemctl restart rpcbind
   ```

10. 在服务器端操作：修改文件操作权限，使三种用户都具有对该文件夹的可读可写可执行权限

    ```shell
    chmod 777 /tmp/s_test
    ```

11. 在客户端操作：挂载nfs文件系统

    ```shell
    # ip地址用ifconfig查看，如果提示没有这个命令使用sudo apt install net-tools安装相应软件包
    # nfsv4
    mount -t nfs -o v4.1 192.168.122.210:/s_test /mnt
    # nfsv3 要写完整的源路径
    mount -t nfs -o v3 192.168.122.210:/tmp/s_test /mnt
    # nfsv2, nfs server 需要修改 /etc/nfs.conf, [nfsd] vers2=y
    mount -t nfs -o v2 192.168.122.210:/tmp/s_test /mnt
    ```