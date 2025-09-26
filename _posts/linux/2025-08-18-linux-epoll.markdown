---
layout: default
linux: true
modal-id: 40000001
date: 2025-08-18
img: pexels-nikiemmert-33127482.jpg
alt: image-alt
project-date: August 2025
client: Start Bootstrap
category: linux
subtitle: epoll
description: epoll 详解
---


### 前言
***
在 linux 中，epoll 是一种用于处理多个文件描述符的 I/O 事件通知机制，它提供了一种更高效的方式来处理多个文件描述符的 I/O 事件。linux 中多路复用方案有 select、poll、epoll、kqueue 等，其中 epoll 是 Linux 下最常用的一种。  
下面是使用 epoll 的一些示例代码：
```c
int main(){
    listen(lfd, ...);

    cfd1 = accept(...);
    cfd2 = accept(...);
    efd = epoll_create(...);

    epoll_ctl(efd, EPOLL_CTL_ADD, cfd1, ...);
    epoll_ctl(efd, EPOLL_CTL_ADD, cfd2, ...);
    epoll_wait(efd, ...)
}
```
其中和 epoll 相关的函数有：  
epoll_create() 创建一个 epoll 实例，返回一个 epoll 文件描述符。  
epoll_ctl() 将一个文件描述符添加到 epoll 实例中，或者从 epoll 实例中删除一个文件描述符。  
epoll_wait() 等待 epoll 实例中的文件描述符发生 I/O 事件，返回发生事件的文件描述符。  
epoll_pwait() 和 epoll_wait() 类似，但是可以指定一个超时时间，如果超时则返回 0。

####  一、accept 创建新 socket
当 accept 之后，进程会创建一个新的 socket，专门用于和对应的客户端通信，然后把它放到当前进程的打开文件列表中。  
<center>   
<img src="../../img/linux/epoll/epoll1.png" class="img-responsive img-centered" alt="image-alt">
</center>
其中一条连接的 socket 内核对象更为具体一点的结构图如下：  
<center>   
<img src="../../img/linux/epoll/epoll2.png" class="img-responsive img-centered" alt="image-alt">
</center>

接下来我们看一下接收连接时 socket 内核对象的创建源码，accept 的系统调用代码位于源文件 `net/socket.c` 中，代码片段如下：
```CPP
static int __sys_accept4_file(struct file *file, struct sockaddr __user *upeer_sockaddr,
			      int __user *upeer_addrlen, int flags)
{
	struct proto_accept_arg arg = { };
	//声明 新file
	struct file *newfile;
	//声明 新文件描述符的数量
	int newfd;

	if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
		return -EINVAL;

	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

	//获取未使用的 fd 数量
	newfd = get_unused_fd_flags(flags);
	if (unlikely(newfd < 0))
		return newfd;

	//接收 socket 连接 （创建 socket 并分配 file）
	newfile = do_accept(file, &arg, upeer_sockaddr, upeer_addrlen,
			    flags);
	if (IS_ERR(newfile)) {
		put_unused_fd(newfd);
		return PTR_ERR(newfile);
	}
	//添加新文件到当前进程的打开文件列表
	fd_install(newfd, newfile);
	return newfd;
}

struct file *do_accept(struct file *file, struct proto_accept_arg *arg,
		       struct sockaddr __user *upeer_sockaddr,
		       int __user *upeer_addrlen, int flags)
{
	//声明原 sock 和 新socket
	struct socket *sock, *newsock;
	//声明新file
	struct file *newfile;
	int err, len;
	struct sockaddr_storage address;
	const struct proto_ops *ops;

	//从原文件中获取 socket
	sock = sock_from_file(file);
	if (!sock)
		return ERR_PTR(-ENOTSOCK);

	//初始化新 socket
	newsock = sock_alloc();
	if (!newsock)
		return ERR_PTR(-ENFILE);
	//获取原 sock 的 ops 用于赋值给 newsock
	ops = READ_ONCE(sock->ops);

	//通过旧 sock 属性填充 newsock
	newsock->type = sock->type;
	newsock->ops = ops;

	/*
	 * We don't need try_module_get here, as the listening socket (sock)
	 * has the protocol module (sock->ops->owner) held.
	 */
	__module_get(ops->owner);

	//申请新的 file 对象，并将新的 socket 绑定上去
	newfile = sock_alloc_file(newsock, flags, sock->sk->sk_prot_creator->name);
	if (IS_ERR(newfile))
		return newfile;

	err = security_socket_accept(sock, newsock);
	if (err)
		goto out_fd;

	arg->flags |= sock->file->f_flags;
	// newsock 接收连接 开始监听
	err = ops->accept(sock, newsock, arg);
	if (err < 0)
		goto out_fd;

	if (upeer_sockaddr) {
		len = ops->getname(newsock, (struct sockaddr *)&address, 2);
		if (len < 0) {
			err = -ECONNABORTED;
			goto out_fd;
		}
		err = move_addr_to_user(&address,
					len, upeer_sockaddr, upeer_addrlen);
		if (err < 0)
			goto out_fd;
	}

	/* File flags are not inherited via accept() unlike another OSes.
	 * 与其他操作系统不同，文件标志不会通过 accept 继承
	 * */
	return newfile;
out_fd:
	fput(newfile);
	return ERR_PTR(err);
}

/**
 * fd_install - install a file pointer in the fd array
 * 在 fd 数组中安装一个文件指针
 * @fd: file descriptor to install the file in
 * @file: the file to install
 *
 * This consumes the "file" refcount, so callers should treat it
 * as if they had called fput(file).
 */
void fd_install(unsigned int fd, struct file *file)
{
	struct files_struct *files = current->files;
	struct fdtable *fdt;

	if (WARN_ON_ONCE(unlikely(file->f_mode & FMODE_BACKING)))
		return;

	//声明 rcu 读，禁止cpu抢占
	rcu_read_lock_sched();

	//若文件列表处于数量调整中
	if (unlikely(files->resize_in_progress)) {
		//rcu 读解锁
		rcu_read_unlock_sched();
		//加自旋锁
		spin_lock(&files->file_lock);
		fdt = files_fdtable(files);
		VFS_BUG_ON(rcu_access_pointer(fdt->fd[fd]) != NULL);
		//添加文件
		rcu_assign_pointer(fdt->fd[fd], file);
		//自旋解锁
		spin_unlock(&files->file_lock);
		return;
	}
	/* coupled with smp_wmb() in expand_fdtable() */
	smp_rmb();
	fdt = rcu_dereference_sched(files->fdt);
	VFS_BUG_ON(rcu_access_pointer(fdt->fd[fd]) != NULL);
	//将文件添加到可用的文件数组中
	rcu_assign_pointer(fdt->fd[fd], file);
	//rcu 读解锁
	rcu_read_unlock_sched();
}
```

##### 1.1 初始化 struct socket 对象  
在上述源码中，首先通过 do_accept 方法创建新 file，其中调用 sock_alloc 申请一个 struct socket 对象出来，然后接着把 listen 状态的 socket 对象上的协议操作函数集合 ops 赋值给新的 socket。

##### 1.2 初始化 struct file 对象  
struct socket 对象中有一个重要的成员 -- file 内核对象指针。这个指针初始化的时候是空的。在 accpet 方法里会调用 sock_alloc_file 来申请内存并初始化，然后将新 file 对象设置到 socket 对象的 file 成员上。

##### 1.3 接收连接  
在 socket 内核对象中除了 file 对象指针以外，有一个核心成员 sock，数据结构非常大，是 socket 的核心内核对象。发送队列、接收队列、等待队列等核心数据结构都位于此。其定义位置文件 inclued/net/socket.h 中。  
ops->accept 对应的方法是 inet_accept。它执行的时候会从握手队列里直接获取创建好的 sock。  
##### 1.4 添加新文件到当前进程的打开文件列表中
当 file、socket、sock 等关键内核对象创建完毕后，剩下的事就是把它挂到当前进程的打开文件列表中。  


####  二、epoll_create 实现
当用户进程调用 epoll_create 时，内核会创建一个 struct eventpoll 的内核对象，并同样把它关联到当前进程的已打开文件列表中。  
<center>   
<img src="../../img/linux/epoll/epoll3.png" class="img-responsive img-centered" alt="image-alt">
</center>

对于 struct eventpoll 对象，更详细的结构如下
<center>   
<img src="../../img/linux/epoll/epoll4.png" class="img-responsive img-centered" alt="image-alt">
</center>


```CPP
/*
 * This structure is stored inside the "private_data" member of the file
 * structure and represents the main data structure for the eventpoll
 * interface.
 * 这个结构存储在文件结构的 private_data 成员中，代表 eventpoll 接口的主要数据结构
 */
struct eventpoll {
	/*
	 * This mutex is used to ensure that files are not removed
	 * while epoll is using them. This is held during the event
	 * collection loop, the file cleanup path, the epoll file exit
	 * code and the ctl operations.
	 * 这个互斥锁用于确保 epoll 使用文件时文件不会被删除。这在事件收集循环，文件清理路径，epoll文件退出代码和 ctl 操作期间进行
	 */
	struct mutex mtx;

	/* Wait queue used by sys_epoll_wait()
	 * sys_epoll_wait 使用的等待队列
	 * */
	wait_queue_head_t wq;

	/* Wait queue used by file->poll()
	 *  file->poll()使用的等待队列
	 * */
	wait_queue_head_t poll_wait;

	/* List of ready file descriptors
	 * 已就绪文件描述符的列表
	 * */
	struct list_head rdllist;

	/* Lock which protects rdllist and ovflist
	 * 保护 rdllist 和 ovflist 的锁
	 * */
	rwlock_t lock;

	/* RB tree root used to store monitored fd structs
	 * RB树的根用来存储受监控的 fd 结构
	 * */
	struct rb_root_cached rbr;

	/*
	 * This is a single linked list that chains all the "struct epitem" that
	 * happened while transferring ready events to userspace w/out
	 * holding ->lock.
	 * 这是一个链表，它将所有在将就绪时间转移到用户空间时发生的 struct epitem 链接到一起，并持有锁
	 */
	struct epitem *ovflist;

	/* wakeup_source used when ep_send_events or __ep_eventpoll_poll is running
	 * 当 ep_send_events 或 __ep_eventpoll_poll 在运行时使用 wakeup_source
	 * */
	struct wakeup_source *ws;

	/* The user that created the eventpoll descriptor
	 * 创建 eventpoll 描述符的用户
	 * */
	struct user_struct *user;

	// 文件
	struct file *file;

	/* used to optimize loop detection check
	 * 用于优化环路检测
	 * */
	u64 gen;
	struct hlist_head refs;

	/*
	 * usage count, used together with epitem->dying to
	 * orchestrate the disposal of this struct
	 * 使用计数，与 epitem->dying 一起使用，以编排该结构的位置
	 */
	refcount_t refcount;

#ifdef CONFIG_NET_RX_BUSY_POLL
	/* used to track busy poll napi_id */
	unsigned int napi_id;
	/* busy poll timeout */
	u32 busy_poll_usecs;
	/* busy poll packet budget */
	u16 busy_poll_budget;
	bool prefer_busy_poll;
#endif

#ifdef CONFIG_DEBUG_LOCK_ALLOC
	/* tracks wakeup nests for lockdep validation */
	u8 nests;
#endif
};
```  

eventpoll 这个结构体中的几个成员的含义如下：  
- wq：等待队列链表。软中断数据就绪的时候会通过 wq 来找到阻塞在 epoll 对象上的用户进程。  
- rbr：一棵红黑树。为了支持对海量连接的高效查找、插入和删除，eventpoll 内部使用了一棵红黑树。通过这棵树来管理用户进程下添加进来的所有 socket 连接 
- rdllist：就绪的描述符的链表。当有的连接就绪的时候，内核会把就绪的连接放到 rdllist 链表里。这样应用进程只需要判断链表就能找出就绪进程，而不用去遍历整棵树。  

eventpoll 初始化源码如下：  
```cpp
/*
 * Open an eventpoll file descriptor.
 * 打开一个 eventpoll 文件描述符
 */
static int do_epoll_create(int flags)
{
	int error, fd;
	struct eventpoll *ep = NULL;
	struct file *file;

	/* Check the EPOLL_* constant for consistency. 检查 epoll常数是否一致 */
	BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);

	if (flags & ~EPOLL_CLOEXEC)
		return -EINVAL;
	/*
	 * Create the internal data structure ("struct eventpoll").
	 * 创建内部数据结构 eventpoll
	 */
	error = ep_alloc(&ep);
	if (error < 0)
		return error;
	/*
	 * Creates all the items needed to setup an eventpoll file. That is,
	 * a file structure and a free file descriptor.
	 * 创建设置 eventpoll 文件所需的所有项，也就是说，一个文件和一个空闲文件描述符
	 */
	fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
	if (fd < 0) {
		error = fd;
		goto out_free_ep;
	}
	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC));
	if (IS_ERR(file)) {
		error = PTR_ERR(file);
		goto out_free_fd;
	}
	//设置 epoll文件
	ep->file = file;
	//将文件安装到文件数组中
	fd_install(fd, file);
	return fd;

out_free_fd:
	put_unused_fd(fd);
out_free_ep:
	ep_clear_and_put(ep);
	return error;
}

static int ep_alloc(struct eventpoll **pep)
{
	//初始化 epoll 属性
	struct eventpoll *ep;

	ep = kzalloc(sizeof(*ep), GFP_KERNEL);
	if (unlikely(!ep))
		return -ENOMEM;

	mutex_init(&ep->mtx);
	rwlock_init(&ep->lock);
	init_waitqueue_head(&ep->wq);
	init_waitqueue_head(&ep->poll_wait);
	INIT_LIST_HEAD(&ep->rdllist);
	ep->rbr = RB_ROOT_CACHED;
	ep->ovflist = EP_UNACTIVE_PTR;
	ep->user = get_current_user();
	refcount_set(&ep->refcount, 1);

	*pep = ep;

	return 0;
}
```

####  三、epoll_ctl 添加 socket
假设我们现在和客户端们的多个连接的 socket 都创建好了，也创建好了 epoll 内核对象。在使用 epoll_ctl 注册每一个 socket 的时候，内核会做如下三件事：  
1. 分配一个红黑树节点对象 epitem
2. 添加等待事件到 socket 的等待队列中，其回调函数是 ep_poll_callback
3. 将 epitem 添加到红黑树中

通过 epoll_ctl 添加两个 socket 以后，这些内核数据结构最终在进程中的关系图大致如下：  
<center>   
<img src="../../img/linux/epoll/epoll5.png" class="img-responsive img-centered" alt="image-alt">
</center>

源码如下：
```cpp
// file：fs/eventpoll.c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
        struct epoll_event __user *, event)
{
    struct eventpoll *ep;
    struct file *file, *tfile;

    //根据 epfd 找到 eventpoll 内核对象
    file = fget(epfd);
    ep = file->private_data;

    //根据 socket 句柄号， 找到其 file 内核对象
    tfile = fget(fd);

    switch (op) {
    case EPOLL_CTL_ADD:
        if (!epi) {
            epds.events |= POLLERR | POLLHUP;
            error = ep_insert(ep, &epds, tfile, fd);
        } else
            error = -EEXIST;
        clear_tfile_check_list();
        break;
}
```
在 eopll_ctl 中首先根据传入 fd 找到 eventpoll、socket 相关的内核对象。 对于 EPOLL_CTL_ADD 操作来说，会然后执行到 ep_insert 函数。所有的注册都是在这个函数中完成的。  
```CPP
//file: fs/eventpoll.c
static int ep_insert(struct eventpoll *ep, 
                struct epoll_event *event,
                struct file *tfile, int fd)
{
    //3.1 分配并初始化 epitem
    //分配一个epi对象
    struct epitem *epi;
    if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
        return -ENOMEM;

    //对分配的epi进行初始化
    //epi->ffd中存了句柄号和struct file对象地址
    INIT_LIST_HEAD(&epi->pwqlist);
    epi->ep = ep;
    ep_set_ffd(&epi->ffd, tfile, fd);

    //3.2 设置 socket 等待队列
    //定义并初始化 ep_pqueue 对象
    struct ep_pqueue epq;
    epq.epi = epi;
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

    //调用 ep_ptable_queue_proc 注册回调函数 
    //实际注入的函数为 ep_poll_callback
    revents = ep_item_poll(epi, &epq.pt);

    ......
    //3.3 将epi插入到 eventpoll 对象中的红黑树中
    ep_rbtree_insert(ep, epi);
    ......
}
```

#####  3.1 分配并初始化 epitem  
对于每一个socket，调用 epoll_ctl 的时候，都会为之分配一个 epitem。该结构的主要数据如下：  
```CPP
//file: fs/eventpoll.c
struct epitem {

    //红黑树节点
    struct rb_node rbn;

    //socket文件描述符信息
    struct epoll_filefd ffd;

    //所归属的 eventpoll 对象
    struct eventpoll *ep;

    //等待队列
    struct list_head pwqlist;
}
```
对 epitem 进行了一些初始化，首先在 epi->ep = ep 这行代码中将 ep 指针指向 eventpoll 对象。另外用要添加的 socket 的 file、fd 来填充 epitem->ffd。  
<center>   
<img src="../../img/linux/epoll/epoll6.png" class="img-responsive img-centered" alt="image-alt">
</center>
其中使用到的 ep_set_ffd 函数如下：  
```cpp
//file: fs/eventpoll.c
static inline void ep_set_ffd(struct epoll_filefd *ffd,
                              struct file *file, int fd)
{
    ffd->file = file;
    ffd->fd = fd;
}
```

#####  3.2 设置 socket 队列  
在创建 epitem 并初始化之后， ep_insert 中第二件事情就是设置 socket对象上的等待任务队列。并把函数 fs/eventpoll.c 文件下的 ep_poll_callback 设置为数据就绪时候的回调函数。  
<center>   
<img src="../../img/linux/epoll/epoll7.png" class="img-responsive img-centered" alt="image-alt">
</center>
首先来看 ep_item_poll：  

``` CPP
static inline unsigned int ep_item_poll(struct epitem *epi, poll_table *pt)
{
    pt->_key = epi->event.events;

    return epi->ffd.file->f_op->poll(epi->ffd.file, pt) & epi->event.events;
}
```
这里调用到了 socket 下的 file->f_op->poll。通过上面第一节的 socket 结构图，我们知道这个函数实际上是 socket_poll。  
```CPP
/* No kernel lock held - perfect */
static unsigned int sock_poll(struct file *file, poll_table *wait)
{
    ...
    return sock->ops->poll(file, sock, wait);
}
```
同样回看第一节里的 socket 的结构图， socket->ops->poll 其实指向的是 tcp_poll。  
```CPP
//file: net/ipv4/tcp.c
unsigned int tcp_poll(struct file *file, struct socket *sock, poll_table *wait)
{
    struct sock *sk = sock->sk;

    sock_poll_wait(file, sk_sleep(sk), wait);
}
```
在 sock_poll_wait 的第二个参数传参前，先调用了 sk_sleep 函数。在这个函数里他获取了 sock 对象下的等待队列表头 wait_queue_head_t，待会等待队列项就插入这里。 这里稍微注意下，是socket的等待队列，不是epoll对象的。  
来看 sk_sleep 函数：  
```CPP
//file: include/net/sock.h
static inline wait_queue_head_t *sk_sleep(struct sock *sk)
{
    BUILD_BUG_ON(offsetof(struct socket_wq, wait) != 0);
    return &rcu_dereference_raw(sk->sk_wq)->wait;
}
```
接着真正进入 sock_poll_wait 函数，这里会调用 sock_poll_wait 函数，这个函数会调用 wait_queue_head_t 的 wait_event_lock_irqsave 函数。  
```CPP
static inline void sock_poll_wait(struct file *filp,
        wait_queue_head_t *wait_address, poll_table *p)
{
    poll_wait(filp, wait_address, p);
}

static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
{
    if (p && p->_qproc && wait_address)
        p->_qproc(filp, wait_address, p);
}
```
这里的 qproc 是个函数指针，他在前面的 init_poll_funcptr 调用时被设置成了 ep_ptable_queue_proc 函数。  
```CPP
static int ep_insert(...)
{
    ...
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
    ...
}

//file: include/linux/poll.h
static inline void init_poll_funcptr(poll_table *pt, 
    poll_queue_proc qproc)
{
    pt->_qproc = qproc;
    pt->_key   = ~0UL; /* all events enabled */
}
```
在 ep_ptable_queue_proc 函数中，新建了一个等待队列项，并注册其回调函数为 ep_poll_callback 函数。 然后再将这个等待项添加到 socket 的等待队列中。  
```CPP
//file: fs/eventpoll.c
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
                 poll_table *pt)
{
    struct eppoll_entry *pwq;
    f (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
                //初始化回调方法
                init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);

                //将ep_poll_callback放入socket的等待队列whead（注意不是epoll的等待队列）
                add_wait_queue(whead, &pwq->wait);

}
```
在阻塞式的系统调用 recvfrom 里，由于需要再数据就绪的时候唤醒用户进程，所以等待对象项的 private 会设置成当前用户进程描述符 current。  
而我们今天的 socket 是交给 epoll 来管理的，不需要在一个 socket 就绪的时候唤醒进程，所以这个里 q->private 没啥用就设置成了 NULL。  
```CPP
//file:include/linux/wait.h
static inline void init_waitqueue_func_entry(
    wait_queue_t *q, wait_queue_func_t func)
{
    q->flags = 0;
    q->private = NULL;

    //ep_poll_callback 注册到 wait_queue_t对象上
    //有数据到达的时候调用 q->func
    q->func = func;   
}
```
如上，等待队列项中仅仅只设置了回调函数 q->func 为 ep_poll_callback。软中断将数据收到 socket的接收队列后，会通过注册的这个 ep_poll_callback 函数来回调，进而通知到 epoll 对象。  

#####  3.3 插入红黑树  
分配完 epitem 对象后，紧接着并把它插入到红黑树中。一个插入了一些 socket 描述符的 epoll 里的红黑树的示意图如下：  
<center>   
<img src="../../img/linux/epoll/epoll8.png" class="img-responsive img-centered" alt="image-alt">
</center>


####  四、epoll_wait 等待接收  
epoll_wait 做的事情不复杂。当它被调用时观察 eventpoll->rdllist 链表里有没有数据即可。有数据就返回，没有数据就创建一个等待队列项，将其添加到 eventpoll 的等待队列上，然后把自己阻塞掉。  
<center>   
<img src="../../img/linux/epoll/epoll9.png" class="img-responsive img-centered" alt="image-alt">
</center>

源码如下：  
```CPP
/**
 * ep_poll - Retrieves ready events, and delivers them to the caller-supplied
 *           event buffer.
 *           检索已准备好的事件，并将它们传递给调用者提供的事件缓冲区
 *
 * @ep: Pointer to the eventpoll context. 指向 eventpoll 上下文的指针
 * @events: Pointer to the userspace buffer where the ready events should be
 *          stored.
 *          指向用户缓冲区的指针，就绪事件存储在此
 * @maxevents: Size (in terms of number of events) of the caller event buffer.调用者事件缓冲区大小（以事件数量表示）
 *
 * @timeout: Maximum timeout for the ready events fetch operation, in
 *           timespec. If the timeout is zero, the function will not block,
 *           while if the @timeout ptr is NULL, the function will block
 *           until at least one event has been retrieved (or an error
 *           occurred).
 *           准备事件获取操作的最大 超时事件。如果为0，函数不会阻塞，然而如果为null，函数将阻塞直到有一个事件发生或发生异常
 *
 * Return: the number of ready events which have been fetched, or an
 *          error code, in case of error.
 */
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, struct timespec64 *timeout)
{
	int res, eavail, timed_out = 0;
	u64 slack = 0;
	wait_queue_entry_t wait;
	ktime_t expires, *to = NULL;

	lockdep_assert_irqs_enabled();

	if (timeout && (timeout->tv_sec | timeout->tv_nsec)) {
		slack = select_estimate_accuracy(timeout);
		to = &expires;
		*to = timespec64_to_ktime(*timeout);
	} else if (timeout) {
		/*
		 * Avoid the unnecessary trip to the wait queue loop, if the
		 * caller specified a non blocking operation.
		 */
		timed_out = 1;
	}

	/*
	 * This call is racy: We may or may not see events that are being added
	 * to the ready list under the lock (e.g., in IRQ callbacks). For cases
	 * with a non-zero timeout, this thread will check the ready list under
	 * lock and will add to the wait queue.  For cases with a zero
	 * timeout, the user by definition should not care and will have to
	 * recheck again.
	 * 这个调用是动态的：我们可能会也可能不会看到正在被添加到锁下的就绪列表中的事件。
	 * 对于非零超时的情况，该线程将检查锁下的就绪列表并将其添加到等待队列中。
	 * 对于超时为零的情况，用户应该不用关心，并且必须再次检查
	 */
	eavail = ep_events_available(ep);

	while (1) {
		//若事件已就绪
		if (eavail) {
			//尝试发送事件
			res = ep_try_send_events(ep, events, maxevents);
			if (res)
				return res;
		}

		if (timed_out)
			return 0;

		eavail = ep_busy_loop(ep);
		if (eavail)
			continue;

		if (signal_pending(current))
			return -EINTR;

		/*
		 * Internally init_wait() uses autoremove_wake_function(),
		 * thus wait entry is removed from the wait queue on each
		 * wakeup. Why it is important? In case of several waiters
		 * each new wakeup will hit the next waiter, giving it the
		 * chance to harvest new event. Otherwise wakeup can be
		 * lost. This is also good performance-wise, because on
		 * normal wakeup path no need to call __remove_wait_queue()
		 * explicitly, thus ep->lock is not taken, which halts the
		 * event delivery.
		 * 在内部init_wait() 使用 autoremove_wake_function()，因此等待条目在每次唤醒时从等待队列中删除。
		 * 为什么它很重要？在几个等待者的情况下，每个新的唤醒将集中下一个等待者，给它机会收货新的事件，否则唤醒可能会丢失。
		 * 这在性能方面也很好，因为在正常的唤醒路径上不需要显示的调用 __remove_wait_queue，因此不会采取 ep->lock 锁，这会停止事件传递
		 *
		 * In fact, we now use an even more aggressive function that
		 * unconditionally removes, because we don't reuse the wait
		 * entry between loop iterations. This lets us also avoid the
		 * performance issue if a process is killed, causing all of its
		 * threads to wake up without being removed normally.
		 * 实际上，我们现在使用了一个更加激进的函数，它无条件的删除，因为我们不会在循环迭代直接重用等待条目。如果一个进程被杀死，导致他的所有线程在没有被正常移除的情况下被唤醒，这也让我们避免了性能问题
		 */
		init_wait(&wait);
		wait.func = ep_autoremove_wake_function;

		write_lock_irq(&ep->lock);
		/*
		 * Barrierless variant, waitqueue_active() is called under
		 * the same lock on wakeup ep_poll_callback() side, so it
		 * is safe to avoid an explicit barrier.
		 * 无屏障的变体，waitqueue_active 在唤醒 ep_poll_callback  端的相同锁下被调用，因此避免显示屏障是安全的
		 */
		__set_current_state(TASK_INTERRUPTIBLE);

		/*
		 * Do the final check under the lock. ep_start/done_scan()
		 * plays with two lists (->rdllist and ->ovflist) and there
		 * is always a race when both lists are empty for short
		 * period of time although events are pending, so lock is
		 * important.
		 * 在锁下做最后的检查，ep_start/done_scan() 使用两个列表 (->rdllist and ->ovflist)  ，当两个列表在短时间内为空时总是存在竞争，尽管事件正在挂起，因此锁很重要
		 */
		eavail = ep_events_available(ep);
		if (!eavail)
			//把新 waitqueue 添加到 epoll->wq 链表里
			__add_wait_queue_exclusive(&ep->wq, &wait);

		write_unlock_irq(&ep->lock);

		if (!eavail)
			timed_out = !ep_schedule_timeout(to) ||
				    //让出cpu 主动进入睡眠
				!schedule_hrtimeout_range(to, slack,
							  HRTIMER_MODE_ABS);
		__set_current_state(TASK_RUNNING);

		/*
		 * We were woken up, thus go and try to harvest some events.
		 * If timed out and still on the wait queue, recheck eavail
		 * carefully under lock, below.
		 */
		eavail = 1;

		if (!list_empty_careful(&wait.entry)) {
			write_lock_irq(&ep->lock);
			/*
			 * If the thread timed out and is not on the wait queue,
			 * it means that the thread was woken up after its
			 * timeout expired before it could reacquire the lock.
			 * Thus, when wait.entry is empty, it needs to harvest
			 * events.
			 */
			if (timed_out)
				eavail = list_empty(&wait.entry);
			__remove_wait_queue(&ep->wq, &wait);
			write_unlock_irq(&ep->lock);
		}
	}
}
```

#####  4.1 判断就绪队列上有没有事件就绪  
首先调用 ep_events_available() 判断就绪队列上有没有事件就绪。  
```CPP
//file: fs/eventpoll.c
static inline int ep_events_available(struct eventpoll *ep)
{
    return !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
}
```

#####  4.2 定义等待时间并关联当前进程  
假设确实没有就绪的连接，那接着会进入 init_waitqueue_entry 中定义等待任务，并把 current（当前进程）添加到 waitqueue 上。  
```CPP
//file: include/linux/wait.h
static inline void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)
{
    q->flags = 0;
    q->private = p;
    q->func = default_wake_function;
}
```
注意这里的回调函数名称是 default_wake_function，后续在第五节数据来了会调用到该函数  

#####  4.3 添加到等待队列  
```CPP
static inline void __add_wait_queue_exclusive(wait_queue_head_t *q,
                                wait_queue_t *wait)
{
    wait->flags |= WQ_FLAG_EXCLUSIVE;
    __add_wait_queue(q, wait);
}
```
在这里，把上一小节定义的等待任务添加到 epoll 对象的等待队列中。 

#####  4.4 让出CPU主动进入睡眠状态  
通过 set_current_state 把当前进程设置为可打断。 调用 schedule_hrtimeout_range() 让出CPU，并设置超时时间。  
```CPP
//file: kernel/hrtimer.c
int __sched schedule_hrtimeout_range(ktime_t *expires, 
    unsigned long delta, const enum hrtimer_mode mode)
{
    return schedule_hrtimeout_range_clock(
            expires, delta, mode, CLOCK_MONOTONIC);
}

int __sched schedule_hrtimeout_range_clock(...)
{
    schedule();
    ...
}
```

####  五、数据来了  
在前面 epoll_ctl 执行的时候，内核为每一个 socket 上都添加一个等待队列项。在 epoll_wait 运行完的时候，又在 event poll 对象上添加了等待队列元素。在讨论数据开始接受之前，把这些队列项的内容再稍微总结一下。  

<center>   
<img src="../../img/linux/epoll/epoll10.png" class="img-responsive img-centered" alt="image-alt">
</center>

- socket->sock->sk_data_ready 设置的就绪处理函数是 sock_def_readable  
- 在 socket 的等待队列项中，其回调函数是 ep_poll_callback。另外其 private 没有用，指向的是空指针 null  
- 在 event poll 的等待队列项中，回调函数是 default_wake_function，其 private 指向的是等待该事件的用户进程  

#####  5.1 接收数据到任务队列  
```CPP
// file: net/ipv4/tcp_ipv4.c
int tcp_v4_rcv(struct sk_buff *skb)
{
    ......
    th = tcp_hdr(skb); //获取tcp header
    iph = ip_hdr(skb); //获取ip header

    //根据数据包 header 中的 ip、端口信息查找到对应的socket
    sk = __inet_lookup_skb(&tcp_hashinfo, skb, th->source, th->dest);
    ......

    //socket 未被用户锁定
    if (!sock_owned_by_user(sk)) {
        {
            if (!tcp_prequeue(sk, skb))
                ret = tcp_v4_do_rcv(sk, skb);
        }
    }
}
```  
在 tcp_v4_rcv 中首先根据收到的网络包的 header 里面的 source 的 dest 信息来在本机上查询对应的 socket。找到以后，直接进入接收的主体函数 tcp_v4_do_rcv 中。  
```CPP
//file: net/ipv4/tcp_ipv4.c
int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
    if (sk->sk_state == TCP_ESTABLISHED) { 

        //执行连接状态下的数据处理
        if (tcp_rcv_established(sk, skb, tcp_hdr(skb), skb->len)) {
            rsk = sk;
            goto reset;
        }
        return 0;
    }

    //其它非 ESTABLISH 状态的数据包处理
    ......
}
```  
我们假设处理的是 ESTABLISH 状态下的包，这样就又进入了 tcp_rcv_establish 函数中进行处理。  
```CPP
//file: net/ipv4/tcp_input.c
int tcp_rcv_established(struct sock *sk, struct sk_buff *skb,
            const struct tcphdr *th, unsigned int len)
{
    ......

    //接收数据到队列中
    eaten = tcp_queue_rcv(sk, skb, tcp_header_len,
                                    &fragstolen);

    //数据 ready，唤醒 socket 上阻塞掉的进程
    sk->sk_data_ready(sk, 0);
```  
在 tcp_rcv_establish 中通过调用 tcp_queue_rcv() 接收数据到 socket 的数据队列中。  

<center>   
<img src="../../img/linux/epoll/epoll11.png" class="img-responsive img-centered" alt="image-alt">
</center>

```CPP
//file: net/ipv4/tcp_input.c
static int __must_check tcp_queue_rcv(struct sock *sk, struct sk_buff *skb, int hdrlen,
            bool *fragstolen)
{
    //把接收到的数据放到 socket 的接收队列的尾部
    if (!eaten) {
        __skb_queue_tail(&sk->sk_receive_queue, skb);
        skb_set_owner_r(skb, sk);
    }
    return eaten;
}
```

#####  5.2 查找就绪回调函数  
调用  tcp_queue_rcv() 接收完成之后，接着再调用 sk_data_ready() 来唤醒在 socket上等待的用户进程。这又是一个函数指针。  
回想上面第一节我们在 accept 函数创建 socket 流量里提到的 socket_init_data 的函数，在这个函数里已经把 sk_data_ready 设置成 sock_def_readable 函数了。它是默认的数据就绪处理函数。  
当 socket 上数据就绪时候，内核将以 sock_def_readable() 这个函数为入口，找到 epoll_ctl 添加到 socket 时在其上设置的回调函数 ep_poll_callback。  

<center>   
<img src="../../img/linux/epoll/epoll12.png" class="img-responsive img-centered" alt="image-alt">
</center>

我们来详细看下细节：  
```CPP
//file: net/core/sock.c
static void sock_def_readable(struct sock *sk, int len)
{
    struct socket_wq *wq;

    rcu_read_lock();
    wq = rcu_dereference(sk->sk_wq);

    //这个名字起的不好，并不是有阻塞的进程，
    //而是判断等待队列不为空
    if (wq_has_sleeper(wq))
        //执行等待队列项上的回调函数
        wake_up_interruptible_sync_poll(&wq->wait, POLLIN | POLLPRI |
                        POLLRDNORM | POLLRDBAND);
    sk_wake_async(sk, SOCK_WAKE_WAITD, POLL_IN);
    rcu_read_unlock();
}
```
这里的函数名其实有迷惑人的地方：  
- wq_has_sleeper() 对于简单的 recvfrom 系统调用来说，确实是判断是否有进程阻塞。但是对于 epoll 下的 socket 只是判断等待队列不为空，不一定有进程阻塞。  
- wake_up_interruptible_sync_poll() 只是会进入到 socket 等待队列项上设置的回调函数，并不一定有唤醒进程的操作。  

接下来重点看  wake_up_interruptible_sync_poll,看一下内核是怎么找到等待队列项里注册的回调函数的。  
```CPP
//file: include/linux/wait.h
#define wake_up_interruptible_sync_poll(x, m)       \
    __wake_up_sync_key((x), TASK_INTERRUPTIBLE, 1, (void *) (m))
    

//file: kernel/sched/core.c
void __wake_up_sync_key(wait_queue_head_t *q, unsigned int mode,
            int nr_exclusive, void *key)
{
    ...
    __wake_up_common(q, mode, nr_exclusive, wake_flags, key);
}


static void __wake_up_common(wait_queue_head_t *q, unsigned int mode,
            int nr_exclusive, int wake_flags, void *key)
{
    wait_queue_t *curr, *next;

    list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
        unsigned flags = curr->flags;

        if (curr->func(curr, mode, wake_flags, key) &&
                (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
            break;
    }
}
```
在 __wake_up_common() 中，选出等待队列里注册的某个元素 curr，回调其 curr->func。回忆我们 ep_insert 调用的时候，把这个 func 设置成 ep_poll_callback。  

#####  5.3 执行 socket 就绪回调函数  
在上一小节找到了 socket 等待队列项里注册的函数 ep_poll_callback，软中断会执行这个函数。  
```CPP
//file: fs/eventpoll.c
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    //获取 wait 对应的 epitem
    struct epitem *epi = ep_item_from_wait(wait);

    //获取 epitem 对应的 eventpoll 结构体
    struct eventpoll *ep = epi->ep;

    //1. 将当前epitem 添加到 eventpoll 的就绪队列中
    list_add_tail(&epi->rdllink, &ep->rdllist);

    //2. 查看 eventpoll 的等待队列上是否有在等待
    if (waitqueue_active(&ep->wq))
        wake_up_locked(&ep->wq);
```
在 ep_poll_callback 根据等待任务队列项上的额外的 base 指针可以找到 epitem，进而可以找到 evnetpoll 对象。  
首先它做的第一个件事就是把自己的 epitem 添加到 epoll 的就绪队列中。  
接着它又会查看 eventpoll 对象上的等待队列里是否有等待项。如果没执行软中断的事情就做完了。如果有等待项，那就查找到等待项了设置的回调函数。   
<center>   
<img src="../../img/linux/epoll/epoll13.png" class="img-responsive img-centered" alt="image-alt">
</center>
调用 wake_up_locked(&ep->wq) => __wake_up_common  

```CPP
static void __wake_up_common(wait_queue_head_t *q, unsigned int mode,
            int nr_exclusive, int wake_flags, void *key)
{
    wait_queue_t *curr, *next;

    list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
        unsigned flags = curr->flags;

        if (curr->func(curr, mode, wake_flags, key) &&
                (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
            break;
    }
}
```
在 __wake_up_common里，调用 curr->func。这里的func是在 epoll_wait 传入的 default_wake_function 函数。  

#####  5.4 执行 epoll 就绪通知  
在 defaulf_wake_function 中找到等待队列项里的进程描述符，然后唤醒。  

<center>   
<img src="../../img/linux/epoll/epoll14.png" class="img-responsive img-centered" alt="image-alt">
</center>

源码如下：  
```CPP
//file:kernel/sched/core.c
int default_wake_function(wait_queue_t *curr, unsigned mode, int wake_flags,
                void *key)
{
    return try_to_wake_up(curr->private, mode, wake_flags);
}
```
等待队列项 curr->private 指针是在 epoll 对象上等待而被阻塞掉的进程。  
将 epoll_wait 进程推入可运行队列，等待内核重新调度进程。然后 epoll_wait 对应的这个进程重新运行后，就从 schedule 恢复。  
当进程醒来后，继续从 epoll_wait 时暂停的代码继续指向。把 rdlist 中就绪的事件返回给用户进程。  
```CPP
//file: fs/eventpoll.c
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
             int maxevents, long timeout)
{

    ......
    __remove_wait_queue(&ep->wq, &wait);

    set_current_state(TASK_RUNNING);
    }
check_events:
    //返回就绪事件给用户进程
    ep_send_events(ep, events, maxevents))
}
```

从用户角度看，epoll_wait 只是多等待了一会儿而已，但执行流还是顺序的。   


####  五、总结  
用一幅图总结一下 epoll 的整个工作流程。  
<center>   
<img src="../../img/linux/epoll/epoll15.png" class="img-responsive img-centered" alt="image-alt">
</center>

总结下，epoll 相关的函数里内核运行环境分两部分：  
- 用户进程内核态。进行调用 epoll_wait，等函数时会将进程陷入内核态来执行。这部分代码负责查看接收队列，以及负责把当前进程阻塞掉，让出 CPU。  
- 硬软中断上下文。在这些组件中，将包从网卡接收过来进行处理，然后放到 socket 的接收队列。对于 epoll 来说，再找到 socket 关联的 epitem，并把它添加到 epoll 对象的就绪列表中。这个时候再捎带检查下 epoll 上是否有被阻塞的进程，如果有则唤醒。  




> 参考博客  
> https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247484905&idx=1&sn=a74ed5d7551c4fb80a8abe057405ea5e&scene=21&poc_token=HALuomijPelcW11XeXi6qzXJlpKTr4kcK25KiVcG
> https://zhuanlan.zhihu.com/p/26351726602

