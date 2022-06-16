---
title: nfs源代码解析5——mount
date: 2022-06-15 21:57:01
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,mount]
---

# Mount

## 1. Client

[gdb log file](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_mount_gdb_log1.txt)

```c
mount
  __do_sys_mount
    do_mount
      path_mount
        do_new_mount
          vfs_get_tree
            nfs_get_tree
              nfs4_try_get_tree
                nfs4_create_server
                  nfs4_init_server
                    nfs4_set_client
                      nfs_get_client
                        nfs4_alloc_client
                          nfs_create_rpc_client
                            rpc_create
                              rpc_create_xprt
                                rpc_ping
                                  rpc_call_null_helper
                                    rpc_run_task
                                      rpc_execute
                        nfs4_init_client
                          nfs4_discover_server_trunking
                            nfs41_discover_server_trunking
                              nfs4_proc_exchange_id
                                _nfs4_proc_exchange_id
                                  nfs4_run_exchange_id
                                    rpc_run_task
                                      rpc_execute
kthread
  nfs4_run_state_manager
    nfs4_state_manager
      nfs4_reclaim_lease
        nfs4_establish_lease
          nfs41_init_clientid
            nfs4_proc_exchange_id
              _nfs4_proc_exchange_id
                nfs4_run_exchange_id
                  rpc_run_task
                    rpc_execute
            nfs4_proc_create_session
              _nfs4_proc_create_session
                rpc_call_sync
                  rpc_run_task
                    rpc_execute
      nfs4_state_end_reclaim_reboot
        nfs4_reclaim_complete
          nfs41_proc_reclaim_complete
            nfs4_call_sync_custom
              rpc_run_task
                rpc_execute

/* 就是上面那个mount的执行，不过因为中间多了个kthread的执行流程 */    
mount
  __do_sys_mount
    do_mount
      path_mount
        do_new_mount
          vfs_get_tree
            nfs_get_tree
              nfs4_try_get_tree
                nfs4_create_server
                  nfs4_server_common_setup
                    nfs4_get_rootfh
                      nfs4_proc_get_rootfh
                        nfs41_find_root_sec
                          nfs41_proc_secinfo_no_name
                            _nfs41_proc_secinfo_no_name
                              nfs4_call_sync_custom
                                rpc_run_task
                                  rpc_execute
                          nfs4_lookup_root_sec
                            nfs4_lookup_root
                              _nfs4_lookup_root
                                nfs4_call_sync
                                  nfs4_call_sync_sequence
                                    nfs4_do_call_sync
                                      nfs4_call_sync_custom
                                        rpc_run_task
                                          rpc_execute
                        nfs4_server_capabilities
                          _nfs4_server_capabilities
                            nfs4_call_sync
                              nfs4_call_sync_sequence
                                nfs4_do_call_sync
                                  nfs4_call_sync_custom
                                    rpc_run_task
                                      rpc_execute
                        nfs4_do_fsinfo
                          _nfs4_do_fsinfo
                            nfs4_call_sync
                              nfs4_call_sync_sequence
                                nfs4_do_call_sync
                                  nfs4_call_sync_custom
                                    rpc_run_task
                                      rpc_execute
                    nfs_probe_server
                      nfs_probe_fsinfo
                        nfs4_server_capabilities
                          _nfs4_server_capabilities
                            nfs4_call_sync
                              nfs4_call_sync_sequence
                                nfs4_do_call_sync
                                  nfs4_call_sync_custom
                                    rpc_run_task
                                      rpc_execute
                        nfs4_proc_fsinfo
                          nfs4_do_fsinfo
                            _nfs4_do_fsinfo
                              nfs4_call_sync
                                nfs4_call_sync_sequence
                                  nfs4_do_call_sync
                                    nfs4_call_sync_custom
                                      rpc_run_task
                                        rpc_execute
                        nfs4_proc_pathconf
                          _nfs4_proc_pathconf
                            nfs4_call_sync
                              nfs4_call_sync_sequence
                                nfs4_do_call_sync
                                  nfs4_call_sync_custom
                                    rpc_run_task
                                      rpc_execute
              do_nfs4_mount
                fc_mount
                  vfs_get_tree
                    nfs_get_tree
                      nfs_get_tree_common
                        nfs_get_root
                          nfs4_proc_get_root
                            nfs4_server_capabilities
                              _nfs4_server_capabilities
                                nfs4_call_sync
                                  nfs4_call_sync_sequence
                                    nfs4_do_call_sync
                                      nfs4_call_sync_custom
                                        rpc_run_task
                                          rpc_execute
                            nfs4_proc_getattr
                              _nfs4_proc_getattr
                                nfs4_do_call_sync
                                  nfs4_call_sync_custom
                                    rpc_run_task
                                      rpc_execute
                mount_subtree
                  vfs_path_lookup
                    filename_lookup
                      path_lookupat
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
                      lookup_last
                        walk_component
                          lookup_slow
                            __lookup_slow
                              nfs_lookup
                                nfs4_proc_lookup
                                  nfs4_proc_lookup_common
                                    _nfs4_proc_lookup
                                      nfs4_do_call_sync
                                        nfs4_call_sync_custom
                                          rpc_run_task
                                            rpc_execute
                          step_into
                            handle_mounts
                              traverse_mounts
                                __traverse_mounts
                                  follow_automount
                                    nfs_d_automount
                                      nfs4_submount
                                        nfs4_proc_lookup_mountpoint
                                          nfs4_proc_lookup_common
                                            _nfs4_proc_lookup
                                              nfs4_do_call_sync
                                                nfs4_call_sync_custom
                                                  rpc_run_task
                                                    rpc_execute
                                        nfs_do_submount
                                          nfs_clone_server
                                            nfs_probe_server
                                              nfs_probe_fsinfo
                                                nfs4_server_capabilities
                                                  _nfs4_server_capabilities
                                                    nfs4_call_sync
                                                      nfs4_call_sync_sequence
                                                        nfs4_do_call_sync
                                                          nfs4_call_sync_custom
                                                            rpc_run_task
                                                              rpc_execute
                                                nfs4_proc_fsinfo
                                                  nfs4_do_fsinfo
                                                    _nfs4_do_fsinfo
                                                      nfs4_call_sync
                                                        nfs4_call_sync_sequence
                                                          nfs4_do_call_sync
                                                            nfs4_call_sync_custom
                                                              rpc_run_task
                                                                rpc_execute
                                                nfs4_proc_pathconf
                                                  _nfs4_proc_pathconf
                                                    nfs4_call_sync
                                                      nfs4_call_sync_sequence
                                                        nfs4_do_call_sync
                                                          nfs4_call_sync_custom
                                                            rpc_run_task
                                                              rpc_execute
                                          vfs_get_tree
                                            nfs_get_tree
                                              nfs_get_tree_common
                                                nfs_get_root
                                                  nfs4_proc_get_root
                                                    nfs4_server_capabilities
                                                      _nfs4_server_capabilities
                                                        nfs4_call_sync
                                                          nfs4_call_sync_sequence
                                                            nfs4_do_call_sync
                                                              nfs4_call_sync_custom
                                                                rpc_run_task
                                                                  rpc_execute
                                                    nfs4_proc_getattr
                                                      _nfs4_proc_getattr
                                                        nfs4_do_call_sync
                                                          nfs4_call_sync_custom
                                                            rpc_run_task
                                                              rpc_execute
```

## 2. Server

**服务器端执行了哪些函数？如何使用gdb追踪？**

```c

```

# Unmount

## 1. Client

[gdb log file](https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_unmount_gdb_log1.txt)

```c
task_work_run
  __cleanup_mnt
    cleanup_mnt
      deactivate_super
        deactivate_locked_super
          nfs_kill_super
            nfs_free_server
              nfs_put_client
                nfs4_free_client
                  nfs4_shutdown_client
                    nfs41_shutdown_client
                      nfs4_destroy_session
                        nfs4_proc_destroy_session
                          rpc_call_sync
                            rpc_run_task
                              rpc_execute
                      nfs4_destroy_clientid
                        nfs4_proc_destroy_clientid
                          _nfs4_proc_destroy_clientid
                            rpc_call_sync
                              rpc_run_task
                                rpc_execute
```

## 2. Server

**服务器端执行了哪些函数？如何使用gdb追踪？**

```c

```

