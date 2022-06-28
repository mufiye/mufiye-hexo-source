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
            put_nfs_open_context_sync  /* __put_nfs_open_context的封装 */
              __put_nfs_open_context
                nfs4_close_context
                  nfs4_close_sync
                    __nfs4_close
                      nfs4_do_close  /* 关闭文件的请求 */
                        rpc_run_task
                          rpc_execute
          nfs_fscache_release_file   /* 函数体为空？*/
```

## 重点结构体

### 1）nfs_open_context

```c
struct nfs_open_context {
	struct nfs_lock_context lock_context;
	fl_owner_t flock_owner;
	struct dentry *dentry;
	const struct cred *cred;
	struct rpc_cred __rcu *ll_cred;	/* low-level cred - use to check for expiry */
	struct nfs4_state *state;
	fmode_t mode;

	unsigned long flags;
#define NFS_CONTEXT_RESEND_WRITES	(1)
#define NFS_CONTEXT_BAD			(2)
#define NFS_CONTEXT_UNLOCK	(3)
#define NFS_CONTEXT_FILE_OPEN		(4)
	int error;

	struct list_head list;
	struct nfs4_threshold	*mdsthreshold;
	struct rcu_head	rcu_head;
};
```

### 2）nfs4_closedata

```c
struct nfs4_closedata {
	struct inode *inode;          /* 对应的inode */
	struct nfs4_state *state;     /* 对应的state */
	struct nfs_closeargs arg;     /* close请求的参数 */
	struct nfs_closeres res;      /* close请求的结果 */
	struct {
		struct nfs4_layoutreturn_args arg;
		struct nfs4_layoutreturn_res res;
		struct nfs4_xdr_opaque_data ld_private;
		u32 roc_barrier;
		bool roc;
	} lr;
	struct nfs_fattr fattr;       /* nfs文件属性 */
	unsigned long timestamp;      /* 时间戳 */
};
```

## nfs_file_release



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
