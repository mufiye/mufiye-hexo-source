---
title: nfs增加耗时统计信息
date: 2022-06-25 13:55:11
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,ospp]
---

# 1. Client

所有的请求都要执行rpc_execute函数，并且会在call_start函数中记录下为何种操作（可以参考输出统计信息的源码分析文档）。因此做客户端的统计耗时统计信息就需要认真阅读代码，分析相应operation的出口和入口位置，个人觉得，一般的operation都有其对应的函数。

# 2. Server

```c
kthread
  nfsd
    svc_process
      svc_process_common
        nfsd_dispatch
          nfsd4_proc_compound  /* 处理复合请求（多种请求混合） */
            nfsd4_increment_op_stats
```

在下面的循环体中添加，保证对所有op都适用

```c
while (!status && resp->opcnt < args->opcnt) {
	op = &args->ops[resp->opcnt++];
	...
}
```

