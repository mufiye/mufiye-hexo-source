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
          nfs_file_clear_open_context
            invalidate_inode_pages2   /* 清空该文件的inode对应的页缓存 */
            put_nfs_open_context_sync /* __put_nfs_open_context的封装 */
              __put_nfs_open_context  /* 将open_context从lru链表中删除，调用nfs4_close_context，并清除nfs和rpc认证信息（cred）*/
                nfs4_close_context    /* 两种调用方式，同步或者异步(__nfs4_close的wait参数) */
                  nfs4_close_sync    /* 同步的方式关闭 */
                    __nfs4_close     /* 根据访问模式（只读，读写等）判断是否需要发送close请求，同时对nfs4_state（对应的是该文件的打开状态）做相应的修改 */
                      nfs4_do_close  /* 发送close请求 */
                        rpc_run_task
                          rpc_execute
          nfs_fscache_release_file   /* 函数体为空？*/
```

## 重点结构体

### 1）nfs_open_context

```c
struct nfs_open_context {
	struct nfs_lock_context lock_context;
	fl_owner_t flock_owner;         /* 文件锁 */
	struct dentry *dentry;          /* 对应目录项 */
	const struct cred *cred;        /* 认证信息 */
	struct rpc_cred __rcu *ll_cred;	/* low-level cred - use to check for expiry */
	struct nfs4_state *state;       /* 对应文件的打开状态 */
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

## __nfs4_close

四个参数，state是open操作创建的nfs4_state结构，fmode表示关闭的访问模式，wait表示是同步还是异步操作。nfs4_state结构中的n_rdonly、n_wronly、n_rdwr分别表示这个用户以只读权限、只写权限、读写权限打开文件的次数，每执行一次close操作都需要减少相应权限的计数，通常情况下，当n_rdonly和n_rdwr都减到0时就可以向服务器发起请求释放读权限，当n_wronly和n_rdwr都减到0时就可以向服务器发起请求释放写权限。但是涉及到delegation就会有一些特殊的情况，当一个客户端的两个用户打开同一个文件，第一个用户如果被分配了delegation，第二个用户就可以不使用open请求就直接打开该文件，而在关闭时自然也不需要发送close请求。

如果不需要发送close请求，则只需要调用nfs4_put_open_state函数清除相应的nfs4_state数据和调用nfs4_put_state_owner函数将对应的state_owner从LRU链表中释放。如果需要发送close请求，则调用nfs4_do_close发送close请求报文。

# 2. Server

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound
            nfsd4_close
              nfs4_preprocess_seqid_op
              nfsd4_bump_seqid
              nfs4_inc_and_copy_stateid
              nfsd4_close_open_stateid
              nfs4_put_stid
            nfsd4_encode_operation
              nfsd4_encode_close
```

## 重点结构体

### 1）nfsd4_close

```c
struct nfsd4_close {
	u32		cl_seqid;           /* request */
	stateid_t	cl_stateid;         /* request+response */
};
```

### 2）nfs4_ol_stateid

nfs4_ol_stateid对应于客户端的nfs4_state

```c
struct nfs4_ol_stateid {
	struct nfs4_stid		st_stid;
	struct list_head		st_perfile;
	struct list_head		st_perstateowner;
	struct list_head		st_locks;
	struct nfs4_stateowner		*st_stateowner;
	struct nfs4_clnt_odstate	*st_clnt_odstate;
/*
 * These bitmasks use 3 separate bits for READ, ALLOW, and BOTH;
 * 这些掩码只使用了三个位，分别是只读、只写和读写
 */
	unsigned char			st_access_bmap;
	unsigned char			st_deny_bmap;
	struct nfs4_ol_stateid		*st_openstp;
	struct mutex			st_mutex;
};
```

## nfsd4_close

