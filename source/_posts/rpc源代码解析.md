---
title: rpc源代码解析
date: 2022-06-21 11:25:06
categories: 操作系统内核
tags: [源码解析,net,rpc]
---
# RPC - 客户端
## rpc_clnt
代表一个client handle，由rpc_create创建，它会创建transport，并发送ping，判断是否对方支持这个RPC。
```c
struct rpc_clnt {
    refcount_t         cl_count;    /* Number of references */
    unsigned int       cl_clid;     /* client id */
    struct list_head   cl_clients;  /* Global list of clients */
    struct list_head   cl_tasks;	/* List of tasks */
    atomic_t           cl_pid;      /* task PID counter */
    spinlock_t         cl_lock;     /* spinlock */
    struct rpc_xprt __rcu *	cl_xprt;    /* transport */
    const struct rpc_procinfo *cl_procinfo;	/* procedure info */
    u32			cl_prog,	/* RPC program number */
				cl_vers,	/* RPC version number */
				cl_maxproc;	/* max procedure number */
    struct rpc_auth *	cl_auth;	/* authenticator */
	struct rpc_stat *	cl_stats;	/* per-program statistics */
	struct rpc_iostats *	cl_metrics;	/* per-client statistics */
    ...
	const struct rpc_timeout *cl_timeout;	/* Timeout strategy */
    ...
	struct rpc_clnt *	cl_parent;	/* Points to parent of clones */
	struct rpc_rtt		cl_rtt_default;
	struct rpc_timeout	cl_timeout_default;
	const struct rpc_program *cl_program;
	const char *		cl_principal;	/* use for machine cred */
#if IS_ENABLED(CONFIG_SUNRPC_DEBUG)
	struct dentry		*cl_debugfs;	/* debugfs directory */
#endif
	struct rpc_sysfs_client *cl_sysfs;	/* sysfs directory */
	/* cl_work is only needed after cl_xpi is no longer used,
	 * and that are of similar size
	 */
	union {
		struct rpc_xprt_iter	cl_xpi;
		struct work_struct	cl_work;
	};
	const struct cred	*cl_cred;
	unsigned int		cl_max_connect; /* max number of transports not to the same IP */
};
```
## rpc_xprt
代表一个连接（感觉有些像TCP连接），由xprt_create_transport创建。
```c
struct rpc_xprt {
    struct kref		kref;		/* Reference count */
    const struct rpc_xprt_ops *ops;		/* transport methods */
    unsigned int		id;		/* transport id */
    
    const struct rpc_timeout *timeout;	/* timeout parms */
    struct sockaddr_storage	addr;		/* server address */
    size_t			addrlen;	/* size of server address */
    int			prot;		/* IP protocol */
    
    unsigned long		cong;		/* current congestion */
	unsigned long		cwnd;		/* congestion window */
    
    size_t			max_payload;	/* largest RPC payload size, in bytes */
    
    struct rpc_wait_queue	binding;	/* requests waiting on rpcbind */
    struct rpc_wait_queue	sending;	/* requests waiting to send */
    struct rpc_wait_queue	pending;	/* requests in flight */
    struct rpc_wait_queue	backlog;	/* waiting for slot */
    struct list_head	free;		/* free slots */
    unsigned int		max_reqs;	/* max number of slots */
	unsigned int		min_reqs;	/* min number of slots */
	unsigned int		num_reqs;	/* total slots */
	unsigned long		state;		/* transport state */
    unsigned char		resvport   : 1,	/* use a reserved port */
				        reuseport  : 1; /* reuse port on reconnect */
	atomic_t		    swapper;	/* we're swapping over this transport */
	unsigned int        bind_index;	/* bind function index */
    	/*
	 * Multipath
	 */
	struct list_head	xprt_switch;

	/*
	 * Connection of transports
	 */
	unsigned long		bind_timeout, reestablish_timeout;
	unsigned int		connect_cookie;	/* A cookie that gets bumped every time the                                                  transport is reconnected */

	/*
	 * Disconnection of idle transports
	 */
	struct work_struct	task_cleanup;
	struct timer_list	timer;
	unsigned long		last_used,
				        idle_timeout,
				        connect_timeout,
				        max_reconnect_timeout;

	/*
	 * Send stuff
	 */
	atomic_long_t		queuelen;
	spinlock_t		transport_lock;	/* lock transport info */
	spinlock_t		reserve_lock;	/* lock slot table */
	spinlock_t		queue_lock;	/* send/receive queue lock */
	u32			xid;		/* Next XID value to use */
	struct rpc_task *	snd_task;	/* Task blocked in send */

	struct list_head	xmit_queue;	/* Send queue */
	atomic_long_t		xmit_queuelen;

	struct svc_xprt		*bc_xprt;	/* NFSv4.1 backchannel */
#if defined(CONFIG_SUNRPC_BACKCHANNEL)
	struct svc_serv		*bc_serv;       /* The RPC service which will */
						/* process the callback */
	unsigned int		bc_alloc_max;
	unsigned int		bc_alloc_count;	/* Total number of preallocs */
	atomic_t		bc_slot_count;	/* Number of allocated slots */
	spinlock_t		bc_pa_lock;	/* Protects the preallocated
						 * items */
	struct list_head	bc_pa_list;	/* List of preallocated
						 * backchannel rpc_rqst's */
#endif /* CONFIG_SUNRPC_BACKCHANNEL */

	struct rb_root		recv_queue;	/* Receive queue */

	struct {
		unsigned long		bind_count,	/* total number of binds */
					connect_count,	/* total number of connects */
					connect_start,	/* connect start timestamp */
					connect_time,	/* jiffies waiting for connect */
					sends,		/* how many complete requests */
					recvs,		/* how many complete requests */
					bad_xids,	/* lookup_rqst didn't find XID */
					max_slots;	/* max rpc_slots used */

		unsigned long long	req_u,		/* average requests on the wire */
					bklog_u,	/* backlog queue utilization */
					sending_u,	/* send q utilization */
					pending_u;	/* pend q utilization */
	} stat;

	struct net		*xprt_net;
	netns_tracker		ns_tracker;
	const char		*servername;
	const char		*address_strings[RPC_DISPLAY_MAX];
#if IS_ENABLED(CONFIG_SUNRPC_DEBUG)
	struct dentry		*debugfs;		/* debugfs directory */
#endif
	struct rcu_head		rcu;
	const struct xprt_class	*xprt_class;
	struct rpc_sysfs_xprt	*xprt_sysfs;
	bool			main; /*mark if this is the 1st transport */
};
```
## rpc_task
代表一个task,内部是一个有限状态机。

```c
struct rpc_task {
    atomic_t		tk_count;	/* Reference count */
    int			tk_status;	/* result of last operation */
    struct list_head	tk_task;	/* global list of tasks */
    
    /*
	 * callback：to be executed after waking up
	 * action：next procedure for async tasks
	 */
	void			(*tk_callback)(struct rpc_task *);
	void			(*tk_action)(struct rpc_task *);
    
    unsigned long		tk_timeout;	/* timeout for rpc_sleep() */
	unsigned long		tk_runstate;	/* Task run status */
    
    struct rpc_wait_queue 	*tk_waitqueue;	/* RPC wait queue we're on */
	union {
		struct work_struct	tk_work;	/* Async task work queue */
		struct rpc_wait		tk_wait;	/* RPC wait */
	} u;

	int			tk_rpc_status;	/* Result of last RPC operation */
    /*
	 * RPC call state
	 */
	struct rpc_message	tk_msg;		/* RPC call info */
	void *			tk_calldata;	/* Caller private data */
	const struct rpc_call_ops *tk_ops;	/* Caller callbacks */

	struct rpc_clnt *	tk_client;	/* RPC client */
	struct rpc_xprt *	tk_xprt;	/* Transport */
	struct rpc_cred *	tk_op_cred;	/* cred being operated on */

	struct rpc_rqst *	tk_rqstp;	/* RPC request */

	struct workqueue_struct	*tk_workqueue;	/* Normally rpciod, but could
						 * be any workqueue
						 */
	ktime_t			tk_start;	/* RPC task init timestamp */

	pid_t			tk_owner;	/* Process id for batching tasks */
	unsigned short		tk_flags;	/* misc flags */
	unsigned short		tk_timeouts;	/* maj timeouts */

#if IS_ENABLED(CONFIG_SUNRPC_DEBUG) || IS_ENABLED(CONFIG_TRACEPOINTS)
	unsigned short		tk_pid;		/* debugging aid */
#endif
	unsigned char		tk_priority : 2,/* Task priority */
				tk_garb_retry : 2,
				tk_cred_retry : 2,
				tk_rebind_retry : 2;
};
```
## rpc_message

rpc message里面包括传输的参数，返回的参数地址，发生/接收时候用到编码/解码函数等。

```c
struct rpc_message {
	const struct rpc_procinfo *rpc_proc;	/* Procedure information */
	void *			rpc_argp;	/* Arguments */
	void *			rpc_resp;	/* Result */
	const struct cred *	rpc_cred;	/* Credentials */
};
```

## rpc_task_setup

```c
struct rpc_task_setup {
	struct rpc_task *task;
	struct rpc_clnt *rpc_client;
	struct rpc_xprt *rpc_xprt;
	struct rpc_cred *rpc_op_cred;	/* credential being operated on */
	const struct rpc_message *rpc_message;
	const struct rpc_call_ops *callback_ops;
	void *callback_data;
	struct workqueue_struct *workqueue;
	unsigned short flags;
	signed char priority;
};
```

## workqueue_struct

* rpciod_workqueue：调用rpc_execute运行rpc task状态机。同步RPC调用运行在调用者的运行上下文，异步RPC在这个work queue执行rpc_execute。
* xprtiod_workqueue：处理网络收发
* nfsiod_workqueue：rpc task使用结束后，需要调用rpc_free_task进行回收，回收可以同步，也可以异步，异步回收时在这个work queue中。这是nfs的RPC请求时提供的。

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

## svc_rqst

```c
/*
 * The context of a single thread, including the request currently being
 * processed.
 */
struct svc_rqst {
    struct list_head	rq_all;		/* all threads list */
	struct rcu_head		rq_rcu_head;	/* for RCU deferred kfree */
	struct svc_xprt *	rq_xprt;	/* transport ptr */

	struct sockaddr_storage	rq_addr;	/* peer address */
	size_t			rq_addrlen;
	struct sockaddr_storage	rq_daddr;	/* dest addr of request - reply from here */
    size_t			rq_daddrlen;
    
    struct svc_serv *	rq_server;	/* RPC service definition */
	struct svc_pool *	rq_pool;	/* thread pool */
	const struct svc_procedure *rq_procinfo;/* procedure info */
	struct auth_ops *	rq_authop;	/* authentication flavour */
	struct svc_cred		rq_cred;	/* auth info */
	void *			rq_xprt_ctxt;	/* transport specific context ptr */
	struct svc_deferred_req*rq_deferred;	/* deferred request we are replaying */

	struct xdr_buf		rq_arg;
	struct xdr_stream	rq_arg_stream;
	struct xdr_stream	rq_res_stream;
	struct page		*rq_scratch_page;
	struct xdr_buf		rq_res;
	struct page		*rq_pages[RPCSVC_MAXPAGES + 1];
	struct page *		*rq_respages;	/* points into rq_pages */
	struct page *		*rq_next_page; /* next reply page to use */
	struct page *		*rq_page_end;  /* one past the last page */

	struct pagevec		rq_pvec;
	struct kvec		rq_vec[RPCSVC_MAXPAGES]; /* generally useful.. */
	struct bio_vec		rq_bvec[RPCSVC_MAXPAGES];

	__be32			rq_xid;		/* transmission id */
	u32			rq_prog;	/* program number */
	u32			rq_vers;	/* program version */
	u32			rq_proc;	/* procedure number */
	u32			rq_prot;	/* IP protocol */
	int			rq_cachetype;	/* catering to nfsd */
#define	RQ_SECURE	(0)			/* secure port */
#define	RQ_LOCAL	(1)			/* local request */
#define	RQ_USEDEFERRAL	(2)			/* use deferral */
#define	RQ_DROPME	(3)			/* drop current reply */
#define	RQ_SPLICE_OK	(4)			/* turned off in gss privacy
						 * to prevent encrypting page
						 * cache pages */
#define	RQ_VICTIM	(5)			/* about to be shut down */
#define	RQ_BUSY		(6)			/* request is busy */
#define	RQ_DATA		(7)			/* request has data */
	unsigned long		rq_flags;	/* flags field */
	ktime_t			rq_qtime;	/* enqueue time */

	void *			rq_argp;	/* decoded arguments */
	void *			rq_resp;	/* xdr'd results */
	void *			rq_auth_data;	/* flavor-specific data */
	__be32			rq_auth_stat;	/* authentication status */
	int			rq_auth_slack;	/* extra space xdr code
						 * should leave in head
						 * for krb5i, krb5p.
						 */
	int			rq_reserved;	/* space on socket outq
						 * reserved for this request
						 */
	ktime_t			rq_stime;	/* start time */

	struct cache_req	rq_chandle;	/* handle passed to caches for 
						 * request delaying 
						 */
	/* Catering to nfsd */
	struct auth_domain *	rq_client;	/* RPC peer info */
	struct auth_domain *	rq_gssclient;	/* "gss/"-style peer info */
	struct svc_cacherep *	rq_cacherep;	/* cache info */
	struct task_struct	*rq_task;	/* service thread */
	spinlock_t		rq_lock;	/* per-request lock */
	struct net		*rq_bc_net;	/* pointer to backchannel's
						 * net namespace
						 */
	void **			rq_lease_breaker; /* The v4 client breaking a lease */
};
```

## svc_serv

```c
/*
 * RPC service.
 *
 * An RPC service is a ``daemon,'' possibly multithreaded, which
 * receives and processes incoming RPC messages.
 * It has one or more transport sockets associated with it, and maintains
 * a list of idle threads waiting for input.
 *
 * We currently do not support more than one RPC program per daemon.
 */
struct svc_serv {
	struct svc_program *	sv_program;	/* RPC program */
	struct svc_stat *	sv_stats;	/* RPC statistics */
	spinlock_t		sv_lock;
	struct kref		sv_refcnt;
	unsigned int		sv_nrthreads;	/* # of server threads */
	unsigned int		sv_maxconn;	/* max connections allowed or
						 * '0' causing max to be based
						 * on number of threads. */

	unsigned int		sv_max_payload;	/* datagram payload size */
	unsigned int		sv_max_mesg;	/* max_payload + 1 page for overheads */
	unsigned int		sv_xdrsize;	/* XDR buffer size */
	struct list_head	sv_permsocks;	/* all permanent sockets */
	struct list_head	sv_tempsocks;	/* all temporary sockets */
	int			sv_tmpcnt;	/* count of temporary sockets */
	struct timer_list	sv_temptimer;	/* timer for aging temporary sockets */

	char *			sv_name;	/* service name */

	unsigned int		sv_nrpools;	/* number of thread pools */
	struct svc_pool *	sv_pools;	/* array of thread pools */
	int			(*sv_threadfn)(void *data);

#if defined(CONFIG_SUNRPC_BACKCHANNEL)
	struct list_head	sv_cb_list;	/* queue for callback requests
						 * that arrive over the same
						 * connection */
	spinlock_t		sv_cb_lock;	/* protects the svc_cb_list */
	wait_queue_head_t	sv_cb_waitq;	/* sleep here if there are no
						 * entries in the svc_cb_list */
	bool			sv_bc_enabled;	/* service uses backchannel */
#endif /* CONFIG_SUNRPC_BACKCHANNEL */
};
```

## svc_procedure

```c
/*
 * RPC procedure info
 */
struct svc_procedure {
	/* process the request: */
	__be32			(*pc_func)(struct svc_rqst *);
	/* XDR decode args: */
	bool			(*pc_decode)(struct svc_rqst *rqstp,
					     struct xdr_stream *xdr);
	/* XDR encode result: */
	bool			(*pc_encode)(struct svc_rqst *rqstp,
					     struct xdr_stream *xdr);
	/* XDR free result: */
	void			(*pc_release)(struct svc_rqst *);
	unsigned int		pc_argsize;	/* argument struct size */
	unsigned int		pc_ressize;	/* result struct size */
	unsigned int		pc_cachetype;	/* cache info (NFS) */
	unsigned int		pc_xdrressize;	/* maximum size of XDR reply */
	const char *		pc_name;	/* for display */
};
```

## svc_program

```c
/*
 * List of RPC programs on the same transport endpoint
 */
struct svc_program {
	struct svc_program *	pg_next;	/* other programs (same xprt) */
	u32			pg_prog;	/* program number */
	unsigned int		pg_lovers;	/* lowest version */
	unsigned int		pg_hivers;	/* highest version */
	unsigned int		pg_nvers;	/* number of versions */
	const struct svc_version **pg_vers;	/* version array */
	char *			pg_name;	/* service name */
	char *			pg_class;	/* class name: services sharing authentication */
	struct svc_stat *	pg_stats;	/* rpc statistics */
	int			(*pg_authenticate)(struct svc_rqst *);
	__be32			(*pg_init_request)(struct svc_rqst *,
						   const struct svc_program *,
						   struct svc_process_info *);
	int			(*pg_rpcbind_set)(struct net *net,
						  const struct svc_program *,
						  u32 version, int family,
						  unsigned short proto,
						  unsigned short port);
};
```

## svc_stat

```c
struct svc_stat {
	struct svc_program *	program;

	unsigned int		netcnt,
				netudpcnt,
				nettcpcnt,
				nettcpconn;
	unsigned int		rpccnt,
				rpcbadfmt,
				rpcbadauth,
				rpcbadclnt;
};
```

# 参考

1. https://www.codeleading.com/article/26211011766/
2. https://github.com/chenxiaosonggithub/blog/blob/master/kernel/nfs/nfs.md
