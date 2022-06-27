---
title: nfs源代码解析6——other
date: 2022-06-15 21:57:08
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,other]
---

# 1. Delegation

Delegation是一种用于保证nfs文件系统数据一致性的机制，当nfs有多个客户端的时候，每个客户端本地都会有一份文件的数据缓存，此时为了保证各个客户端的数据一致性，通常是在读写操作前发起相应的getAttr请求来判断本地缓存是否有效。

而Delegation这种机制的基本原理是当一个客户端A打开一个文件的时候服务器会给客户端发一个凭证，只要客户端A持有这个凭证就可以认为和其它客户端保持数据一致，而当客户端B试图访问服务器端上的该文件时，服务器端会先暂停响应客户端B的请求，而向客户端A发送相应的请求，客户端A收到请求后会将凭证交还给服务器（该凭证也就失效了）。而由于正常的请求传输通道都是客户端发送请求到服务器端，服务器端处理后进行响应。在delegation的过程中，恰恰相反，因此需要创建一个反向通道用于delegation。

下面这段backtrace也客户端收到服务器端请求后返还delegation的处理流程：

```c
kthread
  nfs4_run_state_manager
    nfs4_state_manager  /* 状态监视器，根据不同状态做不同的事情 */
      nfs_client_return_marked_delegations  /* 返还之前的凭证 */
        nfs_client_for_each_server
          __nfs_list_for_each_server
            nfs_server_return_marked_delegations
              nfs_end_delegation_return
                nfs_do_return_delegation
                  nfs4_proc_delegreturn
                    _nfs4_proc_delegreturn
                      rpc_run_task
                        rpc_execute
```

# 2. Sequence

**这是什么？（设置rpc_execute断点时捕捉到的，其会自动调用）**

```c
kthread
  worker_thread
    process_one_work
      nfs4_renew_state
        nfs41_proc_async_sequence
          _nfs41_proc_sequence
            rpc_run_task
              rpc_execute
```

