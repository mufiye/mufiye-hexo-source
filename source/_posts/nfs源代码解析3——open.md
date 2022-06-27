---
title: nfs源代码解析3——open
date: 2022-06-15 18:51:37
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,open]
---

# 1. Client

## 1.1 cat file

### 1）第一次cat

[gdb log file](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_open_gdb_log.txt)

```c
open
  do_sys_open
    do_sys_openat2
      do_filp_open
        path_openat
          link_path_walk
            may_lookup
              inode_permission
                do_inode_permission  /* 以上都是vfs的操作 */
                  nfs_permission  /* 用于检测访问权限 */
                    nfs_do_access
                      nfs_access_get_cached
                        nfs_access_get_cached_locked
                          __nfs_revalidate_inode
                            nfs4_proc_getattr
                              _nfs4_proc_getattr /* 获取文件属性 */
                                nfs4_do_call_sync
                                  nfs4_call_sync_custom
                                    rpc_run_task
                                      rpc_execute
                      nfs4_proc_access /* 其作用是什么？*/
                        _nfs4_proc_access
                          nfs4_call_sync
                            nfs4_call_sync_sequence
                              nfs4_do_call_sync
                                nfs4_call_sync_custom
                                  rpc_run_task
                                    rpc_execute
          open_last_lookups
            lookup_open
              atomic_open  /* 上面都是vfs的操作 */
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

**主要分析open operation。**



### 2）第二次cat

```c
open
  do_sys_open
    do_sys_openat2
      do_filp_open
        path_openat
          do_open
            vfs_open
              do_dentry_open  /* 上面都是vfs的操作 */
                nfs4_file_open  /* nfs打开文件 */
                  nfs4_atomic_open /* 原子性地打开一个文件 */
    				nfs4_label_init_security
                    nfs4_do_open
                      _nfs4_do_open
                        _nfs4_open_and_get_state
                          _nfs4_proc_open
                            nfs4_run_open_task
                              rpc_run_task
                                rpc_execute
```

* ***Q：什么是原子性地打开文件？***

* A：
* ***Q：_nfs4_open_and_get_state获取的是什么状态？***
* A：
* ***Q：打开文件的过程中发送了什么请求？***
* A：

# 2. Server

## 2.1 cat file

### 1）第一次cat

https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_server_cat_first_gdb_log.txt

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound
            nfsd4_open  /* 服务器端打开，重点研究 */
    		nfsd4_encode_operation
              nfsd4_encode_open  /* 将open operation的返回信息编码，重点研究 */
```
* ***Q：nfsd4_open和nfsd4_encode_open做了什么？***

* A：

### 2）第二次cat

https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_server_cat_second_gdb_log.txt

* ***Q：服务器端没有调用nfsd4_open函数？***

* A
