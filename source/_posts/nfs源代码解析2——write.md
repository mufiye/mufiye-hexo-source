---
title: nfs源代码解析2——write
date: 2022-06-15 15:50:46
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,write]
---

# 1. Client

**Q：为什么也会调用nfs_file_write，但是如果断点在rpc_run_task函数栈中没有nfs_file_write？**

```c
write
  ksys_write
    vfs_write
      new_sync_write
        call_write_iter
          nfs_file_write
```

**正确的流程：**

```c
__do_sys_dup2
  ksys_dup3
    do_dup2
      filp_close
        nfs4_file_flush  /* Flush all dirty pages */
          nfs_wb_all     /* flush the inode to disk. */
            filemap_write_and_wait
              filemap_write_and_wait_range
                __filemap_fdatawrite_range
                  filemap_fdatawrite_wbc
                    do_writepages
                      nfs_writepages
                        nfs_pageio_init_write /* 初始化nfs_pageio_descriptor */
    					  nfs_initiate_pgio
                            nfs_pageio_init /* 赋值rw_ops为nfs_rw_write_ops */
    					write_cache_pages
    					  nfs_writepages_callback  /* write_cache_pages的回调函数，添加脏页到nfs_pageio_descriptor上 */
                        nfs_pageio_complete  /* 负责向服务器提交写请求 */
                          nfs_pageio_complete_mirror
                            nfs_pageio_doio
                              nfs_generic_pg_pgios
                                nfs_initiate_pgio
                                  rpc_run_task
                                    rpc_execute
```

## nfs_writepages

nfs_writepages函数中的nfs_inc_stats负责增加对应事件的引用计数，nfs_pageio_init_write负责初始化操作所需要的nfs_pageio_descriptor结构体，write_cache_pages作为vfs层的函数，其在缓存中查找脏页并且对脏页执行nfs_writepages_callback函数，该函数将脏页添加到nfs_pageio_descriptor的pg_list中，而nfs_pageio_complete负责向服务器提交写脏页请求。

# 2. Server

**仔细研究nfsd4_write**

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound
            nfsd4_write
```

