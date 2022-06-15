---
title: nfs源代码解析2——write
date: 2022-06-15 15:50:46
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,write]
---

# 1. Client

==Q：为什么也会调用nfs_file_write，但是如果断点在rpc_run_task函数栈中没有nfs_file_write？==

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
write?
  ksys_dup3
    do_dup2
      filp_close
        nfs4_file_flush
          nfs_wb_all
            filemap_write_and_wait
              filemap_write_and_wait_range
                __filemap_fdatawrite_range
                  filemap_fdatawrite_wbc
                    do_writepages
                      nfs_writepages
                        nfs_pageio_complete
                          nfs_pageio_complete_mirror
                            nfs_pageio_doio
                              nfs_generic_pg_pgios
                                nfs_initiate_pgio
                                  rpc_run_task
    								rpc_execute
```

# 2. Server

==仔细研究nfsd4_write==

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound
            nfsd4_write
```

