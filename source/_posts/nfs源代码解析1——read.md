---
title: nfs源代码解析1——read
date: 2022-06-14 22:33:53
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,read]
---

# 1. Client

nfs v4

generic_file_read_iter有两种读的方式，分为直接IO与缓存IO，IO类型由读取的file类型决定（include/linux/fs.h的init_sync_kiocb函数中）。

总的来说就是客户端发起读系统调用，之后根据是否有缓存的情况发起读rpc请求。

==Q1：读操作应该存在某种缓存机制?==

==A1：是的，缓存IO。==

==重点研究nfs_file_read函数以及一系列文件系统通用的IO流程==

```c
read
	ksys_read
		vfs_read
			new_sync_read
				call_read_iter
					nfs_file_read
						generic_file_read_iter
							filemap_read
								filemap_get_pages
									page_cache_sync_readahead
										page_cache_sync_ra
											ondemand_readahead
												page_cache_ra_order
													do_page_cache_ra
														page_cache_ra_unbounded
															read_pages
																nfs_readahead
    																nfs_pageio_complete_read
																		nfs_pageio_complete
																			nfs_pageio_complete_mirror
																				nfs_pageio_doio
																					nfs_generic_pg_pgios
																						nfs_initiate_pgio
																							rpc_run_task
																								rpc_execute
```

# 2. Server

服务器端收到请求，进行处理并返回结果。

==重点研究nfsd4_read函数==

```c
kthread
	nfsd
		svc_process
			svc_process_common
				nfsd_dispatch
					nfsd4_proc_compound
						nfsd4_read
```

