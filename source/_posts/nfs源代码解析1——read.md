---
title: nfs源代码解析1——read
date: 2022-06-15 14:07:16
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,read]
---

# 1. 读取成功

## 1.1 Client

nfs v4

generic_file_read_iter有两种读的方式，分为直接IO与缓存IO，IO类型由读取的file类型决定（include/linux/fs.h的init_sync_kiocb函数中）。

总的来说就是客户端发起读系统调用，之后根据是否有缓存的情况发起读rpc请求。

**Q0：读操作应该存在某种缓存机制?**

**A0：是的，缓存IO。**

**重点研究nfs_file_read函数以及一系列文件系统通用的IO流程**

```c
read
  ksys_read
    vfs_read
      new_sync_read
        call_read_iter /* file->f_op->read_iter */
          nfs_file_read /* 读取nfs file */
            generic_file_read_iter /* 用于更新缓存数据 */
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
                                  nfs_pageio_init_read
                                    nfs_pageio_init /* 初始化rw_ops为nfs_rw_read_ops */
    							  readahead_page /* Get the next page to read. */
    							  readpage_async_filler /* 创建读取page的请求 */
                                  nfs_pageio_complete_read 
                                    nfs_pageio_complete
                                      nfs_pageio_complete_mirror
                                        nfs_pageio_doio
                                          nfs_generic_pg_pgios 
                                            nfs_initiate_pgio  /* 填充rpc_task_setup，启动rpc task */
                                              rpc_run_task
                                                rpc_execute
```

### 重点结构体

#### 1）nfs_readdesc

nfs读操作的描述符

```c
struct nfs_readdesc {
	struct nfs_pageio_descriptor pgio;
	struct nfs_open_context *ctx;
};
```

#### 2）nfs_pageio_descriptor

nfs_pageio_descriptor这个结构体中包含了nfs缓存页的inode信息，向服务器发起IO请求的函数以及一些控制信息。

```c
struct nfs_pageio_descriptor {
	struct inode		*pg_inode;  /* page对应的inode组成的链表？ */
	const struct nfs_pageio_ops *pg_ops; /* nfs page相关的操作 */
	const struct nfs_rw_ops *pg_rw_ops; /* nfs读写页操作 */
	
    ...
    
	const struct rpc_call_ops *pg_rpc_callops; /* rpc请求相关的函数 */
	const struct nfs_pgio_completion_ops *pg_completion_ops; /* 当读写结束或者出错后调用这里的函数 */
	
    ...
        
	u32			pg_mirror_count;
	struct nfs_pgio_mirror	*pg_mirrors;
	struct nfs_pgio_mirror	 pg_mirrors_static[1];
	struct nfs_pgio_mirror	*pg_mirrors_dynamic;
	u32			pg_mirror_idx;	/* current mirror */
    
	...
};
```

#### 3）nfs_page

nfs_page是nfs的page结构，每个nfs_page关联一个缓存页。

```c
struct nfs_page {
	struct list_head	wb_list;	/* Defines state of page: */
	struct page		*wb_page;	   /* page to read in/write out */
	struct nfs_lock_context	*wb_lock_context;	/* lock context info */
	pgoff_t			wb_index;	/* Offset >> PAGE_SHIFT */
	unsigned int		wb_offset,	/* Offset & ~PAGE_MASK */
				wb_pgbase,	/* Start of page data */
				wb_bytes;	/* Length of request */
	struct kref		wb_kref;	/* reference count */
	unsigned long		wb_flags;
	struct nfs_write_verifier	wb_verf;	/* Commit cookie */
	struct nfs_page		*wb_this_page;  /* list of reqs for this page */
	struct nfs_page		*wb_head;       /* head pointer for req list */
	unsigned short		wb_nio;		/* Number of I/O attempts */
};
```

#### 4）nfs_pgio_header

nfs_pageio_descriptor可以组合在一起作为一条报文，nfs_pgio_header是该报文的头部信息。

```c
struct nfs_pgio_header {
	struct inode		*inode;
	const struct cred		*cred;
	struct list_head	pages;
	struct nfs_page		*req;
	struct nfs_writeverf	verf;		/* Used for writes */
	fmode_t			rw_mode;
	struct pnfs_layout_segment *lseg;
	loff_t			io_start;
	const struct rpc_call_ops *mds_ops;
	void (*release) (struct nfs_pgio_header *hdr);
	const struct nfs_pgio_completion_ops *completion_ops;
	const struct nfs_rw_ops	*rw_ops;
	struct nfs_io_completion *io_completion;
	
    ...

	/*
	 * rpc data
	 */
	struct rpc_task		task;
	struct nfs_fattr	fattr;
	struct nfs_pgio_args	args;		/* argument struct */
	struct nfs_pgio_res	res;		/* result struct */
    
	...
};
```

### nfs_file_read

使用直接IO或者缓存IO读取数据，如果使用缓存的方式，nfs_revalidate_mapping函数检查了本地缓存的文件属性，如果文件属性无效了就向服务器发起GETATTR请求以此更新本地缓存的文件属性。之后读取数据时如果缓存中的数据有效，就使用该数据；如果缓存数据失效则调用generic_file_read_iter更新缓存数据。

### nfs_readahead

nfs_find_open_context函数和get_nfs_open_context函数获取上下文信息，这主要涉及同步互斥以及访问权限？。nfs_pageio_init_read初始化了一个nfs_pageio_descriptor结构。之后循环体使用readahead_page函数获取需要读取的页，readpage_async_filler函数负责创建读请求，将读请求添加到队列中，其会为缓存页创建一个nfs_page结构。nfs_pageio_complete_read函数负责发起读请求。

### readpage_async_filler

首先检查fs cache中的数据是否有效，如果有效就用fs cache中的数据填充缓存页，而不必创建read请求了；如果无效则创建请求并将请求添加到nfs_pageio_descriptor中。

### Questions

* *Q1：**trace**函数的作用（比如**trace_nfs_aop_readahead**）*

* A1：

  

* *Q2：**nfs_open_context**是什么？*
* A2：



* *Q3：**nfs_pgio_mirror**是干嘛用的？*
* A3：

## 1.2 Server

服务器端收到请求，进行处理并返回结果。

**重点研究nfsd4_read函数**

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound  // op->status = op->opdesc->op_func(...)
            nfsd4_read  /* 设置读取的位置、数量 */
            nfsd4_encode_operation  /* 将read operation的结果编码 */
    		  nfsd4_encode_read  /* in nfsd4_enc_ops */
                nfsd4_encode_splice_read  /* 将读取到的结果编码到resp用于传输给client */
```

### Questions

* *Q1：zero copy read？*
* A1：https://blog.csdn.net/u013256816/article/details/52589524

# 2. 读取失败（暂不做分析）

## 2.1 没有该文件

### Client

https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_read_fail_gdb_log.txt

### Server

https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_server_read_fail_gdb_log.txt

## 2.2 no reply（gdb pause）

https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_read_disconnected_gdb_log.txt

## 2.3 poweroff

https://github.com/mufiye/mufiye_backup/blob/master/nfs/nfs_client_read_poweroff_gdb_log.txt
