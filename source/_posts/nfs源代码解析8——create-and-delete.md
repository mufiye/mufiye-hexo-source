---
title: nfs源代码解析8——create and delete
date: 2022-06-26 20:27:14
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,create,delete]
---

# 1. Client

## 1.1 create file（touch）

[gdb log - touch](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_touch-create_file_gdb_log.txt)

```c
__se_sys_newstat
  vfs_stat
    vfs_fstatat
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
__do_sys_openat
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
__do_sys_utimensat
  do_utimes
    do_utimes_fd
      vfs_utimes
        notify_change
          nfs_setattr
            nfs4_proc_setattr
              nfs4_do_setattr
                _nfs4_do_setattr
                  nfs4_call_sync
                    nfs4_call_sync_sequence
                      nfs4_do_call_sync
                        nfs4_call_sync_custom
                          rpc_run_task
                            rpc_execute
task_work_run
  ____fput
    __fput
      nfs_file_release
        nfs_file_clear_open_context
          put_nfs_open_context_sync
            __put_nfs_open_context
              nfs4_close_context
                nfs4_close_sync
                  __nfs4_close
                    nfs4_do_close
                      rpc_run_task
                        rpc_execute
```

## 1.2 delete file

[gdb log - delete](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_rmFile_gdb_log.txt)

```c
__se_sys_newfstatat
  __do_sys_newfstatat
    vfs_fstatat
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
__x64_sys_unlinkat
  __do_sys_unlinkat
    do_unlinkat
      vfs_unlink
        nfs_unlink
          nfs_safe_remove
            nfs4_proc_remove
              _nfs4_proc_remove
                nfs4_call_sync
                  nfs4_call_sync_sequence
                    nfs4_do_call_sync
                      nfs4_call_sync_custom
                        rpc_run_task
                          rpc_execute
```

## 1.3 create dir

[gdb log - mkdir](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_mkdir_gdb_log.txt)

```c
__do_sys_mkdir
  do_mkdirat
    vfs_mkdir
      nfs_mkdir
        nfs4_proc_mkdir
          _nfs4_proc_mkdir
            nfs4_do_create
              nfs4_call_sync
                nfs4_call_sync_sequence
                  nfs4_do_call_sync
                    nfs4_call_sync_custom
                      rpc_run_task
                        rpc_execute
```
## 1.4 delete dir（rm -rf with tab）

[gdb log - rm dir](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_rmrfDir-withTab_gdb_log.txt)

```c
__do_sys_openat
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
            do_open
              complete_walk
                nfs_weak_revalidate
                  nfs_lookup_verify_inode
                    __nfs_revalidate_inode
                      nfs4_proc_getattr
                        _nfs4_proc_getattr
                          nfs4_do_call_sync
                            nfs4_call_sync_custom
                              rpc_run_task
                                rpc_execute
__do_sys_getdents64
  iterate_dir
    nfs_readdir
      readdir_search_pagecache
        find_and_lock_cache_page
          nfs_readdir_xdr_to_array
            nfs_readdir_xdr_filler
              nfs4_proc_readdir
                _nfs4_proc_readdir
                  nfs4_call_sync
                    nfs4_call_sync_sequence
                      nfs4_do_call_sync
                        nfs4_call_sync_custom
                          rpc_run_task
                            rpc_execute
__se_sys_newstat
  vfs_stat
    vfs_fstatat
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

...

__do_sys_unlinkat
  do_rmdir
    vfs_rmdir
      nfs_rmdir
        nfs4_proc_rmdir
          _nfs4_proc_remove
            nfs4_call_sync
              nfs4_call_sync_sequence
                nfs4_do_call_sync
                  nfs4_call_sync_custom
                    rpc_run_task
                      rpc_execute
```
# 2. Server

