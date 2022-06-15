---
title: nfs源代码解析4——close
date: 2022-06-15 18:51:44
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,close]
---

# 1. Client

## 1.1 cat file

```c
close
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

## 1.2 

==写文件时应该也一样吧?==

# 2. Server

## 1.1 cat file

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound
     		nfsd4_close
```

