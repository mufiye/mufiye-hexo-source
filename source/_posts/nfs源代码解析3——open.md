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
                          nfs4_call_sync
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

#### nfs_do_access

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

#### nfs_atomic_open



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
                nfs4_file_open  /* nfs打开文件 */
                  nfs4_atomic_open /* 原子性地打开一个文件 */
    				nfs4_label_init_security
                    nfs4_do_open
                      _nfs4_do_open
                        _nfs4_open_and_get_state
                          _nfs4_proc_open
                            nfs4_run_open_task
                              rpc_run_task
                                rpc_execute
```

* ***Q：什么是原子性地打开文件？***

* A：
* ***Q：_nfs4_open_and_get_state获取的是什么状态？***
* A：
* ***Q：打开文件的过程中发送了什么请求？***
* A：

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
            nfsd4_open  /* 服务器端打开，重点研究 */
    		nfsd4_encode_operation
              nfsd4_encode_open  /* 将open operation的返回信息编码，重点研究 */
```
* ***Q：nfsd4_open和nfsd4_encode_open做了什么？***

* A：

### 2）第二次cat

https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_server_cat_second_gdb_log.txt

* ***Q：服务器端没有调用nfsd4_open函数？***

* A：
