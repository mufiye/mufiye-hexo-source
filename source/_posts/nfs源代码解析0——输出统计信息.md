---
title: nfs源代码解析0——输出统计信息
date: 2022-06-14 22:33:53
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,procfs,统计信息]
---

# 1. Client

## 1.1 读取/proc/net/rpc/nfs

```c
read
  ksys_read
    vfs_read
      proc_reg_read
        pde_read
          seq_read
            seq_read_iter
              rpc_proc_show
```
## 1.2 客户端方法的枚举：

```c
/* in include/linux/nfs4.h */
enum {
	NFSPROC4_CLNT_NULL = 0,		/* Unused */
	NFSPROC4_CLNT_READ,
	NFSPROC4_CLNT_WRITE,
	NFSPROC4_CLNT_COMMIT,
	NFSPROC4_CLNT_OPEN,
	NFSPROC4_CLNT_OPEN_CONFIRM,
	NFSPROC4_CLNT_OPEN_NOATTR,
	NFSPROC4_CLNT_OPEN_DOWNGRADE,
	NFSPROC4_CLNT_CLOSE,
	NFSPROC4_CLNT_SETATTR,
	NFSPROC4_CLNT_FSINFO,
	NFSPROC4_CLNT_RENEW,
	NFSPROC4_CLNT_SETCLIENTID,
	NFSPROC4_CLNT_SETCLIENTID_CONFIRM,
	NFSPROC4_CLNT_LOCK,
	NFSPROC4_CLNT_LOCKT,
	NFSPROC4_CLNT_LOCKU,
	NFSPROC4_CLNT_ACCESS,
	NFSPROC4_CLNT_GETATTR,
	NFSPROC4_CLNT_LOOKUP,
	NFSPROC4_CLNT_LOOKUP_ROOT,
	NFSPROC4_CLNT_REMOVE,
	NFSPROC4_CLNT_RENAME,
	NFSPROC4_CLNT_LINK,
	NFSPROC4_CLNT_SYMLINK,
	NFSPROC4_CLNT_CREATE,
	NFSPROC4_CLNT_PATHCONF,
	NFSPROC4_CLNT_STATFS,
	NFSPROC4_CLNT_READLINK,
	NFSPROC4_CLNT_READDIR,
	NFSPROC4_CLNT_SERVER_CAPS,
	NFSPROC4_CLNT_DELEGRETURN,
	NFSPROC4_CLNT_GETACL,
	NFSPROC4_CLNT_SETACL,
	NFSPROC4_CLNT_FS_LOCATIONS,
	NFSPROC4_CLNT_RELEASE_LOCKOWNER,
	NFSPROC4_CLNT_SECINFO,
	NFSPROC4_CLNT_FSID_PRESENT,

	NFSPROC4_CLNT_EXCHANGE_ID,
	NFSPROC4_CLNT_CREATE_SESSION,
	NFSPROC4_CLNT_DESTROY_SESSION,
	NFSPROC4_CLNT_SEQUENCE,
	NFSPROC4_CLNT_GET_LEASE_TIME,
	NFSPROC4_CLNT_RECLAIM_COMPLETE,
	NFSPROC4_CLNT_LAYOUTGET,
	NFSPROC4_CLNT_GETDEVICEINFO,
	NFSPROC4_CLNT_LAYOUTCOMMIT,
	NFSPROC4_CLNT_LAYOUTRETURN,
	NFSPROC4_CLNT_SECINFO_NO_NAME,
	NFSPROC4_CLNT_TEST_STATEID,
	NFSPROC4_CLNT_FREE_STATEID,
	NFSPROC4_CLNT_GETDEVICELIST,
	NFSPROC4_CLNT_BIND_CONN_TO_SESSION,
	NFSPROC4_CLNT_DESTROY_CLIENTID,

	NFSPROC4_CLNT_SEEK,
	NFSPROC4_CLNT_ALLOCATE,
	NFSPROC4_CLNT_DEALLOCATE,
	NFSPROC4_CLNT_LAYOUTSTATS,
	NFSPROC4_CLNT_CLONE,
	NFSPROC4_CLNT_COPY,
	NFSPROC4_CLNT_OFFLOAD_CANCEL,

	NFSPROC4_CLNT_LOOKUPP,
	NFSPROC4_CLNT_LAYOUTERROR,
	NFSPROC4_CLNT_COPY_NOTIFY,

	NFSPROC4_CLNT_GETXATTR,
	NFSPROC4_CLNT_SETXATTR,
	NFSPROC4_CLNT_LISTXATTRS,
	NFSPROC4_CLNT_REMOVEXATTR,
	NFSPROC4_CLNT_READ_PLUS,
};
```

## 1.3 客户端改变方法计数

**待仔细研究call_start函数！！！**

```c
__rpc_execute
  call_start
    struct rpc_clnt	*clnt = task->tk_client;
	int idx = task->tk_msg.rpc_proc->p_statidx;
	...
    clnt->cl_program->version[clnt->cl_vers]->counts[idx]++;
```

```c
struct rpc_task {
	...
	struct rpc_message	tk_msg;		/* RPC call info */
    ...
}

struct rpc_message {
	const struct rpc_procinfo *rpc_proc;	/* Procedure information */
	...
};

struct rpc_procinfo {
	...
	u32			p_statidx;	/* Which procedure to account */
	...
};
```

有对应的函数会在运行的过程中对对应属性赋值（以read举例）：

```c
const struct rpc_procinfo nfs4_procedures[] = {
	PROC(READ,		enc_read,		dec_read),
    ...
}
```

```c
read
  ksys_read
    vfs_read
      new_sync_read
        call_read_iter /* file->f_op->read_iter */
          nfs_file_read /* 读取nfs file */
            generic_file_read_iter
              filemap_read /* Read data from the page cache. */
                filemap_get_pages
                  page_cache_sync_readahead /* generic file readahead */
                    page_cache_sync_ra
                      ondemand_readahead
                        page_cache_ra_order
                          do_page_cache_ra
                            page_cache_ra_unbounded
                              read_pages
                                nfs_readahead  /* 页高速缓存读 */
    							readahead_page /* Get the next page to read. */
    							readpage_async_filler /* 填充读取page的request */
                                  nfs_pageio_complete_read 
                                    nfs_pageio_complete
                                      nfs_pageio_complete_mirror
                                        nfs_pageio_doio
                                          nfs_generic_pg_pgios 
                                            nfs_initiate_pgio
                                              rpc_run_task
                                                rpc_execute
```

***补充：操作计数统计***

**read：**

```c
nfs_initiate_pgio  /* hdr->rw_ops->rw_initiate(...); */
  nfs_initiate_read  /* rpc_ops->read_setup(hdr, msg); */
    nfs4_proc_read_setup   /* msg->rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_READ]; */
```

**write：**

```c
nfs_initiate_pgio  /* hdr->rw_ops->rw_initiate(...); */
  nfs_initiate_write  /* rpc_ops->write_setup(hdr, msg, &task_setup_data->rpc_client); */
    nfs4_proc_write_setup  /* msg->rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_WRITE]; */
```

**open：**

```c
nfs4_run_open_task  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_OPEN], */
```

**Access：**

```c
_nfs4_proc_access  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_ACCESS], */
```

**GetAttr：**

```c
_nfs4_proc_getattr  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_GETATTR], */
```

**Close：**

```c
nfs4_do_close  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_CLOSE], */
```

**Sequence：**（?）

```c
_nfs41_proc_sequence  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_SEQUENCE], */
```

**Delegation：**（?）

```c
_nfs4_proc_delegreturn  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_DELEGRETURN], */
```

**Create Session：**

```c
_nfs4_proc_create_session /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_CREATE_SESSION] */
```

**DESTROY_SESSION：**

```c
nfs4_proc_destroy_session /* ... = &nfs4_procedures[NFSPROC4_CLNT_DESTROY_SESSION], */
```

**EXCHANGE_ID：**（?）

```c
nfs4_run_exchange_id  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_EXCHANGE_ID], */
```

**DESTROY_CLIENTID：**（?）

```c
_nfs4_proc_destroy_clientid /* ... = &nfs4_procedures[NFSPROC4_CLNT_DESTROY_CLIENTID], */
```

**Reclaim Complete：**（?）

```c
nfs41_proc_reclaim_complete /* ... = nfs4_procedures[NFSPROC4_CLNT_RECLAIM_COMPLETE], */
```

**SECINFO_NO_NAME：**（?）

```c
_nfs41_proc_secinfo_no_name  /* ... = &nfs4_procedures[NFSPROC4_CLNT_SECINFO_NO_NAME] */
```

**LOOKUP_ROOT：**（?）

```c
_nfs4_lookup_root  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_LOOKUP_ROOT], */
```

**SERVER_CAPS：**（?）

```c
_nfs4_server_capabilities  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_SERVER_CAPS], */
```

**FSINFO：**（?）

```c
_nfs4_do_fsinfo  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_FSINFO], */
```

**PATHCONF：**（?）

```c
_nfs4_proc_pathconf  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_PATHCONF], */
```

**LOOKUP：**（?）

```c
_nfs4_proc_lookup  /* .rpc_proc = &nfs4_procedures[NFSPROC4_CLNT_LOOKUP], */
```

# 2. Server

## 2.1 读取/proc/net/rpc/nfsd

```c
read
  ksys_read
    vfs_read
      proc_reg_read
        pde_read
          seq_read
            seq_read_iter
              nfsd_proc_show
```
## 2.2 服务器端方法的枚举：

```c
/* in include/linux/nfs4.h */
enum nfs_opnum4 {
	OP_ACCESS = 3,
	OP_CLOSE = 4,
	OP_COMMIT = 5,
	OP_CREATE = 6,
	OP_DELEGPURGE = 7,
	OP_DELEGRETURN = 8,
	OP_GETATTR = 9,
	OP_GETFH = 10,
	OP_LINK = 11,
	OP_LOCK = 12,
	OP_LOCKT = 13,
	OP_LOCKU = 14,
	OP_LOOKUP = 15,
	OP_LOOKUPP = 16,
	OP_NVERIFY = 17,
	OP_OPEN = 18,
	OP_OPENATTR = 19,
	OP_OPEN_CONFIRM = 20,
	OP_OPEN_DOWNGRADE = 21,
	OP_PUTFH = 22,
	OP_PUTPUBFH = 23,
	OP_PUTROOTFH = 24,
	OP_READ = 25,
	OP_READDIR = 26,
	OP_READLINK = 27,
	OP_REMOVE = 28,
	OP_RENAME = 29,
	OP_RENEW = 30,
	OP_RESTOREFH = 31,
	OP_SAVEFH = 32,
	OP_SECINFO = 33,
	OP_SETATTR = 34,
	OP_SETCLIENTID = 35,
	OP_SETCLIENTID_CONFIRM = 36,
	OP_VERIFY = 37,
	OP_WRITE = 38,
	OP_RELEASE_LOCKOWNER = 39,

	/* nfs41 */
	OP_BACKCHANNEL_CTL = 40,
	OP_BIND_CONN_TO_SESSION = 41,
	OP_EXCHANGE_ID = 42,
	OP_CREATE_SESSION = 43,
	OP_DESTROY_SESSION = 44,
	OP_FREE_STATEID = 45,
	OP_GET_DIR_DELEGATION = 46,
	OP_GETDEVICEINFO = 47,
	OP_GETDEVICELIST = 48,
	OP_LAYOUTCOMMIT = 49,
	OP_LAYOUTGET = 50,
	OP_LAYOUTRETURN = 51,
	OP_SECINFO_NO_NAME = 52,
	OP_SEQUENCE = 53,
	OP_SET_SSV = 54,
	OP_TEST_STATEID = 55,
	OP_WANT_DELEGATION = 56,
	OP_DESTROY_CLIENTID = 57,
	OP_RECLAIM_COMPLETE = 58,

	/* nfs42 */
	OP_ALLOCATE = 59,
	OP_COPY = 60,
	OP_COPY_NOTIFY = 61,
	OP_DEALLOCATE = 62,
	OP_IO_ADVISE = 63,
	OP_LAYOUTERROR = 64,
	OP_LAYOUTSTATS = 65,
	OP_OFFLOAD_CANCEL = 66,
	OP_OFFLOAD_STATUS = 67,
	OP_READ_PLUS = 68,
	OP_SEEK = 69,
	OP_WRITE_SAME = 70,
	OP_CLONE = 71,

	/* xattr support (RFC8726) */
	OP_GETXATTR                = 72,
	OP_SETXATTR                = 73,
	OP_LISTXATTRS              = 74,
	OP_REMOVEXATTR             = 75,

	OP_ILLEGAL = 10044,
};
```

## 2.3 服务器端改变方法计数

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound  /* 处理复合请求（多种请求混合） */
            nfsd4_increment_op_stats
```

每一个循环处理一个操作

```c
while (!status && resp->opcnt < args->opcnt) {
	op = &args->ops[resp->opcnt++];
	...
}
```

