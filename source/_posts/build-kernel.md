---
title: build kernel
date: 2022-05-30 19:40:13
tags: kernel
---

1. [openEuler kernel source code repo](https://gitee.com/openeuler/kernel ):    branch OLK-5.10

   [linux kernel git repo](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git)

2. 应用该[patch](https://github.com/mufiye/mufiye_blog/blob/master/kernel_environment/0001-x86_64-debug.patch)使debug更方便

3. config
  * 首先生成.config，有一些配置需要参考，比如gcc版本之类的（-j表示使用的核心数）

    ```shell
    make olddefconfig -j16
    ```

  * 根据该[config](https://github.com/mufiye/mufiye_blog/blob/master/kernel_environment/config)修改.config配置

  * 修改.config文件中如下几个CONFIG选项

    ```config
    CONFIG_DEBUG_SECTION_MISMATCH=y  # 防止内联
    CONFIG_DEBUG_INFO=y
    CONFIG_DEBUG_KERNEL=y
    CONFIG_FRAME_POINTER=y  # Makefile中选择GCC编译选项
    CONFIG_GDB_SCRIPTS=y  # gdb python
    ```
4. 进行编译，先编译内核镜像

   ```shell
   make bzImage -j16
   ```

5. 编译模块

   ```shell
   make modules -j16
   ```

6. 进行模块安装

   ```shell
   # INSTALL_MOD_PATH表示模块安装的位置
   make modules_install INSTALL_MOD_PATH=mod -j16
   ```
