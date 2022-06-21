---
title: rpc源代码解析
date: 2022-06-21 11:25:06
categories: 操作系统内核
tags: [源码解析,net,rpc]
---
# RPC - 客户端
## rpc_xprt
代表一个连接，由xprt_create_transport创建。
```c
struct rpc_xprt {
}
```
## rpc_clnt
代表一个client handle，由rpc_create创建，它会创建transport，并发送ping，判断是否对方支持这个RPC。
```c
struct rpc_clnt {
}
```
## rpc_task
代表一个task,内部是一个有限状态机。

```c
struct rpc_task {
}
```
## rpc_rqst
表示一个rpc的请求
```c
struct rpc_rqst {
}
```
## rpc task处理流程
RPC的处理流程本质上是一个有限状态机，状态机状态的改变通过修改task->tk_action实现（处理函数为__rpc_execute）。
```c
// high-level RPC interface.
/* 0.  Initial state */
call_start

/* 1.  Reserve an RPC call slot */
call_reserve

/* 1b.  Grok the result of xprt_reserve() */
call_reserveresult

/* 1c.  Retry reserving an RPC call slot */
call_retry_reserve

/* 2.  Bind and/or refresh the credentials */
call_refresh

/* 2a.  Process the results of a credential refresh */
call_refreshresult

/* 2b.  Allocate the buffer. */
call_allocate

/* 3.  Encode arguments of an RPC call */
call_encode

/* 4.  Get the server port number if not yet set */
call_bind

/* 4a.  Sort out bind result */
call_bind_status

/* 4b.  Connect to the RPC server */
call_connect

/* 4c.  Sort out connect result */
call_connect_status

/* 5.  Transmit the RPC request, and wait for reply */
call_transmit

/* 5a.  Handle cleanup after a transmission */
call_transmit_status

/* 5b.  Send the backchannel RPC reply. */
call_bc_transmit

/* 6.  Sort out the RPC call status */
call_status

/* 7.  Decode the RPC reply */
call_decode
```
# RPC - 服务器端

```c
/* Process the RPC request. */
svc_process
    
/* Common routine for processing the RPC request. */
svc_process_common
```

# 参考

1. https://www.codeleading.com/article/26211011766/
2. https://github.com/chenxiaosonggithub/blog/blob/master/kernel/nfs/nfs.md
