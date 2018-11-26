## epoll
Linux 内核 epoll相关代码位置 fs/eventpoll.c
供用户层调用的三个最常用函数
+ epoll_create
+ epoll_ctl
+ epoll_wait

### epoll_create
```c
SYSCALL_DEFINE1(epoll_create, int, size)
{
	if (size <= 0)
		return -EINVAL;

	return do_epoll_create(0);
}

static int do_epoll_create(int flags)
{
	int error, fd;
	struct eventpoll *ep = NULL;
	struct file *file;

    // 初始化eventpoll数据结构ep
	error = ep_alloc(&ep);

    // Linux 把一切视为文件,为了统一这里创建了一种新的文件系统格式'eventpoll',这样对ep的操作可以抽象为对文件系统的操作;
    // eventpoll_fops是该文件系统的一系列回调,便于内核在发生了特定事件时使用
	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC));
	ep->file = file;
	fd_install(fd, file);

	return fd;
}

static const struct file_operations eventpoll_fops = {
#ifdef CONFIG_PROC_FS
	.show_fdinfo	= ep_show_fdinfo,
#endif
	.release	= ep_eventpoll_release,
	.poll		= ep_eventpoll_poll,
	.llseek		= noop_llseek,
};
```

###  epoll_ctl
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

	// 将用户层要ctl的event内容复制到内核层
	if (ep_op_has_event(op) &&
	    copy_from_user(&epds, event, sizeof(struct epoll_event)))
		goto error_return;


	// 这里f.file->private_data就是前面create阶段建立的eventpoll数据结构
	ep = f.file->private_data;

    // 从eventpoll中找到搜寻fd所代表的epitem
	epi = ep_find(ep, tf.file, fd);

	error = -EINVAL;
    // 根据操作码 决定这个fd 是加入 还是 删除还是 改变属性
	switch (op) {
        // 当操作时 ADD 时,把fd ,epds等事件相关内容insert进 eventpoll 结构
	case EPOLL_CTL_ADD:
		if (!epi) {
			epds.events |= EPOLLERR | EPOLLHUP;
			error = ep_insert(ep, &epds, tf.file, fd, full_check);
		} else
			error = -EEXIST;
		if (full_check)
			clear_tfile_check_list();
		break;
	case EPOLL_CTL_DEL:
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD:
		if (epi) {
			if (!(epi->event.events & EPOLLEXCLUSIVE)) {
				epds.events |= EPOLLERR | EPOLLHUP;
				error = ep_modify(ep, epi, &epds);
			}
		} else
			error = -ENOENT;
		break;
	}

}

static int ep_insert(struct eventpoll *ep, const struct epoll_event *event,
		     struct file *tfile, int fd, int full_check)
{
    struct epitem *epi;
	// 这里使用了一个回调ep_ptable_queue_proc,ep_item_poll将这个回调注册到监听的文件中,这样当该文件触发事件时,会调用ep_ptable_queue_proc,这样就实现了事件信息从硬件到eventpoll结构的传送
	epq.epi = epi;
	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
	revents = ep_item_poll(epi, &epq.pt, 1);

	// 将已经准备好的eventpoll_item插入eventpoll结构中便于管理和搜寻
	ep_rbtree_insert(ep, epi);
}

static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
				 poll_table *pt)
{
	struct epitem *epi = ep_item_from_epqueue(pt);
	struct eppoll_entry *pwq;

    // 这里whead是系统底层的事件内容,就是在这里系统把事件内容复制给了eventpoll结构的队列里,后面的epoll_wait就可以根据eventpoll结构里的事件信息来操作
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

	}
}
```

### epoll_wait
```c
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
		int, maxevents, int, timeout)
{
	return do_epoll_wait(epfd, events, maxevents, timeout);
}

static int do_epoll_wait(int epfd, struct epoll_event __user *events,
			 int maxevents, int timeout)
{
	int error;
	struct fd f;
	struct eventpoll *ep;

	// 取出epfd所代表的文件的内容
	f = fdget(epfd);

    // 取出eventpoll结构
	ep = f.file->private_data;

	// 这里取出所有准备就绪的事件,并把这些事件的信息拷贝到参数events中,这个events指向一块用户空间,完成调用后,用户层可以通过这块空间得到各个事件的信息,并做相应处理
	error = ep_poll(ep, events, maxevents, timeout);

}

static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, long timeout)
{
// ...
check_events:
    // ep_send_events()函数把内核层发生的事件信息传到events(用户层)指向的空间
	if (!res && eavail &&
	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
		goto fetch_events;

	return res;
}

static int ep_send_events(struct eventpoll *ep,
			  struct epoll_event __user *events, int maxevents)
{
	struct ep_send_events_data esed;

	esed.maxevents = maxevents;
	esed.events = events;
    // 从函数名得知,从eventpoll结构扫ready_list,然后传给用户空间esed
	ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
	return esed.res;
}
```

### Linux 怎么把事件让eventpoll结构知晓
+ epoll_create初始化ep;
+ epoll_ctl 控制事件监听,并对要监听的文件符设置回调;
+ linux 怎么把硬件底层触发的事件通过前面设置回调返回给eventpoll结构;
+ epoll_wait 取出eventpoll里的已经就绪的ready_list;