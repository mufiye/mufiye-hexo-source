---
title: nfs源代码解析6——other
date: 2022-06-15 21:57:08
categories: 操作系统内核
tags: [源码解析,文件系统,nfs,other]
---

# 1. Delegation

==这是什么？（设置rpc_execute断点时捕捉到的，其会自动调用）==

```c
kthread
  nfs4_run_state_manager
    nfs4_state_manager
      nfs_client_return_marked_delegations
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

==这是什么？（设置rpc_execute断点时捕捉到的，其会自动调用）==

```
kthread
  worker_thread
    process_one_work
      nfs4_renew_state
        nfs41_proc_async_sequence
          _nfs41_proc_sequence
            rpc_run_task
              rpc_execute
```

