---
title: nfs源代码解析4——close
date: 2022-06-15 18:51:44
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,close]
---

# 1. Client

```c
close
  task_work_run
    ____fput
      __fput
        nfs_file_release  /* 释放一个nfs文件 */
          nfs_file_clear_open_context  /* 清空打开的context */ 
            put_nfs_open_context_sync
              __put_nfs_open_context
                nfs4_close_context
                  nfs4_close_sync
                    __nfs4_close
                      nfs4_do_close  /* 关闭文件的请求 */
                        rpc_run_task
                          rpc_execute
```

* ***Q：为什么nfs_fscache_release_file函数是空的？***

* A：

* ***Q：关闭文件的过程中发送了什么请求？***
* A：

# 2. Server

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound
            nfsd4_close
            nfsd4_encode_operation
              nfsd4_encode_close
```

* ***Q：服务器做了哪些处理？***
* A：
