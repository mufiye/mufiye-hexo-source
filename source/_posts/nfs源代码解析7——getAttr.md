---
title: nfs源代码解析7——getAttr
date: 2022-06-26 20:26:31
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,getAttr]
---

# 1. Client

## 1.1 获取普通文件或者目录项的属性（file和dir一样，ls）

[gdb log for file](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_ls-l_file_gdb_log.txt)

[gdb log for dir](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_ls-ld_dir_gdb_log.txt)

```c
__do_sys_statx
  do_statx
    vfs_statx
      vfs_getattr
        vfs_getattr_nosec
          nfs_getattr
            __nfs_revalidate_inode
              nfs4_proc_getattr
                _nfs4_proc_getattr
                  nfs4_do_call_sync
                    nfs4_call_sync_custom
                      rpc_run_task
                        rpc_execute
```

## 1.2 获取文件系统属性（df -h）

[gdb log](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_df-h_gdb_log.txt)

```c
__do_sys_statfs
  user_statfs
    vfs_statfs
      statfs_by_dentry
        nfs_statfs
          nfs4_proc_statfs
            _nfs4_proc_statfs
              nfs4_call_sync
                nfs4_call_sync_sequence
                  nfs4_do_call_sync
                    nfs4_call_sync_custom
                      rpc_run_task
                        rpc_execute
```

# 2. Server

