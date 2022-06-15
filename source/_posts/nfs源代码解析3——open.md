---
title: nfs源代码解析3——open
date: 2022-06-15 18:51:37
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,open]
---

# 1. Client

## 1.1 cat file

### 1）第一次cat

==Q：为什么我一次cat shell命令会执行三次open？==

```c
open
  do_sys_open
    do_sys_openat2
      do_filp_open
        path_openat
          link_path_walk
            may_lookup
              inode_permission
                do_inode_permission
                  nfs_permission
                    nfs_do_access
                      nfs_access_get_cached
                        nfs_access_get_cached_locked
                          __nfs_revalidate_inode
                            nfs4_proc_getattr
                              _nfs4_proc_getattr
                                nfs4_do_call_sync
                                  nfs4_call_sync_custom
                                    rpc_run_task
                                      rpc_execute
    
/* 第二次执行open operation */
open
  do_sys_open
    do_sys_openat2
      do_filp_open
        path_openat
          link_path_walk
            may_lookup
              inode_permission
                do_inode_permission
                  nfs_permission
                    nfs_do_access
                      nfs4_proc_access
                        _nfs4_proc_access
                          nfs4_call_sync
                            nfs4_call_sync_sequence
                              nfs4_do_call_sync
                                nfs4_call_sync_custom
                                  rpc_run_task
                                    rpc_execute
/* 第三次执行open operation */
open
  do_sys_open
    do_sys_openat2
      do_filp_open
        path_openat
          open_last_lookups
            lookup_open
              atomic_open
                nfs_atomic_open
                  nfs4_atomic_open
                    nfs4_do_open
                      _nfs4_do_open
                        _nfs4_open_and_get_state
                          _nfs4_proc_open
                            nfs4_run_open_task
                              rpc_run_task
                                rpc_execute
```

### 2）第二次cat

```c
open
  do_sys_open
    do_sys_openat2
      do_filp_open
        path_openat
          do_open
            vfs_open
              do_dentry_open
                nfs4_file_open
                  nfs4_atomic_open
                    nfs4_do_open
                      _nfs4_do_open
                        _nfs4_open_and_get_state
                          _nfs4_proc_open
                            nfs4_run_open_task
                              rpc_run_task
                                rpc_execute
```

# 2. Server

## 2.1 cat file

### 1）第一次cat

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound
            nfsd4_open
```
### 2）第二次cat
==Q：服务器端没有调用nfsd4_open函数？==
