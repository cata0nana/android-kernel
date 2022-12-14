# Root Cause Analysis

**Root Cause Analysis (RCA)** is a very important part of **vulnerability research**. With **RCA** we can determine if a **crash** or **bug** can be **exploited**.

**RCA** is basically reverse engineering process to understanding the code that lead to the crash.


## Revisiting Crash {#revisiting-crash}

From the crash log, we already know that it's **Use after Free** vulnerability. Let's revisit the crash report and try to understand why it occurred.

Let's strip away unwanted information and break the crash log in three parts, allocation, free and use


### Allocation {#revisiting-crash-allocation}

```
[<        none        >] save_stack_trace+0x16/0x18 arch/x86/kernel/stacktrace.c:59
[<     inline     >] save_stack mm/kasan/common.c:76
[<     inline     >] set_track mm/kasan/common.c:85
[<        none        >] __kasan_kmalloc+0x133/0x1cc mm/kasan/common.c:501
[<        none        >] kasan_kmalloc+0x9/0xb mm/kasan/common.c:515
[<        none        >] kmem_cache_alloc_trace+0x1bd/0x26f mm/slub.c:2819
[<     inline     >] kmalloc include/linux/slab.h:488
[<     inline     >] kzalloc include/linux/slab.h:661
[<        none        >] binder_get_thread+0x166/0x6db drivers/android/binder.c:4677
[<        none        >] binder_poll+0x4c/0x1c2 drivers/android/binder.c:4805
[<     inline     >] ep_item_poll fs/eventpoll.c:888
[<     inline     >] ep_insert fs/eventpoll.c:1476
[<     inline     >] SYSC_epoll_ctl fs/eventpoll.c:2128
[<        none        >] SyS_epoll_ctl+0x1558/0x24f0 fs/eventpoll.c:2014
[<        none        >] do_syscall_64+0x19e/0x225 arch/x86/entry/common.c:292
[<        none        >] entry_SYSCALL_64_after_hwframe+0x3d/0xa2 arch/x86/entry/entry_64.S:233
```

Here is the simplified call graph.

<p align="center">
  <img src="../images/crash-log-allocation-stack-trace.png" alt="Allocation Stack Trace" title="Allocation Stack Trace"/>
</p>

Relevant source line from the PoC

```c
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
```


### Free {#revisiting-crash-free}

```
[<        none        >] save_stack_trace+0x16/0x18 arch/x86/kernel/stacktrace.c:59
[<     inline     >] save_stack mm/kasan/common.c:76
[<     inline     >] set_track mm/kasan/common.c:85
[<        none        >] __kasan_slab_free+0x18f/0x23f mm/kasan/common.c:463
[<        none        >] kasan_slab_free+0xe/0x10 mm/kasan/common.c:471
[<     inline     >] slab_free_hook mm/slub.c:1407
[<     inline     >] slab_free_freelist_hook mm/slub.c:1458
[<     inline     >] slab_free mm/slub.c:3039
[<        none        >] kfree+0x193/0x5b3 mm/slub.c:3976
[<     inline     >] binder_free_thread drivers/android/binder.c:4705
[<        none        >] binder_thread_dec_tmpref+0x192/0x1d9 drivers/android/binder.c:2053
[<        none        >] binder_thread_release+0x464/0x4bd drivers/android/binder.c:4794
[<        none        >] binder_ioctl+0x48a/0x101c drivers/android/binder.c:5062
[<        none        >] do_vfs_ioctl+0x608/0x106a fs/ioctl.c:46
[<     inline     >] SYSC_ioctl fs/ioctl.c:701
[<        none        >] SyS_ioctl+0x75/0xa4 fs/ioctl.c:692
[<        none        >] do_syscall_64+0x19e/0x225 arch/x86/entry/common.c:292
[<        none        >] entry_SYSCALL_64_after_hwframe+0x3d/0xa2 arch/x86/entry/entry_64.S:233
```

Here is the simplified call graph.

<p align="center">
  <img src="../images/crash-log-free-stack-trace.png" alt="Free Stack Trace" title="Free Stack Trace"/>
</p>

Relevant source line from the PoC

```c
ioctl(fd, BINDER_THREAD_EXIT, NULL);
```

Let's look at the the `binder_free_thread` implementation in `workshop/android-4.14-dev/goldfish/drivers/android/binder.c`.

```c
static void binder_free_thread(struct binder_thread *thread)
{
        [...]
        kfree(thread);
}
```

We see that `binder_thread` structure is being **freed** by calling `kfree` which exactly matches the free call trace. This confirms that the **dangling** chunk is `binder_thread` structure.

Let's see how `struct binder_thread` is defined.

```c
struct binder_thread {
        struct binder_proc *proc;
        struct rb_node rb_node;
        struct list_head waiting_thread_node;
        int pid;
        int looper;              /* only modified by this thread */
        bool looper_need_return; /* can be written by other thread */
        struct binder_transaction *transaction_stack;
        struct list_head todo;
        bool process_todo;
        struct binder_error return_error;
        struct binder_error reply_error;
        wait_queue_head_t wait;
        struct binder_stats stats;
        atomic_t tmp_ref;
        bool is_dead;
        struct task_struct *task;
};
```


### Use {#revisiting-crash-use}

```
[<        none        >] _raw_spin_lock_irqsave+0x3a/0x5d kernel/locking/spinlock.c:160
[<        none        >] remove_wait_queue+0x27/0x122 kernel/sched/wait.c:50
 ?[<        none        >] fsnotify_unmount_inodes+0x1e8/0x1e8 fs/notify/fsnotify.c:99
[<     inline     >] ep_remove_wait_queue fs/eventpoll.c:612
[<        none        >] ep_unregister_pollwait+0x160/0x1bd fs/eventpoll.c:630
[<        none        >] ep_free+0x8b/0x181 fs/eventpoll.c:847
 ?[<        none        >] ep_eventpoll_poll+0x228/0x228 fs/eventpoll.c:942
[<        none        >] ep_eventpoll_release+0x48/0x54 fs/eventpoll.c:879
[<        none        >] __fput+0x1f2/0x51d fs/file_table.c:210
[<        none        >] ____fput+0x15/0x18 fs/file_table.c:244
[<        none        >] task_work_run+0x127/0x154 kernel/task_work.c:113
[<     inline     >] exit_task_work include/linux/task_work.h:22
[<        none        >] do_exit+0x818/0x2384 kernel/exit.c:875
 ?[<        none        >] mm_update_next_owner+0x52f/0x52f kernel/exit.c:468
[<        none        >] do_group_exit+0x12c/0x24b kernel/exit.c:978
 ?[<     inline     >] spin_unlock_irq include/linux/spinlock.h:367
 ?[<        none        >] do_group_exit+0x24b/0x24b kernel/exit.c:975
[<        none        >] SYSC_exit_group+0x17/0x17 kernel/exit.c:989
[<        none        >] SyS_exit_group+0x14/0x14 kernel/exit.c:987
[<        none        >] do_syscall_64+0x19e/0x225 arch/x86/entry/common.c:292
[<        none        >] entry_SYSCALL_64_after_hwframe+0x3d/0xa2 arch/x86/entry/entry_64.S:233
```

Here is the simplified call graph.

<p align="center">
  <img src="../images/crash-log-use-stack-trace.png" alt="Free Stack Trace" title="Free Stack Trace"/>
</p>

We don't see any line in the PoC which calls `SyS_exit_group`. It turns out that the **use** happens when the process exits, and eventually `exit_group` system call is called. This is when it tries to cleanup the resources and uses the **dangling** chunk erroneously.


## Visual Studio Code {#visual-studio-code}

We will use **Visual Studio Code** for **Android** kernel source code **navigation**. I used this project https://github.com/amezin/vscode-linux-kernel for better **intellisense** support. 


## Static Analysis {#static-analysis}

We already know that `binder_thread` is the **dangling** chunk. Let's statically trace the function calls in the crashing PoC and see what's happening.

We want to answer the following questions:

* Why `binder_thread` structure was **allocated**?
* Why `binder_thread` structure was **freed**?
* Why the use of `binder_thread` structure happened when it's already **freed**?


### open {#syscall-open}

```c
fd = open("/dev/binder", O_RDONLY);
```

Let's open `workshop/android-4.14-dev/goldfish/drivers/android/binder.c` and see how `open` system call is implemented.

```c
static const struct file_operations binder_fops = {
	[...]
	.open = binder_open,
	[...]
};
```

We see that `open` system call is handled by `binder_open` function.


Let's follow `binder_open` function and find out what it does.

```c
static int binder_open(struct inode *nodp, struct file *filp)
{
        struct binder_proc *proc;
        [...]
        proc = kzalloc(sizeof(*proc), GFP_KERNEL);
        if (proc == NULL)
                return -ENOMEM;
        [...]
        filp->private_data = proc;
        [...]
        return 0;
}
```

`binder_open` allocates `binder_proc` data structure and assigns it to the `filp->private_data`.


### epoll_create {#syscall-epoll-create}

```c
epfd = epoll_create(1000);
```

Let's open `workshop/android-4.14-dev/goldfish/fs/eventpoll.c` and see how `epoll_create` system call is implemented. We will also follow the call graph and look into all the important functions that `epoll_create` will call.

```c
SYSCALL_DEFINE1(epoll_create, int, size)
{
        if (size <= 0)
                return -EINVAL;

        return sys_epoll_create1(0);
}
```

`epoll_create` checks if `size <= 0` and then calls `sys_epoll_create1`. We can see that `1000` passed as parameter does not have any specific implications. The `size` parameter should be greater than `0`.


Let's follow the `sys_epoll_create1` function.

```c
SYSCALL_DEFINE1(epoll_create1, int, flags)
{
        int error, fd;
        struct eventpoll *ep = NULL;
        struct file *file;
        [...]
        error = ep_alloc(&ep);
        if (error < 0)
                return error;
        [...]
        file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
                                 O_RDWR | (flags & O_CLOEXEC));
        [...]
        ep->file = file;
        fd_install(fd, file);
        return fd;
        [...]
        return error;
}
```

`epoll_create1` calls `ep_alloc`, sets `ep->file = file` and finally returns the **epoll** file descriptor `fd`.


Let's follow `ep_alloc` function and find out what it does.

```c
static int ep_alloc(struct eventpoll **pep)
{
        int error;
        struct user_struct *user;
        struct eventpoll *ep;
        [...]
        ep = kzalloc(sizeof(*ep), GFP_KERNEL);
        [...]
        init_waitqueue_head(&ep->wq);
        init_waitqueue_head(&ep->poll_wait);
        INIT_LIST_HEAD(&ep->rdllist);
        ep->rbr = RB_ROOT_CACHED;
        [...]
        *pep = ep;
        return 0;
        [...]
        return error;
}
```

* allocates `struct eventpoll`, initializes **wait queues** `wq` and `poll_wait` members
* initializes `rbr` member which is the **red black tree** root


`struct eventpoll` is the main data structure used by **event polling** subsystem. Let's see how `eventpoll` structure is defined in `workshop/android-4.14-dev/goldfish/fs/eventpoll.c`.

```c
struct eventpoll {
        /* Protect the access to this structure */
        spinlock_t lock;

        /*
         * This mutex is used to ensure that files are not removed
         * while epoll is using them. This is held during the event
         * collection loop, the file cleanup path, the epoll file exit
         * code and the ctl operations.
         */
        struct mutex mtx;

        /* Wait queue used by sys_epoll_wait() */
        wait_queue_head_t wq;

        /* Wait queue used by file->poll() */
        wait_queue_head_t poll_wait;

        /* List of ready file descriptors */
        struct list_head rdllist;

        /* RB tree root used to store monitored fd structs */
        struct rb_root_cached rbr;

        /*
         * This is a single linked list that chains all the "struct epitem" that
         * happened while transferring ready events to userspace w/out
         * holding ->lock.
         */
        struct epitem *ovflist;

        /* wakeup_source used when ep_scan_ready_list is running */
        struct wakeup_source *ws;

        /* The user that created the eventpoll descriptor */
        struct user_struct *user;

        struct file *file;

        /* used to optimize loop detection check */
        int visited;
        struct list_head visited_list_link;

#ifdef CONFIG_NET_RX_BUSY_POLL
        /* used to track busy poll napi_id */
        unsigned int napi_id;
#endif
};
```


### epoll_ctl {#syscall-epoll-ctl}

```c
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &event);
```

Let's open `workshop/android-4.14-dev/goldfish/fs/eventpoll.c` and see how `epoll_ctl` is implemented. We are passing `EPOLL_CTL_ADD` as the operation parameter.

```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
                struct epoll_event __user *, event)
{
        int error;
        int full_check = 0;
        struct fd f, tf;
        struct eventpoll *ep;
        struct epitem *epi;
        struct epoll_event epds;
        struct eventpoll *tep = NULL;

        error = -EFAULT;
        if (ep_op_has_event(op) &&
            copy_from_user(&epds, event, sizeof(struct epoll_event)))
                goto error_return;

        error = -EBADF;
        f = fdget(epfd);
        if (!f.file)
                goto error_return;

        /* Get the "struct file *" for the target file */
        tf = fdget(fd);
        if (!tf.file)
                goto error_fput;
        [...]
        ep = f.file->private_data;
        [...]
        epi = ep_find(ep, tf.file, fd);

        error = -EINVAL;
        switch (op) {
        case EPOLL_CTL_ADD:
                if (!epi) {
                        epds.events |= POLLERR | POLLHUP;
                        error = ep_insert(ep, &epds, tf.file, fd, full_check);
                } else
                        error = -EEXIST;
                [...]
        [...]
        }
        [...]
        return error;
}
```

* copies `epoll_event` structure from **user space** to **kernel space**
* finds the corresponding `file` pointers of `epfd` and `fd` file descriptors
* gets the pointer to `eventpoll` structure from the `private_data` member of the `file` pointer of the epoll file descriptor `epfd`
* calls `ep_find` to find the pointer to linked `epitem` structure from the **red black tree** node stored in `eventpoll` structure matching the file descriptor `fd`
* if `epitem` is not found for the corresponding `fd`, then it calls `ep_insert` function to allocate and link a `epitem` to `eventpoll` structure's `rbr` member


Let's see how `struct epitem` is defined.

```c
struct epitem {
        union {
                /* RB tree node links this structure to the eventpoll RB tree */
                struct rb_node rbn;
                /* Used to free the struct epitem */
                struct rcu_head rcu;
        };

        /* List header used to link this structure to the eventpoll ready list */
        struct list_head rdllink;

        /*
         * Works together "struct eventpoll"->ovflist in keeping the
         * single linked chain of items.
         */
        struct epitem *next;

        /* The file descriptor information this item refers to */
        struct epoll_filefd ffd;

        /* Number of active wait queue attached to poll operations */
        int nwait;

        /* List containing poll wait queues */
        struct list_head pwqlist;

        /* The "container" of this item */
        struct eventpoll *ep;

        /* List header used to link this item to the "struct file" items list */
        struct list_head fllink;

        /* wakeup_source used when EPOLLWAKEUP is set */
        struct wakeup_source __rcu *ws;

        /* The structure that describe the interested events and the source fd */
        struct epoll_event event;
};
```

Below given diagram shows how an `epitem` structure is linked to `eventpoll` structure.

<p align="center">
  <img src="../images/epitem-eventpoll-link.png" alt="epitem eventpoll link" title="epitem eventpoll link"/>
</p>


Let's follow `ep_insert` function and see what it exactly does.

```c
static int ep_insert(struct eventpoll *ep, struct epoll_event *event,
                     struct file *tfile, int fd, int full_check)
{
        int error, revents, pwake = 0;
        unsigned long flags;
        long user_watches;
        struct epitem *epi;
        struct ep_pqueue epq;
        [...]
        if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
                return -ENOMEM;

        /* Item initialization follow here ... */
        INIT_LIST_HEAD(&epi->rdllink);
        INIT_LIST_HEAD(&epi->fllink);
        INIT_LIST_HEAD(&epi->pwqlist);
        epi->ep = ep;
        ep_set_ffd(&epi->ffd, tfile, fd);
        epi->event = *event;
        [...]

        /* Initialize the poll table using the queue callback */
        epq.epi = epi;
        init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
        [...]
        revents = ep_item_poll(epi, &epq.pt);
        [...]
        ep_rbtree_insert(ep, epi);
        [...]
        return 0;
        [...]
        return error;
}
```

* allocates a temporary structure `ep_pqueue`
* allocates `epitem` structure and initializes it
* initializes `epi->pwqlist` member which is used to link the **poll wait queues**
* sets the `epitem` structure member `ffd->file = file` and `ffd->fd = fd` which is the binder's `file` structure pointer and descriptor in our case by calling `ep_set_ffd`
* sets `epq.epi` to `epi` pointer
* sets `epq.pt->_qproc` to `ep_ptable_queue_proc` **callback** address
* calls `ep_item_poll` passing `epi` and address of `epq.pt` (poll table) as arguments
* finally, links `epitem` structure to `eventpoll` structure's **red black tree** root node by calling `ep_rbtree_insert` function


Let's follow `ep_item_poll` and find out what it does.

```c
static inline unsigned int ep_item_poll(struct epitem *epi, poll_table *pt)
{
        pt->_key = epi->event.events;

        return epi->ffd.file->f_op->poll(epi->ffd.file, pt) & epi->event.events;
}
```

* calls `poll` function in the binder's `file` structure `f_op->poll` passing binder's `file` structure pointer and `poll_table` pointer


> **Note:** Now, we are jumping to **binder** subsystem from **epoll** subsystem.


Let's open `workshop/android-4.14-dev/goldfish/drivers/android/binder.c` and see how `poll` system call is implemented.

```c
static const struct file_operations binder_fops = {
	[...]
	.poll = binder_poll,
	[...]
};
```

We see that `poll` system call is handled by `binder_poll` function.


Let's follow `binder_poll` function and find out what it does.

```c
static unsigned int binder_poll(struct file *filp,
                                struct poll_table_struct *wait)
{
        struct binder_proc *proc = filp->private_data;
        struct binder_thread *thread = NULL;
        [...]
        thread = binder_get_thread(proc);
        if (!thread)
                return POLLERR;
        [...]
        poll_wait(filp, &thread->wait, wait);
        [...]
        return 0;
}
```

* gets the pointer to `binder_proc` structure from `filp->private_data`
* calls `binder_get_thread` passing `binder_proc` structure pointer
* finally calls `poll_wait` passing binder's `file` structure pointer, `&thread->wait` which is `wait_queue_head_t` pointer and `poll_table_struct` pointer


Let's first follow `binder_get_thread` and find out what it does. After that we will follow `poll_wait` function.

```c
static struct binder_thread *binder_get_thread(struct binder_proc *proc)
{
        struct binder_thread *thread;
        struct binder_thread *new_thread;
        [...]
        thread = binder_get_thread_ilocked(proc, NULL);
        [...]
        if (!thread) {
                new_thread = kzalloc(sizeof(*thread), GFP_KERNEL);
                [...]
                thread = binder_get_thread_ilocked(proc, new_thread);
                [...]
        }
        return thread;
}
```

* tries to get the `binder_thread` if present in `proc->threads.rb_node` by calling `binder_get_thread_ilocked`
* else it allocates a `binder_thread` structure
* finally calls `binder_get_thread_ilocked` again, which initializes the newly allocated `binder_thread` structure and link it to the `proc->threads.rb_node` member which is basically a **red black tree** node

If you see the call graph in **[Allocation](root-cause-analysis.md#revisiting-crash-allocation)** section, you will find that this is where the `binder_thread` structure is **allocated**.


Now, let's follow `poll_wait` function and find out what it does.

```c
static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
{
        if (p && p->_qproc && wait_address)
                p->_qproc(filp, wait_address, p);
}
```

* calls the **callback** function assigned to `p->_qproc` passing binder's `file` structure pointer, `wait_queue_head_t` pointer and `poll_table` pointer

If you go up and see `ep_insert` function, you will see that `p->_qproc` was set to `ep_ptable_queue_proc` function's address.


> **Note:** Now, we are jumping back to **epoll** subsystem from **binder** subsystem.


Let's open `workshop/android-4.14-dev/goldfish/fs/eventpoll.c` and try to understand what `ep_ptable_queue_proc` function does.

```c
/*
 * This is the callback that is used to add our wait queue to the
 * target file wakeup lists.
 */
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
				 poll_table *pt)
{
	struct epitem *epi = ep_item_from_epqueue(pt);
	struct eppoll_entry *pwq;

	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
		pwq->whead = whead;
		pwq->base = epi;
		if (epi->event.events & EPOLLEXCLUSIVE)
			add_wait_queue_exclusive(whead, &pwq->wait);
		else
			add_wait_queue(whead, &pwq->wait);
		list_add_tail(&pwq->llink, &epi->pwqlist);
		epi->nwait++;
	} else {
		/* We have to signal that an error occurred */
		epi->nwait = -1;
	}
}
```

* gets pointer to `epitem` structure from `poll_table` by calling `ep_item_from_epqueue` function
* allocates `eppoll_entry` structure and initializes it members
* sets `whead` member of `eppoll_entry` structure to the pointer to `wait_queue_head_t` structure passed by `binder_poll`, which is basically the pointer to `binder_thread->wait`
* links `whead` (`binder_thread->wait`) to `eppoll_entry->wait` by calling `add_wait_queue`
* finally `eppoll_entry->llink` is linked to `epitem->pwqlist` by calling `list_add_tail`


> **Note:** If you look at the code, you will notice that there are **two** places which holds the reference to `binder_thread->wait`. First reference is stored in `eppoll_entry->wait` and the second reference is stored in `eppoll_entry->whead`.


Let's see how `struct eppoll_entry` is defined.

```c
struct eppoll_entry {
        /* List header used to link this structure to the "struct epitem" */
        struct list_head llink;

        /* The "base" pointer is set to the container "struct epitem" */
        struct epitem *base;

        /*
         * Wait queue item that will be linked to the target file wait
         * queue head.
         */
        wait_queue_entry_t wait;

        /* The wait queue head that linked the "wait" wait queue item */
        wait_queue_head_t *whead;
};
```

Below given diagram is the simplified call graph of how `binder_thread` structure is allocated and gets linked to **epoll** subsystem.

<p align="center">
  <img src="../images/binder-thread-eventpoll-link-call-graph.png" alt="binder_thread eventpoll call graph" title="binder_thread eventpoll call graph"/>
</p>


Below given diagram shows how `eventpoll` structure is connected with `binder_thread` structure.

<p align="center">
  <img src="../images/eventpoll-binder-thread-link.png" alt="binder_thread eventpoll connection" title="binder_thread eventpoll connection"/>
</p>


### ioctl {#syscall-ioctl}

```c
ioctl(fd, BINDER_THREAD_EXIT, NULL);
```

Let's open `workshop/android-4.14-dev/goldfish/drivers/android/binder.c` and see how `ioctl` system call is implemented.

```
static const struct file_operations binder_fops = {
        [...]
        .unlocked_ioctl = binder_ioctl,
        .compat_ioctl = binder_ioctl,
        [...]
};
```

We see that `unlocked_ioctl` and `compat_ioctl` system call is handled by `binder_ioctl` function.


Let's follow `binder_ioctl` function and see how it handles `BINDER_THREAD_EXIT` request.

```c
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
        int ret;
        struct binder_proc *proc = filp->private_data;
        struct binder_thread *thread;
        unsigned int size = _IOC_SIZE(cmd);
        void __user *ubuf = (void __user *)arg;
        [...]
        thread = binder_get_thread(proc);
        [...]
        switch (cmd) {
        [...]
        case BINDER_THREAD_EXIT:
                [...]
                binder_thread_release(proc, thread);
                thread = NULL;
                break;
        [...]
        default:
                ret = -EINVAL;
                goto err;
        }
        ret = 0;
        [...]
        return ret;
}
```

* gets the pointer to `binder_thread` structure from `binder_proc` structure
* calls `binder_thread_release` function passing pointers to `binder_proc` and `binder_thread` structures as the parameters


Let's follow `binder_thread_release` and find out what it does.

```c
static int binder_thread_release(struct binder_proc *proc,
                                 struct binder_thread *thread)
{
        [...]
        int active_transactions = 0;
        [...]
        binder_thread_dec_tmpref(thread);
        return active_transactions;
}
```


> **Note:** Remember, we had applied a custom *patch* in this function itself to **reintroduce** the **vulnerability**.


* interesting part of this function is that, it calls the `binder_thread_dec_tmpref` function passing pointer to `binder_thread` structure


Let's follow `binder_thread_dec_tmpref` and find out what it does.

```c
static void binder_thread_dec_tmpref(struct binder_thread *thread)
{
        [...]
        if (thread->is_dead && !atomic_read(&thread->tmp_ref)) {
                [...]
                binder_free_thread(thread);
                return;
        }
        [...]
}
```

* calls `binder_free_thread` function passing pointer to `binder_thread` structure


Let's follow `binder_free_thread` and find out what it does.

```c
static void binder_free_thread(struct binder_thread *thread)
{
        [...]
        kfree(thread);
}
```

* calls `kfree` function which frees the kernel heap chunk storing `binder_thread` structure

If you see the call graph in **[Free](root-cause-analysis.md#revisiting-crash-free)** section, you will find that this is where the `binder_thread` structure is **freed**.


### ep_remove {#syscall-ep-remove}

If you see the call graph in **[Use](root-cause-analysis.md#revisiting-crash-use)** section, you will find that `ep_unregister_pollwait` function is called when `exit_group` system call is executed. `exit_group` is usually called when the process exits. We would want to trigger the call to `ep_unregister_pollwait` at will during exploitation.

Let's look at `workshop/android-4.14-dev/goldfish/fs/eventpoll.c` and try to figure out how we can call `ep_unregister_pollwait` function at will. Basically, we want to inspect the callers of `ep_unregister_pollwait` function.

Looking at the code, I found two interesting callers functions `ep_remove` and `ep_free`. But `ep_remove` is a good candidate because can be called by `epoll_ctl` system call passing `EPOLL_CTL_DEL` as the operation parameter.

```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
                struct epoll_event __user *, event)
{
        [...]
        struct eventpoll *ep;
        struct epitem *epi;
        [...]
        error = -EINVAL;
        switch (op) {
        [...]
        case EPOLL_CTL_DEL:
                if (epi)
                        error = ep_remove(ep, epi);
                else
                        error = -ENOENT;
                break;
        [...]
        }
        [...]
        return error;
}
```

The below given line of code can trigger `ep_unregister_pollwait` function at will.

```c
epoll_ctl(epfd, EPOLL_CTL_DEL, fd, &event);
```


Let's follow `ep_remove` function find out what it does.

```c
static int ep_remove(struct eventpoll *ep, struct epitem *epi)
{
        [...]
        ep_unregister_pollwait(ep, epi);
        [...]
        return 0;
}
```

* calls `ep_unregister_pollwait` function passing pointers to `eventpoll` and `epitem` structures as the parameters


Let's follow `ep_unregister_pollwait` function find out what it does.

```c
static void ep_unregister_pollwait(struct eventpoll *ep, struct epitem *epi)
{
        struct list_head *lsthead = &epi->pwqlist;
        struct eppoll_entry *pwq;

        while (!list_empty(lsthead)) {
                pwq = list_first_entry(lsthead, struct eppoll_entry, llink);
                [...]
                ep_remove_wait_queue(pwq);
                [...]
        }
}
```

* gets the **poll wait queue** `list_head` structure pointer from `epi->pwqlist`.
* gets the pointer `eppoll_entry` from the `epitem->llink` member which of type `struct list_head`
* calls `ep_remove_wait_queue` passing pointer to `eppoll_entry` as the parameter


Let's follow `ep_remove_wait_queue` function find out what it does.

```c
static void ep_remove_wait_queue(struct eppoll_entry *pwq)
{
        wait_queue_head_t *whead;
        [...]
        whead = smp_load_acquire(&pwq->whead);
        if (whead)
                remove_wait_queue(whead, &pwq->wait);
        [...]
}
```

* gets pointer to `wait_queue_head_t` from `eppoll_entry->whead`
* calls `remove_wait_queue` function passing pointers to `wait_queue_head_t` and `eppoll_entry->wait` as the parameters


> **Note:** `eppoll_entry->whead` and `eppoll_entry->wait` both has references to the **dangling** `binder_thread` structure.


Let's open `workshop/android-4.14-dev/goldfish/kernel/sched/wait.c` and follow `remove_wait_queue` function to figure out what it does.

```c
void remove_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
        unsigned long flags;

        spin_lock_irqsave(&wq_head->lock, flags);
        __remove_wait_queue(wq_head, wq_entry);
        spin_unlock_irqrestore(&wq_head->lock, flags);
}
```

* calls `spin_lock_irqsave` function passing pointer `wait_queue_head->lock` to acquire lock


> **Note:** If you look at stack trace in **[Use](root-cause-analysis.md#revisiting-crash-use)** section, you will see that the crash occurred because `_raw_spin_lock_irqsave` used the **dangling** chunk. This is exactly the same place where the use of the **dangling** chunk happened for the first time. Remember `wait_queue_entry` also contains the references to the **dangling** chunk.


* calls `__remove_wait_queue` function passing pointers to `wait_queue_head` and `wait_queue_entry` structures as the parameters


Let's open `workshop/android-4.14-dev/goldfish/include/linux/wait.h` and follow `__remove_wait_queue` function to figure out what it does.

```c
static inline void
__remove_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
{
        list_del(&wq_entry->entry);
}
```

* calls `list_del` function passing pointer to `wait_queue_entry->entry` which is of type `struct list_head` as the parameter


> **Note:** `wait_queue_head` is ignored and not used afterwards.


Let's open `workshop/android-4.14-dev/goldfish/include/linux/list.h` and follow `list_del` function to figure out what it does.

```c
static inline void list_del(struct list_head *entry)
{
        __list_del_entry(entry);
        [...]
}

static inline void __list_del_entry(struct list_head *entry)
{
        [...]
        __list_del(entry->prev, entry->next);
}

static inline void __list_del(struct list_head * prev, struct list_head * next)
{
        next->prev = prev;
        WRITE_ONCE(prev->next, next);
}
```

This is basically **unlink** operation and will write a **pointer** to `binder_thread->wait.head` to `binder_thread->wait.head.next` and `binder_thread->wait.head.prev`, basically **unlink** `eppoll_entry->wait.entry` from `binder_thread->wait.head`.

This is a much better primitive from the point of view of **exploitation** than the first use of **dangling chunk**.

Below given diagrams shows how **circular double linked list** works so that you have better picture of what's really happening.


Let's see how a single initialized node `node1` looks like. In out context, `node1` is `binder_thread->wait.head` and `node2` is `eppoll_entry->wait.entry`.

<p align="center">
  <img src="../images/double-link-list-single-node.png" alt="Double Link List Single Node" title="Double Link List Single Node"/>
</p>

Now, let's see how two nodes `node1` and `node2` are linked.

<p align="center">
  <img src="../images/double-link-list-two-nodes.png" alt="Double Link List Two Nodes" title="Double Link List Two Nodes"/>
</p>

Now, let's see how `node1` node looks like when `node2` node is linked.

<p align="center">
  <img src="../images/double-link-list-unlink.png" alt="Double Link List Unlink" title="Double Link List Unlink"/>
</p>


### Static Analysis Recap {#static-analysis-recap}

Let's do a recap of what we understood from the **root cause analysis** section.

In the beginning of the **[Static Analysis](root-cause-analysis.md#static-analysis)** section we asked **three** questions, let's try to answer those.

* Why `binder_thread` structure was **allocated**?
	* `ep_insert` function triggers the call to `binder_poll` by calling `ep_item_poll` function
	* `binder_poll` tries to find a **thread** to use from the **red black tree** node and if it's not found, a new `binder_thread` structure is allocated


* Why `binder_thread` structure was **freed**?
	* `binder_thread` structure is freed when `ioctl` system call is called explicitly, passing `BINDER_THREAD_EXIT` as the operation code


* Why the use of `binder_thread` structure happened when it's already **freed**?
	* **pointer** to `binder_thread->wait.head` is not removed from `eppoll_entry->whead` and `eppoll_entry->wait.entry` when `binder_thread` structure is **freed** explicitly
	* when the `eventpoll` is removed by calling `epoll_ctl` and passing `EPOLL_CTL_DEL` as the operation parameter, it tries to **unlink** all the **wait queues** and uses the **dangling** `binder_thread` structure


## Dynamic Analysis {#dynamic-analysis}

In this section, we will look into how we can use **GDB** automation to understand the crash behavior.

But before we start doing that, we need to make a hardware changes to the **Android Virtual Device** named **CVE-2019-2215** we created in **[Android Virtual Device](chapters/environment-setup.md#android-virtual-device)** section.

We also need to build the **Android Kernel** without **KASan**, because we don't need the **KASan** support now.


### hw.cpu.ncore {#hw-cpu-ncore}

For better **GDB** debugging and tracing support, it's recommended to set the number of **CPU cores** to **1**.

Open `~/.android/avd/CVE-2019-2215.avd/config.ini` in a text editor and change line `hw.cpu.ncore = 4` to `hw.cpu.ncore = 1`.


### Build Kernel Without KASan {#build-kernel-without-kasan}

This section is exactly same as **[Build Kernel With KASan](vulnerability-trigger.md#build-kernel-with-kasan)**, but this time, we will use a different config file.

You will find the config file in `workshop/build-configs/goldfish.x86_64.relwithdebinfo` directory.

```bash
ARCH=x86_64
BRANCH=relwithdebinfo

CC=clang
CLANG_PREBUILT_BIN=prebuilts-master/clang/host/linux-x86/clang-r377782b/bin
BUILDTOOLS_PREBUILT_BIN=build/build-tools/path/linux-x86
CLANG_TRIPLE=x86_64-linux-gnu-
CROSS_COMPILE=x86_64-linux-androidkernel-
LINUX_GCC_CROSS_COMPILE_PREBUILTS_BIN=prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.9/bin

KERNEL_DIR=goldfish
EXTRA_CMDS=''
STOP_SHIP_TRACEPRINTK=1

FILES="
arch/x86/boot/bzImage
vmlinux
System.map
"

DEFCONFIG=x86_64_ranchu_defconfig
POST_DEFCONFIG_CMDS="check_defconfig && update_debug_config"

function update_debug_config() {
    ${KERNEL_DIR}/scripts/config --file ${OUT_DIR}/.config \
         -e CONFIG_FRAME_POINTER \
         -e CONFIG_DEBUG_INFO \
         -d CONFIG_DEBUG_INFO_REDUCED \
         -d CONFIG_KERNEL_LZ4 \
         -d CONFIG_RANDOMIZE_BASE
    (cd ${OUT_DIR} && \
     make O=${OUT_DIR} $archsubarch CROSS_COMPILE=${CROSS_COMPILE} olddefconfig)
}
```

Now, let's use this config file and start the build process.

```bash
ashfaq@hacksys:~/workshop/android-4.14-dev$ BUILD_CONFIG=../build-configs/goldfish.x86_64.relwithdebinfo build/build.sh
```


### Kernel Tracing {#kernel-tracing}

Our goal is to use **GDB** python **breakpoint** automation to trace function calls and dump the `binder_thread` structure chunk before and after it's **freed**. Also dump the same `binder_thread` structure before and after the **unlink** operation has been done.

You can find a python file `~/workshop/gdb/dynamic-analysis.py`, where I have written some debugging automation to debug this vulnerability at **runtime**.

Let's boot **emulator** with the newly built kernel. 


> **Note:** The *patch* to reintroduce the vulnerability is already applied.


We need four terminal windows this time. Open the first terminal window and launch **emulator**.

```bash
ashfaq@hacksys:~/workshop$ emulator -show-kernel -no-snapshot -wipe-data -avd CVE-2019-2215 -kernel ~/workshop/android-4.14-dev/out/relwithdebinfo/dist/bzImage -qemu -s -S
```

In the second window, we will use **GDB** to attach to the `qemu` instance.

```bash
ashfaq@hacksys:~/workshop$ gdb -quiet ~/workshop/android-4.14-dev/out/relwithdebinfo/dist/vmlinux -ex 'target remote :1234'
```

```
GEF for linux ready, type `gef' to start, `gef config' to configure
77 commands loaded for GDB 8.2 using Python engine 2.7
[*] 3 commands could not be loaded, run `gef missing` to know why.
Reading symbols from /home/ashfaq/workshop/android-4.14-dev/out/kasan/dist/vmlinux...done.
Remote debugging using :1234
warning: while parsing target description (at line 1): Could not load XML document "i386-64bit.xml"
warning: Could not load XML target description; ignoring
0x000000000000fff0 in exception_stacks ()
gef> c
Continuing.
```

Once the **Android** is booted completely, open the third terminal window and where we will build the vulnerability trigger and push it to the **virtual device**.

```bash
ashfaq@hacksys:~/workshop$ cd exploit/
ashfaq@hacksys:~/workshop/exploit$ NDK_ROOT=~/Android/Sdk/ndk/21.0.6113669 make build-trigger push-trigger
Building: cve-2019-2215-trigger
Pushing: cve-2019-2215-trigger to /data/local/tmp
cve-2019-2215-trigger: 1 file pushed, 0 skipped. 44.8 MB/s (3958288 bytes in 0.084s)
```

Now, in the **GDB** window press **CTRL+C** to break in **GDB** so that we can load the custom python script.

You can find `dynamic-analysis.py` which is an automation built on top of `GDB` python scripting in `workshop/gdb`.

```
gef> c
Continuing.
^C
Program received signal SIGINT, Interrupt.
native_safe_halt () at /home/ashfaq/workshop/android-4.14-dev/goldfish/arch/x86/include/asm/irqflags.h:61
61	}
gef> source ~/workshop/gdb/dynamic-analysis.py
Breakpoint 1 at 0xffffffff80824047: file /home/ashfaq/workshop/android-4.14-dev/goldfish/drivers/android/binder.c, line 4701.
Breakpoint 2 at 0xffffffff802aa586: file /home/ashfaq/workshop/android-4.14-dev/goldfish/kernel/sched/wait.c, line 50.
gef> c
Continuing.
```

Now, we can open the fourth terminal window, launch `adb` shell and run the trigger PoC.

```bash
ashfaq@hacksys:~/workshop/exploit$ adb shell
generic_x86_64:/ $ cd /data/local/tmp
generic_x86_64:/data/local/tmp $ ./cve-2019-2215-trigger
generic_x86_64:/data/local/tmp $ 
```

As soon as you execute the trigger PoC, you will see this in the **GDB** terminal window. 

```
binder_free_thread(thread=0xffff88800c18f200)(enter)
0xffff88800c18f200:	0xffff88806793c000	0x0000000000000001
0xffff88800c18f210:	0x0000000000000000	0x0000000000000000
0xffff88800c18f220:	0xffff88800c18f220	0xffff88800c18f220
0xffff88800c18f230:	0x0000002000001b35	0x0000000000000001
0xffff88800c18f240:	0x0000000000000000	0xffff88800c18f248
0xffff88800c18f250:	0xffff88800c18f248	0x0000000000000000
0xffff88800c18f260:	0x0000000000000000	0x0000000000000000
0xffff88800c18f270:	0x0000000000000003	0x0000000000007201
0xffff88800c18f280:	0x0000000000000000	0x0000000000000000
0xffff88800c18f290:	0x0000000000000003	0x0000000000007201
0xffff88800c18f2a0:	0x0000000000000000	0xffff88805c05cae0
0xffff88800c18f2b0:	0xffff88805c05cae0	0x0000000000000000
0xffff88800c18f2c0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f2d0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f2e0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f2f0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f300:	0x0000000000000000	0x0000000000000000
0xffff88800c18f310:	0x0000000000000000	0x0000000000000000
0xffff88800c18f320:	0x0000000000000000	0x0000000000000000
0xffff88800c18f330:	0x0000000000000000	0x0000000000000000
0xffff88800c18f340:	0x0000000000000000	0x0000000000000000
0xffff88800c18f350:	0x0000000000000000	0x0000000000000000
0xffff88800c18f360:	0x0000000000000000	0x0000000000000000
0xffff88800c18f370:	0x0000000000000000	0x0000000000000000
0xffff88800c18f380:	0x0000000000000000	0x0000000000000001
0xffff88800c18f390:	0xffff88806d4bb200

remove_wait_queue(wq_head=0xffff88800c18f2a0, wq_entry=0xffff88805c05cac8)(enter)
0xffff88800c18f200:	0xffff88800c18f600	0x0000000000000001
0xffff88800c18f210:	0x0000000000000000	0x0000000000000000
0xffff88800c18f220:	0xffff88800c18f220	0xffff88800c18f220
0xffff88800c18f230:	0x0000002000001b35	0x0000000000000001
0xffff88800c18f240:	0x0000000000000000	0xffff88800c18f248
0xffff88800c18f250:	0xffff88800c18f248	0x0000000000000000
0xffff88800c18f260:	0x0000000000000000	0x0000000000000000
0xffff88800c18f270:	0x0000000000000003	0x0000000000007201
0xffff88800c18f280:	0x0000000000000000	0x0000000000000000
0xffff88800c18f290:	0x0000000000000003	0x0000000000007201
0xffff88800c18f2a0:	0x0000000000000000	0xffff88805c05cae0
0xffff88800c18f2b0:	0xffff88805c05cae0	0x0000000000000000
0xffff88800c18f2c0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f2d0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f2e0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f2f0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f300:	0x0000000000000000	0x0000000000000000
0xffff88800c18f310:	0x0000000000000000	0x0000000000000000
0xffff88800c18f320:	0x0000000000000000	0x0000000000000000
0xffff88800c18f330:	0x0000000000000000	0x0000000000000000
0xffff88800c18f340:	0x0000000000000000	0x0000000000000000
0xffff88800c18f350:	0x0000000000000000	0x0000000000000000
0xffff88800c18f360:	0x0000000000000000	0x0000000000000000
0xffff88800c18f370:	0x0000000000000000	0x0000000000000000
0xffff88800c18f380:	0x0000000000000000	0x0000000000000001
0xffff88800c18f390:	0xffff88806d4bb200

Breakpoint 3 at 0xffffffff802aa5be: file /home/ashfaq/workshop/android-4.14-dev/goldfish/kernel/sched/wait.c, line 53.
remove_wait_queue_wait.c:52(exit)
0xffff88800c18f200:	0xffff88800c18f600	0x0000000000000001
0xffff88800c18f210:	0x0000000000000000	0x0000000000000000
0xffff88800c18f220:	0xffff88800c18f220	0xffff88800c18f220
0xffff88800c18f230:	0x0000002000001b35	0x0000000000000001
0xffff88800c18f240:	0x0000000000000000	0xffff88800c18f248
0xffff88800c18f250:	0xffff88800c18f248	0x0000000000000000
0xffff88800c18f260:	0x0000000000000000	0x0000000000000000
0xffff88800c18f270:	0x0000000000000003	0x0000000000007201
0xffff88800c18f280:	0x0000000000000000	0x0000000000000000
0xffff88800c18f290:	0x0000000000000003	0x0000000000007201
0xffff88800c18f2a0:	0x0000000000000000	0xffff88800c18f2a8
0xffff88800c18f2b0:	0xffff88800c18f2a8	0x0000000000000000
0xffff88800c18f2c0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f2d0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f2e0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f2f0:	0x0000000000000000	0x0000000000000000
0xffff88800c18f300:	0x0000000000000000	0x0000000000000000
0xffff88800c18f310:	0x0000000000000000	0x0000000000000000
0xffff88800c18f320:	0x0000000000000000	0x0000000000000000
0xffff88800c18f330:	0x0000000000000000	0x0000000000000000
0xffff88800c18f340:	0x0000000000000000	0x0000000000000000
0xffff88800c18f350:	0x0000000000000000	0x0000000000000000
0xffff88800c18f360:	0x0000000000000000	0x0000000000000000
0xffff88800c18f370:	0x0000000000000000	0x0000000000000000
0xffff88800c18f380:	0x0000000000000000	0x0000000000000001
0xffff88800c18f390:	0xffff88806d4bb200
```

Now, let's analyze the output and try to understand what's happening.

If you remember, `binder_free_thread` is the function that will eventually free the `binder_thread` structure.

*After* **free** and *before* the **unlink** operation happens on `binder_thread` structure.

```
0xffff88800c18f2a0:	0x0000000000000000	0xffff88805c05cae0
0xffff88800c18f2b0:	0xffff88805c05cae0	0x0000000000000000
```

`0xffff88800c18f2a0 + 0x8` is the offset of `binder_thread->wait.head` which links `eppoll_entry->wait.entry`.

```
gef> p offsetof(struct binder_thread, wait.head)
$1 = 0xa8
```

`0xffff88805c05cae0` is pointer to `eppoll_entry->wait.entry` which is of type `struct list_head`.

*After* the **unlink** operation happened on `binder_thread` structure.

```
0xffff88800c18f2a0:	0x0000000000000000	0xffff88800c18f2a8
0xffff88800c18f2b0:	0xffff88800c18f2a8	0x0000000000000000
```

If you see closely, after the **unlink** operation happened, a **pointer** to `binder_thread->wait.head` is written to `binder_thread->wait.head.next` and `binder_thread->wait.head.prev`.

This is exactly what we figured out in the **[Static Analysis](root-cause-analysis.md#static-analysis)** section.
