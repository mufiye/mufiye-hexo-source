---
title: nfs源代码解析——重点结构体
date: 2022-06-24 12:50:15
categories: 操作系统内核
tags: [源码解析,文件系统,nfs]
---

# Super Block

```c
struct nfs_server {
	struct nfs_client *	nfs_client;	/* shared client and NFS4 state */
	struct list_head	client_link;	/* List of other nfs_server structs
						 * that share the same client
						 */
	struct list_head	master_link;	/* link in master servers list */
	struct rpc_clnt *	client;		/* RPC client handle */
	struct rpc_clnt *	client_acl;	/* ACL RPC client handle */
	struct nlm_host		*nlm_host;	/* NLM client handle */
	struct nfs_iostats __percpu *io_stats;	/* I/O statistics */
	atomic_long_t		writeback;	/* number of writeback pages */
	unsigned int		write_congested;/* flag set when writeback gets too high */
	unsigned int		flags;		/* various flags */

/* The following are for internal use only. Also see uapi/linux/nfs_mount.h */
#define NFS_MOUNT_LOOKUP_CACHE_NONEG	0x10000
#define NFS_MOUNT_LOOKUP_CACHE_NONE	0x20000
#define NFS_MOUNT_NORESVPORT		0x40000
#define NFS_MOUNT_LEGACY_INTERFACE	0x80000
#define NFS_MOUNT_LOCAL_FLOCK		0x100000
#define NFS_MOUNT_LOCAL_FCNTL		0x200000
#define NFS_MOUNT_SOFTERR		0x400000
#define NFS_MOUNT_SOFTREVAL		0x800000
#define NFS_MOUNT_WRITE_EAGER		0x01000000
#define NFS_MOUNT_WRITE_WAIT		0x02000000
#define NFS_MOUNT_TRUNK_DISCOVERY	0x04000000

	unsigned int		fattr_valid;	/* Valid attributes */
	unsigned int		caps;		/* server capabilities */
	unsigned int		rsize;		/* read size */
	unsigned int		rpages;		/* read size (in pages) */
	unsigned int		wsize;		/* write size */
	unsigned int		wpages;		/* write size (in pages) */
	unsigned int		wtmult;		/* server disk block size */
	unsigned int		dtsize;		/* readdir size */
	unsigned short		port;		/* "port=" setting */
	unsigned int		bsize;		/* server block size */
#ifdef CONFIG_NFS_V4_2
	unsigned int		gxasize;	/* getxattr size */
	unsigned int		sxasize;	/* setxattr size */
	unsigned int		lxasize;	/* listxattr size */
#endif
	unsigned int		acregmin;	/* attr cache timeouts */
	unsigned int		acregmax;
	unsigned int		acdirmin;
	unsigned int		acdirmax;
	unsigned int		namelen;
	unsigned int		options;	/* extra options enabled by mount */
	unsigned int		clone_blksize;	/* granularity of a CLONE operation */
#define NFS_OPTION_FSCACHE	0x00000001	/* - local caching enabled */
#define NFS_OPTION_MIGRATION	0x00000002	/* - NFSv4 migration enabled */

	enum nfs4_change_attr_type
				change_attr_type;/* Description of change attribute */

	struct nfs_fsid		fsid;
	__u64			maxfilesize;	/* maximum file size */
	struct timespec64	time_delta;	/* smallest time granularity */
	unsigned long		mount_time;	/* when this fs was mounted */
	struct super_block	*super;		/* VFS super block */
	dev_t			s_dev;		/* superblock dev numbers */
	struct nfs_auth_info	auth_info;	/* parsed auth flavors */

#ifdef CONFIG_NFS_FSCACHE
	struct fscache_volume	*fscache;	/* superblock cookie */
	char			*fscache_uniq;	/* Uniquifier (or NULL) */
#endif

	u32			pnfs_blksize;	/* layout_blksize attr */
#if IS_ENABLED(CONFIG_NFS_V4)
	u32			attr_bitmask[3];/* V4 bitmask representing the set
						   of attributes supported on this
						   filesystem */
	u32			attr_bitmask_nl[3];
						/* V4 bitmask representing the
						   set of attributes supported
						   on this filesystem excluding
						   the label support bit. */
	u32			exclcreat_bitmask[3];
						/* V4 bitmask representing the
						   set of attributes supported
						   on this filesystem for the
						   exclusive create. */
	u32			cache_consistency_bitmask[3];
						/* V4 bitmask representing the subset
						   of change attribute, size, ctime
						   and mtime attributes supported by
						   the server */
	u32			acl_bitmask;	/* V4 bitmask representing the ACEs
						   that are supported on this
						   filesystem */
	u32			fh_expire_type;	/* V4 bitmask representing file
						   handle volatility type for
						   this filesystem */
	struct pnfs_layoutdriver_type  *pnfs_curr_ld; /* Active layout driver */
	struct rpc_wait_queue	roc_rpcwaitq;
	void			*pnfs_ld_data;	/* per mount point data */

	/* the following fields are protected by nfs_client->cl_lock */
	struct rb_root		state_owners;
#endif
	struct ida		openowner_id;
	struct ida		lockowner_id;
	struct list_head	state_owners_lru;
	struct list_head	layouts;
	struct list_head	delegations;
	struct list_head	ss_copies;

	unsigned long		mig_gen;
	unsigned long		mig_status;
#define NFS_MIG_IN_TRANSITION		(1)
#define NFS_MIG_FAILED			(2)
#define NFS_MIG_TSM_POSSIBLE		(3)

	void (*destroy)(struct nfs_server *);

	atomic_t active; /* Keep trace of any activity to this server */

	/* mountd-related mount options */
	struct sockaddr_storage	mountd_address;
	size_t			mountd_addrlen;
	u32			mountd_version;
	unsigned short		mountd_port;
	unsigned short		mountd_protocol;
	struct rpc_wait_queue	uoc_rpcwaitq;

	/* XDR related information */
	unsigned int		read_hdrsize;

	/* User namespace info */
	const struct cred	*cred;
	bool			has_sec_mnt_opts;
};
```

```c
struct nfs_client {
	refcount_t		cl_count;
	atomic_t		cl_mds_count;
	int			cl_cons_state;	/* current construction state (-ve: init error) */
#define NFS_CS_READY		0		/* ready to be used */
#define NFS_CS_INITING		1		/* busy initialising */
#define NFS_CS_SESSION_INITING	2		/* busy initialising  session */
	unsigned long		cl_res_state;	/* NFS resources state */
#define NFS_CS_CALLBACK		1		/* - callback started */
#define NFS_CS_IDMAP		2		/* - idmap started */
#define NFS_CS_RENEWD		3		/* - renewd started */
#define NFS_CS_STOP_RENEW	4		/* no more state to renew */
#define NFS_CS_CHECK_LEASE_TIME	5		/* need to check lease time */
	unsigned long		cl_flags;	/* behavior switches */
#define NFS_CS_NORESVPORT	0		/* - use ephemeral src port */
#define NFS_CS_DISCRTRY		1		/* - disconnect on RPC retry */
#define NFS_CS_MIGRATION	2		/* - transparent state migr */
#define NFS_CS_INFINITE_SLOTS	3		/* - don't limit TCP slots */
#define NFS_CS_NO_RETRANS_TIMEOUT	4	/* - Disable retransmit timeouts */
#define NFS_CS_TSM_POSSIBLE	5		/* - Maybe state migration */
#define NFS_CS_NOPING		6		/* - don't ping on connect */
#define NFS_CS_DS		7		/* - Server is a DS */
#define NFS_CS_REUSEPORT	8		/* - reuse src port on reconnect */
	struct sockaddr_storage	cl_addr;	/* server identifier */
	size_t			cl_addrlen;
	char *			cl_hostname;	/* hostname of server */
	char *			cl_acceptor;	/* GSSAPI acceptor name */
	struct list_head	cl_share_link;	/* link in global client list */
	struct list_head	cl_superblocks;	/* List of nfs_server structs */

	struct rpc_clnt *	cl_rpcclient;
	const struct nfs_rpc_ops *rpc_ops;	/* NFS protocol vector */
	int			cl_proto;	/* Network transport protocol */
	struct nfs_subversion *	cl_nfs_mod;	/* pointer to nfs version module */

	u32			cl_minorversion;/* NFSv4 minorversion */
	unsigned int		cl_nconnect;	/* Number of connections */
	unsigned int		cl_max_connect; /* max number of xprts allowed */
	const char *		cl_principal;  /* used for machine cred */

#if IS_ENABLED(CONFIG_NFS_V4)
	struct list_head	cl_ds_clients; /* auth flavor data servers */
	u64			cl_clientid;	/* constant */
	nfs4_verifier		cl_confirm;	/* Clientid verifier */
	unsigned long		cl_state;

	spinlock_t		cl_lock;

	unsigned long		cl_lease_time;
	unsigned long		cl_last_renewal;
	struct delayed_work	cl_renewd;

	struct rpc_wait_queue	cl_rpcwaitq;

	/* idmapper */
	struct idmap *		cl_idmap;

	/* Client owner identifier */
	const char *		cl_owner_id;

	u32			cl_cb_ident;	/* v4.0 callback identifier */
	const struct nfs4_minor_version_ops *cl_mvops;
	unsigned long		cl_mig_gen;

	/* NFSv4.0 transport blocking */
	struct nfs4_slot_table	*cl_slot_tbl;

	/* The sequence id to use for the next CREATE_SESSION */
	u32			cl_seqid;
	/* The flags used for obtaining the clientid during EXCHANGE_ID */
	u32			cl_exchange_flags;
	struct nfs4_session	*cl_session;	/* shared session */
	bool			cl_preserve_clid;
	struct nfs41_server_owner *cl_serverowner;
	struct nfs41_server_scope *cl_serverscope;
	struct nfs41_impl_id	*cl_implid;
	/* nfs 4.1+ state protection modes: */
	unsigned long		cl_sp4_flags;
#define NFS_SP4_MACH_CRED_MINIMAL  1	/* Minimal sp4_mach_cred - state ops
					 * must use machine cred */
#define NFS_SP4_MACH_CRED_CLEANUP  2	/* CLOSE and LOCKU */
#define NFS_SP4_MACH_CRED_SECINFO  3	/* SECINFO and SECINFO_NO_NAME */
#define NFS_SP4_MACH_CRED_STATEID  4	/* TEST_STATEID and FREE_STATEID */
#define NFS_SP4_MACH_CRED_WRITE    5	/* WRITE */
#define NFS_SP4_MACH_CRED_COMMIT   6	/* COMMIT */
#define NFS_SP4_MACH_CRED_PNFS_CLEANUP  7 /* LAYOUTRETURN */
#if IS_ENABLED(CONFIG_NFS_V4_1)
	wait_queue_head_t	cl_lock_waitq;
#endif /* CONFIG_NFS_V4_1 */
#endif /* CONFIG_NFS_V4 */

	/* Our own IP address, as a null-terminated string.
	 * This is used to generate the mv0 callback address.
	 */
	char			cl_ipaddr[48];
	struct net		*cl_net;
	struct list_head	pending_cb_stateids;
};
```

# Inode

```c
struct nfs_inode {
	/*
	 * The 64bit 'inode number'
	 */
	__u64 fileid;

	/*
	 * NFS file handle
	 */
	struct nfs_fh		fh;

	/*
	 * Various flags
	 */
	unsigned long		flags;			/* atomic bit ops */
	unsigned long		cache_validity;		/* bit mask */

	/*
	 * read_cache_jiffies is when we started read-caching this inode.
	 * attrtimeo is for how long the cached information is assumed
	 * to be valid. A successful attribute revalidation doubles
	 * attrtimeo (up to acregmax/acdirmax), a failure resets it to
	 * acregmin/acdirmin.
	 *
	 * We need to revalidate the cached attrs for this inode if
	 *
	 *	jiffies - read_cache_jiffies >= attrtimeo
	 *
	 * Please note the comparison is greater than or equal
	 * so that zero timeout values can be specified.
	 */
	unsigned long		read_cache_jiffies;
	unsigned long		attrtimeo;
	unsigned long		attrtimeo_timestamp;

	unsigned long		attr_gencount;

	struct rb_root		access_cache;
	struct list_head	access_cache_entry_lru;
	struct list_head	access_cache_inode_lru;

	union {
		/* Directory */
		struct {
			/* "Generation counter" for the attribute cache.
			 * This is bumped whenever we update the metadata
			 * on the server.
			 */
			unsigned long	cache_change_attribute;
			/*
			 * This is the cookie verifier used for NFSv3 readdir
			 * operations
			 */
			__be32		cookieverf[NFS_DIR_VERIFIER_SIZE];
			/* Readers: in-flight sillydelete RPC calls */
			/* Writers: rmdir */
			struct rw_semaphore	rmdir_sem;
		};
		/* Regular file */
		struct {
			atomic_long_t	nrequests;
			struct nfs_mds_commit_info commit_info;
			struct mutex	commit_mutex;
		};
	};

	/* Open contexts for shared mmap writes */
	struct list_head	open_files;

#if IS_ENABLED(CONFIG_NFS_V4)
	struct nfs4_cached_acl	*nfs4_acl;
        /* NFSv4 state */
	struct list_head	open_states;
	struct nfs_delegation __rcu *delegation;
	struct rw_semaphore	rwsem;

	/* pNFS layout information */
	struct pnfs_layout_hdr *layout;
#endif /* CONFIG_NFS_V4*/
	/* how many bytes have been written/read and how many bytes queued up */
	__u64 write_io;
	__u64 read_io;
#ifdef CONFIG_NFS_FSCACHE
	struct fscache_cookie	*fscache;
#endif
	struct inode		vfs_inode;

#ifdef CONFIG_NFS_V4_2
	struct nfs4_xattr_cache *xattr_cache;
#endif
};
```

# nfs_fh

```c
struct nfs_fh {
	unsigned short		size;
	unsigned char		data[NFS_MAXFHSIZE];
};
```













