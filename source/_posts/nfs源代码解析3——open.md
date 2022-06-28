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
                    nfs_do_access /* 用于获取（本地缓存或远端）并检查访问权限 */
                      nfs_access_get_cached  /* 检查缓存 */
                        nfs_access_get_cached_locked
                          __nfs_revalidate_inode /* 刷新nfs缓存的文件属性 */
                            nfs4_proc_getattr  /* 获取文件属性 */
                              _nfs4_proc_getattr 
                                nfs4_do_call_sync /* 发起同步nfs请求 */
                                  nfs4_call_sync_custom
                                    rpc_run_task
                                      rpc_execute
                      nfs4_proc_access /* 循环体执行_nfs4_proc_access */
                        _nfs4_proc_access /* 首先检查delegation,无效则发起access请求并更新本地缓存 */
                          nfs4_call_sync  /* 用于发起rpc请求 */
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

#### nfs_do_access（access operation）

nfs_do_access函数首先使用nfs_access_get_cached函数在文件索引的红黑树中查找对应文件的访问权限，如果找到了就直接检查权限并返回；如果没有找到就向服务器发起access请求并将请求到的权限信息添加到红黑树中，之后检查权限并返回。

```c
/* NFSv3/v4 Access mode cache entry */
struct nfs_access_entry {
	struct rb_node		rb_node; /* 对应的红黑数节点 */
	struct list_head	lru;     /* 对应的LRU链表 */
	kuid_t			fsuid;       /* 拥有者id号 */
	kgid_t			fsgid;       /* 拥有者组号 */
	struct group_info	*group_info;  /* 用户组信息 */
	__u32			mask;       /* 用户的访问权限 */
	struct rcu_head		rcu_head;
};
```

#### open相关的结构体

nfs4_opendata是nfs用于open请求的数据，nfs4_state对应的是某个文件的状态，nfs4_state_owner对应的是多个状态（nfs4_state）的持有者（可以理解为一个客户端用户）。

##### 1）nfs4_opendata

```c
struct nfs4_opendata {
	struct kref kref;  /* 该opendata的引用计数 */
	struct nfs_openargs o_arg;  /* open请求的参数 */
	struct nfs_openres o_res;   /* open请求的结果 */
	struct nfs_open_confirmargs c_arg;  /* open confirm请求的参数 */
	struct nfs_open_confirmres c_res;   /* open confirm请求的结果 */
	struct nfs4_string owner_name;      /* open的文件的所有者 */
	struct nfs4_string group_name;      /* open的文件的所在组 */
	struct nfs4_label *a_label;         /* ？ */
	struct nfs_fattr f_attr;            /* nfs文件属性 */
	struct dentry *dir;                 /* open的文件所在的目录 */
	struct dentry *dentry;              /* open的文件的目录项 */
	struct nfs4_state_owner *owner;     /* 表示打开状态的所有者 */
	struct nfs4_state *state;           /* 表示该open请求的状态 */
    
	...
};
```

##### 2）nfs4_state

```c
struct nfs4_state {
	struct list_head open_states;	/* List of states for the same state_owner */
	struct list_head inode_states;	/* List of states for the same inode */
	struct list_head lock_states;	/* List of subservient lock stateids */

	struct nfs4_state_owner *owner;	/* Pointer to the open owner */
	struct inode *inode;		/* Pointer to the inode */
    
    nfs4_stateid stateid;		/* Current stateid: may be delegation */
	nfs4_stateid open_stateid;	/* OPEN stateid */
    
    unsigned int n_rdonly;		/* Number of read-only references */
	unsigned int n_wronly;		/* Number of write-only references */
	unsigned int n_rdwr;		/* Number of read/write references */
	fmode_t state;			/* State on the server (R,W, or RW) */
	refcount_t count;

	wait_queue_head_t waitq;
	struct rcu_head rcu_head;
}
```

##### 3）nfs4_state_owner

```c
struct nfs4_state_owner {
	struct nfs_server    *so_server;   /* 所属的nfs_server */
	struct list_head     so_lru;	   /* 对应于nfs_server中的state_owners_lru链表元素，该链表用于保存空闲的nfs4_state_owner */
	unsigned long        so_expires;   /* 挂载到上述链表中的时间，也就是过期时间 */
	struct rb_node	     so_server_node;  /* nfs_server的所有nfs4_state_owner被加到红黑树中，这个对应的是一个节点 */

	const struct cred    *so_cred;	 /* Associated cred，用户信息 */

	spinlock_t	     so_lock;        
	atomic_t	     so_count;
	unsigned long	     so_flags;
	struct list_head     so_states;
	struct nfs_seqid_counter so_seqid;
	seqcount_spinlock_t  so_reclaim_seqcount;
	struct mutex	     so_delegreturn_mutex;
};
```

#### nfs_atomic_open（open operation）



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
                nfs4_file_open
                  nfs4_atomic_open 
    				nfs4_label_init_security
                    nfs4_do_open
                      _nfs4_do_open
                        _nfs4_open_and_get_state
                          _nfs4_proc_open
                            nfs4_run_open_task
                              rpc_run_task
                                rpc_execute
```

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
            nfsd4_open  
    		nfsd4_encode_operation
              nfsd4_encode_open
```
### 2）第二次cat

https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_server_cat_second_gdb_log.txt

* ***Q：服务器端没有调用nfsd4_open函数？***

* A：应该是读取了本地缓存，所以没有像服务器端发送open请求。
