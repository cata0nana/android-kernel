# Linux Privilege Escalation

The end goal of this workshop is to use a **Android** kernel **vulnerability** to achieve privilege escalation i.e `root`. In **Linux** `root` is the super user with `uid=0(root) gid=0(root)` and has all the access rights.


## Light Weight Process {#light-weight-process}

Linux uses **Light Weight Process** to implement better support multi-threading. Each **light weight process** is assigned a process descriptor called `task_struct` and is defined in `include/linux/sched.h`.

```c
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
        /*
         * For reasons of header soup (see current_thread_info()), this
         * must be the first element of task_struct.
         */
        struct thread_info              thread_info;
#endif
        /* -1 unrunnable, 0 runnable, >0 stopped: */
        volatile long                   state;

        /*
         * This begins the randomizable portion of task_struct. Only
         * scheduling-critical items should be added above here.
         */
        randomized_struct_fields_start

        void                            *stack;
        atomic_t                        usage;
        /* Per task flags (PF_*), defined further below: */
        unsigned int                    flags;
        unsigned int                    ptrace;

#ifdef CONFIG_SMP
        struct llist_node               wake_entry;
        int                             on_cpu;
#ifdef CONFIG_THREAD_INFO_IN_TASK
        /* Current CPU: */
        unsigned int                    cpu;
#endif
        unsigned int                    wakee_flips;
        unsigned long                   wakee_flip_decay_ts;
        struct task_struct              *last_wakee;

        int                             wake_cpu;
#endif
        int                             on_rq;

        int                             prio;
        int                             static_prio;
        int                             normal_prio;
        unsigned int                    rt_priority;

        const struct sched_class        *sched_class;
        struct sched_entity             se;
        struct sched_rt_entity          rt;
#ifdef CONFIG_SCHED_WALT
        struct ravg ravg;
        /*
         * 'init_load_pct' represents the initial task load assigned to children
         * of this task
         */
        u32 init_load_pct;
        u64 last_sleep_ts;
#endif

#ifdef CONFIG_CGROUP_SCHED
        struct task_group               *sched_task_group;
#endif
        struct sched_dl_entity          dl;

#ifdef CONFIG_PREEMPT_NOTIFIERS
        /* List of struct preempt_notifier: */
        struct hlist_head               preempt_notifiers;
#endif

#ifdef CONFIG_BLK_DEV_IO_TRACE
        unsigned int                    btrace_seq;
#endif

        unsigned int                    policy;
        int                             nr_cpus_allowed;
        cpumask_t                       cpus_allowed;

#ifdef CONFIG_PREEMPT_RCU
        int                             rcu_read_lock_nesting;
        union rcu_special               rcu_read_unlock_special;
        struct list_head                rcu_node_entry;
        struct rcu_node                 *rcu_blocked_node;
#endif /* #ifdef CONFIG_PREEMPT_RCU */

#ifdef CONFIG_TASKS_RCU
        unsigned long                   rcu_tasks_nvcsw;
        u8                              rcu_tasks_holdout;
        u8                              rcu_tasks_idx;
        int                             rcu_tasks_idle_cpu;
        struct list_head                rcu_tasks_holdout_list;
#endif /* #ifdef CONFIG_TASKS_RCU */

        struct sched_info               sched_info;

        struct list_head                tasks;
#ifdef CONFIG_SMP
        struct plist_node               pushable_tasks;
        struct rb_node                  pushable_dl_tasks;
#endif

        struct mm_struct                *mm;
        struct mm_struct                *active_mm;

        /* Per-thread vma caching: */
        struct vmacache                 vmacache;

#ifdef SPLIT_RSS_COUNTING
        struct task_rss_stat            rss_stat;
#endif
        int                             exit_state;
        int                             exit_code;
        int                             exit_signal;
        /* The signal sent when the parent dies: */
        int                             pdeath_signal;
        /* JOBCTL_*, siglock protected: */
        unsigned long                   jobctl;

        /* Used for emulating ABI behavior of previous Linux versions: */
        unsigned int                    personality;

        /* Scheduler bits, serialized by scheduler locks: */
        unsigned                        sched_reset_on_fork:1;
        unsigned                        sched_contributes_to_load:1;
        unsigned                        sched_migrated:1;
        unsigned                        sched_remote_wakeup:1;
#ifdef CONFIG_PSI
        unsigned                        sched_psi_wake_requeue:1;
#endif

        /* Force alignment to the next boundary: */
        unsigned                        :0;

        /* Unserialized, strictly 'current' */

        /* Bit to tell LSMs we're in execve(): */
        unsigned                        in_execve:1;
        unsigned                        in_iowait:1;
#ifndef TIF_RESTORE_SIGMASK
        unsigned                        restore_sigmask:1;
#endif
#ifdef CONFIG_MEMCG
        unsigned                        memcg_may_oom:1;
#ifndef CONFIG_SLOB
        unsigned                        memcg_kmem_skip_account:1;
#endif
#endif
#ifdef CONFIG_COMPAT_BRK
        unsigned                        brk_randomized:1;
#endif
#ifdef CONFIG_CGROUPS
        /* disallow userland-initiated cgroup migration */
        unsigned                        no_cgroup_migration:1;
#endif

        unsigned long                   atomic_flags; /* Flags requiring atomic access. */

        struct restart_block            restart_block;

        pid_t                           pid;
        pid_t                           tgid;

#ifdef CONFIG_CC_STACKPROTECTOR
        /* Canary value for the -fstack-protector GCC feature: */
        unsigned long                   stack_canary;
#endif
        /*
         * Pointers to the (original) parent process, youngest child, younger sibling,
         * older sibling, respectively.  (p->father can be replaced with
         * p->real_parent->pid)
         */

        /* Real parent process: */
        struct task_struct __rcu        *real_parent;

        /* Recipient of SIGCHLD, wait4() reports: */
        struct task_struct __rcu        *parent;

        /*
         * Children/sibling form the list of natural children:
         */
        struct list_head                children;
        struct list_head                sibling;
        struct task_struct              *group_leader;

        /*
         * 'ptraced' is the list of tasks this task is using ptrace() on.
         *
         * This includes both natural children and PTRACE_ATTACH targets.
         * 'ptrace_entry' is this task's link on the p->parent->ptraced list.
         */
        struct list_head                ptraced;
        struct list_head                ptrace_entry;

        /* PID/PID hash table linkage. */
        struct pid_link                 pids[PIDTYPE_MAX];
        struct list_head                thread_group;
        struct list_head                thread_node;

        struct completion               *vfork_done;

        /* CLONE_CHILD_SETTID: */
        int __user                      *set_child_tid;

        /* CLONE_CHILD_CLEARTID: */
        int __user                      *clear_child_tid;

        u64                             utime;
        u64                             stime;
#ifdef CONFIG_ARCH_HAS_SCALED_CPUTIME
        u64                             utimescaled;
        u64                             stimescaled;
#endif
        u64                             gtime;
#ifdef CONFIG_CPU_FREQ_TIMES
        u64                             *time_in_state;
        unsigned int                    max_state;
#endif
        struct prev_cputime             prev_cputime;
#ifdef CONFIG_VIRT_CPU_ACCOUNTING_GEN
        struct vtime                    vtime;
#endif

#ifdef CONFIG_NO_HZ_FULL
        atomic_t                        tick_dep_mask;
#endif
        /* Context switch counts: */
        unsigned long                   nvcsw;
        unsigned long                   nivcsw;

        /* Monotonic time in nsecs: */
        u64                             start_time;

        /* Boot based time in nsecs: */
        u64                             real_start_time;

        /* MM fault and swap info: this can arguably be seen as either mm-specific or thread-specific: */
        unsigned long                   min_flt;
        unsigned long                   maj_flt;

#ifdef CONFIG_POSIX_TIMERS
        struct task_cputime             cputime_expires;
        struct list_head                cpu_timers[3];
#endif

        /* Process credentials: */

        /* Tracer's credentials at attach: */
        const struct cred __rcu         *ptracer_cred;

        /* Objective and real subjective task credentials (COW): */
        const struct cred __rcu         *real_cred;

        /* Effective (overridable) subjective task credentials (COW): */
        const struct cred __rcu         *cred;

        /*
         * executable name, excluding path.
         *
         * - normally initialized setup_new_exec()
         * - access it with [gs]et_task_comm()
         * - lock it with task_lock()
         */
        char                            comm[TASK_COMM_LEN];

        struct nameidata                *nameidata;

#ifdef CONFIG_SYSVIPC
        struct sysv_sem                 sysvsem;
        struct sysv_shm                 sysvshm;
#endif
#ifdef CONFIG_DETECT_HUNG_TASK
        unsigned long                   last_switch_count;
#endif
        /* Filesystem information: */
        struct fs_struct                *fs;

        /* Open file information: */
        struct files_struct             *files;

        /* Namespaces: */
        struct nsproxy                  *nsproxy;

        /* Signal handlers: */
        struct signal_struct            *signal;
        struct sighand_struct           *sighand;
        sigset_t                        blocked;
        sigset_t                        real_blocked;
        /* Restored if set_restore_sigmask() was used: */
        sigset_t                        saved_sigmask;
        struct sigpending               pending;
        unsigned long                   sas_ss_sp;
        size_t                          sas_ss_size;
        unsigned int                    sas_ss_flags;

        struct callback_head            *task_works;

        struct audit_context            *audit_context;
#ifdef CONFIG_AUDITSYSCALL
        kuid_t                          loginuid;
        unsigned int                    sessionid;
#endif
        struct seccomp                  seccomp;

        /* Thread group tracking: */
        u32                             parent_exec_id;
        u32                             self_exec_id;

        /* Protection against (de-)allocation: mm, files, fs, tty, keyrings, mems_allowed, mempolicy: */
        spinlock_t                      alloc_lock;

        /* Protection of the PI data structures: */
        raw_spinlock_t                  pi_lock;

        struct wake_q_node              wake_q;

#ifdef CONFIG_RT_MUTEXES
        /* PI waiters blocked on a rt_mutex held by this task: */
        struct rb_root_cached           pi_waiters;
        /* Updated under owner's pi_lock and rq lock */
        struct task_struct              *pi_top_task;
        /* Deadlock detection and priority inheritance handling: */
        struct rt_mutex_waiter          *pi_blocked_on;
#endif

#ifdef CONFIG_DEBUG_MUTEXES
        /* Mutex deadlock detection: */
        struct mutex_waiter             *blocked_on;
#endif

#ifdef CONFIG_TRACE_IRQFLAGS
        unsigned int                    irq_events;
        unsigned long                   hardirq_enable_ip;
        unsigned long                   hardirq_disable_ip;
        unsigned int                    hardirq_enable_event;
        unsigned int                    hardirq_disable_event;
        int                             hardirqs_enabled;
        int                             hardirq_context;
        unsigned long                   softirq_disable_ip;
        unsigned long                   softirq_enable_ip;
        unsigned int                    softirq_disable_event;
        unsigned int                    softirq_enable_event;
        int                             softirqs_enabled;
        int                             softirq_context;
#endif

#ifdef CONFIG_LOCKDEP
# define MAX_LOCK_DEPTH                 48UL
        u64                             curr_chain_key;
        int                             lockdep_depth;
        unsigned int                    lockdep_recursion;
        struct held_lock                held_locks[MAX_LOCK_DEPTH];
#endif

#ifdef CONFIG_LOCKDEP_CROSSRELEASE
#define MAX_XHLOCKS_NR 64UL
        struct hist_lock *xhlocks; /* Crossrelease history locks */
        unsigned int xhlock_idx;
        /* For restoring at history boundaries */
        unsigned int xhlock_idx_hist[XHLOCK_CTX_NR];
        unsigned int hist_id;
        /* For overwrite check at each context exit */
        unsigned int hist_id_save[XHLOCK_CTX_NR];
#endif

#ifdef CONFIG_UBSAN
        unsigned int                    in_ubsan;
#endif

        /* Journalling filesystem info: */
        void                            *journal_info;

        /* Stacked block device info: */
        struct bio_list                 *bio_list;

#ifdef CONFIG_BLOCK
        /* Stack plugging: */
        struct blk_plug                 *plug;
#endif

        /* VM state: */
        struct reclaim_state            *reclaim_state;

        struct backing_dev_info         *backing_dev_info;

        struct io_context               *io_context;

        /* Ptrace state: */
        unsigned long                   ptrace_message;
        siginfo_t                       *last_siginfo;

        struct task_io_accounting       ioac;
#ifdef CONFIG_PSI
        /* Pressure stall state */
        unsigned int                    psi_flags;
#endif
#ifdef CONFIG_TASK_XACCT
        /* Accumulated RSS usage: */
        u64                             acct_rss_mem1;
        /* Accumulated virtual memory usage: */
        u64                             acct_vm_mem1;
        /* stime + utime since last update: */
        u64                             acct_timexpd;
#endif
#ifdef CONFIG_CPUSETS
        /* Protected by ->alloc_lock: */
        nodemask_t                      mems_allowed;
        /* Seqence number to catch updates: */
        seqcount_t                      mems_allowed_seq;
        int                             cpuset_mem_spread_rotor;
        int                             cpuset_slab_spread_rotor;
#endif
#ifdef CONFIG_CGROUPS
        /* Control Group info protected by css_set_lock: */
        struct css_set __rcu            *cgroups;
        /* cg_list protected by css_set_lock and tsk->alloc_lock: */
        struct list_head                cg_list;
#endif
#ifdef CONFIG_INTEL_RDT
        u32                             closid;
        u32                             rmid;
#endif
#ifdef CONFIG_FUTEX
        struct robust_list_head __user  *robust_list;
#ifdef CONFIG_COMPAT
        struct compat_robust_list_head __user *compat_robust_list;
#endif
        struct list_head                pi_state_list;
        struct futex_pi_state           *pi_state_cache;
#endif
#ifdef CONFIG_PERF_EVENTS
        struct perf_event_context       *perf_event_ctxp[perf_nr_task_contexts];
        struct mutex                    perf_event_mutex;
        struct list_head                perf_event_list;
#endif
#ifdef CONFIG_DEBUG_PREEMPT
        unsigned long                   preempt_disable_ip;
#endif
#ifdef CONFIG_NUMA
        /* Protected by alloc_lock: */
        struct mempolicy                *mempolicy;
        short                           il_prev;
        short                           pref_node_fork;
#endif
#ifdef CONFIG_NUMA_BALANCING
        int                             numa_scan_seq;
        unsigned int                    numa_scan_period;
        unsigned int                    numa_scan_period_max;
        int                             numa_preferred_nid;
        unsigned long                   numa_migrate_retry;
        /* Migration stamp: */
        u64                             node_stamp;
        u64                             last_task_numa_placement;
        u64                             last_sum_exec_runtime;
        struct callback_head            numa_work;

        struct list_head                numa_entry;
        struct numa_group               *numa_group;

        /*
         * numa_faults is an array split into four regions:
         * faults_memory, faults_cpu, faults_memory_buffer, faults_cpu_buffer
         * in this precise order.
         *
         * faults_memory: Exponential decaying average of faults on a per-node
         * basis. Scheduling placement decisions are made based on these
         * counts. The values remain static for the duration of a PTE scan.
         * faults_cpu: Track the nodes the process was running on when a NUMA
         * hinting fault was incurred.
         * faults_memory_buffer and faults_cpu_buffer: Record faults per node
         * during the current scan window. When the scan completes, the counts
         * in faults_memory and faults_cpu decay and these values are copied.
         */
        unsigned long                   *numa_faults;
        unsigned long                   total_numa_faults;

        /*
         * numa_faults_locality tracks if faults recorded during the last
         * scan window were remote/local or failed to migrate. The task scan
         * period is adapted based on the locality of the faults with different
         * weights depending on whether they were shared or private faults
         */
        unsigned long                   numa_faults_locality[3];

        unsigned long                   numa_pages_migrated;
#endif /* CONFIG_NUMA_BALANCING */

        struct tlbflush_unmap_batch     tlb_ubc;

        struct rcu_head                 rcu;

        /* Cache last used pipe for splice(): */
        struct pipe_inode_info          *splice_pipe;

        struct page_frag                task_frag;

#ifdef CONFIG_TASK_DELAY_ACCT
        struct task_delay_info          *delays;
#endif

#ifdef CONFIG_FAULT_INJECTION
        int                             make_it_fail;
        unsigned int                    fail_nth;
#endif
        /*
         * When (nr_dirtied >= nr_dirtied_pause), it's time to call
         * balance_dirty_pages() for a dirty throttling pause:
         */
        int                             nr_dirtied;
        int                             nr_dirtied_pause;
        /* Start of a write-and-pause period: */
        unsigned long                   dirty_paused_when;

#ifdef CONFIG_LATENCYTOP
        int                             latency_record_count;
        struct latency_record           latency_record[LT_SAVECOUNT];
#endif
        /*
         * Time slack values; these are used to round up poll() and
         * select() etc timeout values. These are in nanoseconds.
         */
        u64                             timer_slack_ns;
        u64                             default_timer_slack_ns;

#ifdef CONFIG_KASAN
        unsigned int                    kasan_depth;
#endif

#ifdef CONFIG_FUNCTION_GRAPH_TRACER
        /* Index of current stored address in ret_stack: */
        int                             curr_ret_stack;

        /* Stack of return addresses for return function tracing: */
        struct ftrace_ret_stack         *ret_stack;

        /* Timestamp for last schedule: */
        unsigned long long              ftrace_timestamp;

        /*
         * Number of functions that haven't been traced
         * because of depth overrun:
         */
        atomic_t                        trace_overrun;

        /* Pause tracing: */
        atomic_t                        tracing_graph_pause;
#endif

#ifdef CONFIG_TRACING
        /* State flags for use by tracers: */
        unsigned long                   trace;

        /* Bitmask and counter of trace recursion: */
        unsigned long                   trace_recursion;
#endif /* CONFIG_TRACING */

#ifdef CONFIG_KCOV
        /* Coverage collection mode enabled for this task (0 if disabled): */
        enum kcov_mode                  kcov_mode;

        /* Size of the kcov_area: */
        unsigned int                    kcov_size;

        /* Buffer for coverage collection: */
        void                            *kcov_area;

        /* KCOV descriptor wired with this task or NULL: */
        struct kcov                     *kcov;
#endif

#ifdef CONFIG_MEMCG
        struct mem_cgroup               *memcg_in_oom;
        gfp_t                           memcg_oom_gfp_mask;
        int                             memcg_oom_order;

        /* Number of pages to reclaim on returning to userland: */
        unsigned int                    memcg_nr_pages_over_high;
#endif

#ifdef CONFIG_UPROBES
        struct uprobe_task              *utask;
#endif
#if defined(CONFIG_BCACHE) || defined(CONFIG_BCACHE_MODULE)
        unsigned int                    sequential_io;
        unsigned int                    sequential_io_avg;
#endif
#ifdef CONFIG_DEBUG_ATOMIC_SLEEP
        unsigned long                   task_state_change;
#endif
        int                             pagefault_disabled;
#ifdef CONFIG_MMU
        struct task_struct              *oom_reaper_list;
#endif
#ifdef CONFIG_VMAP_STACK
        struct vm_struct                *stack_vm_area;
#endif
#ifdef CONFIG_THREAD_INFO_IN_TASK
        /* A live task holds one reference: */
        atomic_t                        stack_refcount;
#endif
#ifdef CONFIG_LIVEPATCH
        int patch_state;
#endif
#ifdef CONFIG_SECURITY
        /* Used by LSM modules for access restriction: */
        void                            *security;
#endif

        /*
         * New fields for task_struct should be added above here, so that
         * they are included in the randomized portion of task_struct.
         */
        randomized_struct_fields_end

        /* CPU-specific state of this task: */
        struct thread_struct            thread;

        /*
         * WARNING: on x86, 'thread_struct' contains a variable-sized
         * structure.  It *MUST* be at the end of 'task_struct'.
         *
         * Do not put anything below here!
         */
};
```

This data structure contains all the information to manage a process. One of the interesting members in this `task_struct` structure is `cred`.


## Process Credentials {#process-credentials}

The **security context** of a task is defined by `struct cred` and is defined in `include/linux/cred.h`.

```c
struct cred {
        atomic_t        usage;
#ifdef CONFIG_DEBUG_CREDENTIALS
        atomic_t        subscribers;    /* number of processes subscribed */
        void            *put_addr;
        unsigned        magic;
#define CRED_MAGIC      0x43736564
#define CRED_MAGIC_DEAD 0x44656144
#endif
        kuid_t          uid;            /* real UID of the task */
        kgid_t          gid;            /* real GID of the task */
        kuid_t          suid;           /* saved UID of the task */
        kgid_t          sgid;           /* saved GID of the task */
        kuid_t          euid;           /* effective UID of the task */
        kgid_t          egid;           /* effective GID of the task */
        kuid_t          fsuid;          /* UID for VFS ops */
        kgid_t          fsgid;          /* GID for VFS ops */
        unsigned        securebits;     /* SUID-less security management */
        kernel_cap_t    cap_inheritable; /* caps our children can inherit */
        kernel_cap_t    cap_permitted;  /* caps we're permitted */
        kernel_cap_t    cap_effective;  /* caps we can actually use */
        kernel_cap_t    cap_bset;       /* capability bounding set */
        kernel_cap_t    cap_ambient;    /* Ambient capability set */
#ifdef CONFIG_KEYS
        unsigned char   jit_keyring;    /* default keyring to attach requested
                                         * keys to */
        struct key __rcu *session_keyring; /* keyring inherited over fork */
        struct key      *process_keyring; /* keyring private to this process */
        struct key      *thread_keyring; /* keyring private to this thread */
        struct key      *request_key_auth; /* assumed request_key authority */
#endif
#ifdef CONFIG_SECURITY
        void            *security;      /* subjective LSM security */
#endif
        struct user_struct *user;       /* real user ID subscription */
        struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
        struct group_info *group_info;  /* supplementary groups for euid/fsgid */
        /* RCU deletion */
        union {
                int non_rcu;                    /* Can we skip RCU deletion? */
                struct rcu_head rcu;            /* RCU deletion hook */
        };
} __randomize_layout;
```

In most of the Linux **kernel exploits**, you must have seen that to achieve `root` they use 

```c
commit_creds(prepare_kernel_cred(NULL));
```

Let's try to look into these two functions and see what they do. First, let's look into `prepare_kernel_cred` function which is defined in `kernel/cred.c`.

```c
struct cred *prepare_kernel_cred(struct task_struct *daemon)
{
        const struct cred *old;
        struct cred *new;

        new = kmem_cache_alloc(cred_jar, GFP_KERNEL);
        if (!new)
                return NULL;

        kdebug("prepare_kernel_cred() alloc %p", new);

        if (daemon)
                old = get_task_cred(daemon);
        else
                old = get_cred(&init_cred);

        validate_creds(old);

        *new = *old;
        [...]
        validate_creds(new);
        return new;

error:
        [...]
        return NULL;
}
```

This function basically take a pointer `task_struct` for which we want to prepare *kernel credentials*. The important part of the function is that if we provide `NULL` as the pointer to `task_struct` it will get the default credentials which is `init_cred`. `init_cred` is a global `struct cred` defined in `kernel/cred.c` which is used to initialize the credentials for the `init_task` which is the first **task** in Linux.

```c
/*
 * The initial credentials for the initial task
 */
struct cred init_cred = {
        .usage                  = ATOMIC_INIT(4),
#ifdef CONFIG_DEBUG_CREDENTIALS
        .subscribers            = ATOMIC_INIT(2),
        .magic                  = CRED_MAGIC,
#endif
        .uid                    = GLOBAL_ROOT_UID,
        .gid                    = GLOBAL_ROOT_GID,
        .suid                   = GLOBAL_ROOT_UID,
        .sgid                   = GLOBAL_ROOT_GID,
        .euid                   = GLOBAL_ROOT_UID,
        .egid                   = GLOBAL_ROOT_GID,
        .fsuid                  = GLOBAL_ROOT_UID,
        .fsgid                  = GLOBAL_ROOT_GID,
        .securebits             = SECUREBITS_DEFAULT,
        .cap_inheritable        = CAP_EMPTY_SET,
        .cap_permitted          = CAP_FULL_SET,
        .cap_effective          = CAP_FULL_SET,
        .cap_bset               = CAP_FULL_SET,
        .user                   = INIT_USER,
        .user_ns                = &init_user_ns,
        .group_info             = &init_groups,
};
```

Let's look at what these defines mean.

```c
#define GLOBAL_ROOT_UID     (uint32_t)0
#define GLOBAL_ROOT_GID     (uint32_t)0
#define SECUREBITS_DEFAULT  (uint32_t)0x00000000
#define CAP_EMPTY_SET       (uint64_t)0
#define CAP_FULL_SET        (uint64_t)0x3FFFFFFFFF
```

`init_cred` basically sets the `cred` structure as shown below.

```c
cred->uid = 0;
cred->gid = 0;
cred->suid = 0;
cred->idid = 0;
cred->euid = 0;
cred->egid = 0;
cred->fsuid = 0;
cred->fsgid = 0;
cred->securebits = 0;
cred->cap_inheritable.cap[0] = 0;
cred->cap_inheritable.cap[1] = 0;
cred->cap_permitted.cap[0] = 0x3F;
cred->cap_permitted.cap[1] = 0xFFFFFFFF;
cred->cap_effective.cap[0] = 0x3F;
cred->cap_effective.cap[1] = 0xFFFFFFFF;
cred->cap_bset.cap[0] = 0x3F;
cred->cap_bset.cap[1] = 0xFFFFFFFF;
cred->cap_ambient.cap[0] = 0;
cred->cap_ambient.cap[1] = 0;
```

Let's look at the `commit_creds` function and try to understand what it does.

```c
int commit_creds(struct cred *new)
{
        struct task_struct *task = current;
        const struct cred *old = task->real_cred;

        [...]

        rcu_assign_pointer(task->real_cred, new);
        rcu_assign_pointer(task->cred, new);
        
        [...]

        return 0;
}
```

`commit_creds` basically sets the `task->real_cred` and `task->cred` with the pointer to new `cred` structure. However, as we had passed `NULL` to `prepare_kernel_cred` address of `init_cred`.

This is how we get `root` and this basically means **privilege escalation**


## SELinux {#selinux}

**Security-Enhanced Linux** was developed by **National Security Agency (NSA)** using **Linux Security Modules (LSM)**.

There are two modes of **SELinux**

* **permissive** - permission denials are *logged* but not *enforced*
* **enforcing** - permission denials are *logged* and *enforced*

In **Android** the default mode of **SELinux** is **enforcing** and even if we get **root**, we are subjected to **SELinux** rules. 

```bash
generic_x86_64:/ $ getenforce                                                                                      
Enforcing
```

So, we need to disable **SELinux** as well.


### selinux_enforcing {#selinux-enforcing}

`selinux_enforcing` is a global variable which dictates whether **SELinux** is **enforced** or **not**. If we can figure out where `selinux_enforcing` is in memory and set it to `NULL`, then we can disable **SELinux** globally and now **SELinux** will be in **permissive** mode instead of **enforcing** mode.


## SecComp {#seccomp}

**SecComp** stands for **Secure Computing** mode and is a Linux kernel feature that allows to **filter system calls**. When enabled, the process can only make **four** system calls `read()`, `write()`, `exit()`, and `sigreturn()`.

When running the **exploit** from `adb` shell we are not subjected to **seccomp**. However, if we bundle the **exploit** in an **Android** application, we would be subjected to **seccomp**.

In this workshop, we are not going to look at **seccomp**.
