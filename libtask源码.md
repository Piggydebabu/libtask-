# libtask源码

## 基础部分

task结构体:

```c
struct Task
{
	char	name[256];	// offset known to acid
	char	state[256];
    // task链表,为调度作准备
	Task	*next;
	Task	*prev;
	Task	*allnext;
	Task	*allprev;
    // 上下文,封装了ucontext_t
	Context	context;
    // 睡眠时间
	uvlong	alarmtime;
    // task id
	uint	id;
    // 栈地址与栈大小
	uchar	*stk;
	uint	stksize;
    // 是否退出
	int	exiting;
    // 在所有task中的索引
	int	alltaskslot;
    // 是否为系统协程
	int	system;
    // 是否准备好
	int	ready;
    // 启动入口函数
	void	(*startfn)(void*);
    // 入口函数的参数
	void	*startarg;
    // 自定义数据(userdata)
	void	*udata;
};
```

libtask实现了main函数, main函数中调用了taskcreate()和taskscheduler()

main函数:

```c
int
main(int argc, char **argv)
{
	struct sigaction sa, osa;
	// 注册信号处理函数
	memset(&sa, 0, sizeof sa);
	sa.sa_handler = taskinfo;
	sa.sa_flags = SA_RESTART;
	sigaction(SIGQUIT, &sa, &osa);

#ifdef SIGINFO
	sigaction(SIGINFO, &sa, &osa);
#endif
	// 保存传入的命令行参数,准备传入taskmain
	argv0 = argv[0];
	taskargc = argc;
	taskargv = argv;
	// 主栈大小256K
	if(mainstacksize == 0)
		mainstacksize = 256*1024;
	// 创建第一个协程并调度
    taskcreate(taskmainstart, nil, mainstacksize);
	taskscheduler();
    // 不应该在main返回
	fprint(2, "taskscheduler returned in main!\n");
	abort();
	return 0;
}
```

taskcreate()

```c
/// 参数:协程包裹的函数,函数参数,协程栈大小
/// 返回值:协程id
/// 调用taskalloc()和taskready()
int
taskcreate(void (*fn)(void*), void *arg, uint stack)
{
	int id;
	Task *t;

    // 用taskalloc创建协程
	t = taskalloc(fn, arg, stack);
	taskcount++;
	id = t->id;
	if(nalltask%64 == 0){
		alltask = realloc(alltask, (nalltask+64)*sizeof(alltask[0]));
		if(alltask == nil){
			fprint(2, "out of memory\n");
			abort();
		}
	}
	t->alltaskslot = nalltask;
	alltask[nalltask++] = t;
    // 加入执行队列
	taskready(t);
	return id;
}
/// 协程创建函数
static Task*
taskalloc(void (*fn)(void*), void *arg, uint stack)
{
	Task *t;
	sigset_t zero;
	uint x, y;
	ulong z;

	/* allocate the task and stack together */
    // 协程本身大小和栈大小之和
	t = malloc(sizeof *t+stack);
	if(t == nil){
		fprint(2, "taskalloc malloc: %r\n");
		abort();
	}
    // 置零
	memset(t, 0, sizeof *t);
    // 栈的内存位置
	t->stk = (uchar*)(t+1);
    // 栈大小
	t->stksize = stack;
    // 全局id+1
	t->id = ++taskidgen;
    // 入口函数及参数
	t->startfn = fn;
	t->startarg = arg;

	/* do a reasonable initialization */
	memset(&t->context.uc, 0, sizeof t->context.uc);
	sigemptyset(&zero);
    // 初始化uc_sigmask字段为空，即不阻塞信号
	sigprocmask(SIG_BLOCK, &zero, &t->context.uc.uc_sigmask);

	/* must initialize with current context */
    // 获取当前上下文,存储到context.uc,即初始化uc
	if(getcontext(&t->context.uc) < 0){
		fprint(2, "getcontext: %r\n");
		abort();
	}

	/* call makecontext to do the real work. */
	/* leave a few words open on both ends */
    // 设置栈地址与栈大小
	t->context.uc.uc_stack.ss_sp = t->stk+8;
	t->context.uc.uc_stack.ss_size = t->stksize-64;
	/*
	 * All this magic is because you have to pass makecontext a
	 * function that takes some number of word-sized variables,
	 * and on 64-bit machines pointers are bigger than words.
	 */
//print("make %p\n", t);
    // 处理64位与32位指针差异,将t分解为x和y传入
	z = (ulong)t;
	y = z;
	z >>= 16;	/* hide undefined 32-bit shift from 32-bit compilers */
	x = z>>16;
    // 保存信息到uc字段
	makecontext(&t->context.uc, (void(*)())taskstart, 2, y, x);

	return t;
}
// taskready()
void
taskready(Task *t)
{
    // 设置对应位
	t->ready = 1;
    // 添加到执行队列
	addtask(&taskrunqueue, t);
}
```

taskrunqueue记录所有ready的协程, 执行taskcreate创建协程后, 协程还没有运行起来, 需要用到调度器调度执行

```c
static void
taskscheduler(void)
{
	int i;
	Task *t;

	taskdebug("scheduler enter");
    // 无限循环
	for(;;){
        // 没有用户协程了,退出
		if(taskcount == 0)
			exit(taskexitval);
        // 从就绪队列拿出一个协程
		t = taskrunqueue.head;
		if(t == nil){
			fprint(2, "no runnable tasks! %d tasks stalled\n", taskcount);
			exit(1);
		}
        // 从就绪队列删除该协程
		deltask(&taskrunqueue, t);
        // 就绪变为运行,故ready = 0
		t->ready = 0;
        // 全局变量,当前运行协程变为t
		taskrunning = t;
        // 切换次数加一
		tasknswitch++;
		taskdebug("run %d (%s)", t->id, t->name);
        // 切换到t执行,并保存当前上下文到taskschedcontext
		contextswitch(&taskschedcontext, &t->context);
//print("back in scheduler\n");
        // t执行完了,切换回来(协程主动yield),现在没有协程运行
		taskrunning = nil;
        // 刚刚执行的t退出了
		if(t->exiting){
            // 若不是系统协程, 协程数减一
			if(!t->system)
				taskcount--;
            // 当前协程在alltask的索引
			i = t->alltaskslot;
			// 将alltask最后一个协程与当前协程交换, 并更新被置换的协程的索引
            alltask[i] = alltask[--nalltask];
			alltask[i]->alltaskslot = i;
            // 释放t内存
			free(t);
		}
	}
}
```

调度器的核心逻辑就三个 

-  从就绪队列中拿出一个协程t，并把t移出就绪队列 
-  通过contextswitch切换到协程t中执行 
-  协程t切换回调度中心，如果t已经退出，则修改数据结构，然后回收他占据的内存。继续调度其他协程执行

至此，协程就开始跑起来了。并且也有了调度系统。这里的调度机制是比较简单的，就是按着先进先出的方式就绪调度，并且是非抢占的。即没有按时间片调度的概念，一个协程的执行时间由自己决定，放弃执行的权力也是自己控制的，当协程不想执行了可以调用taskyield让出cpu。

```c
int
taskyield(void)
{
	int n;
	// 当前切换协程的次数
	n = tasknswitch;
    // 当前协程插入就绪队列
	taskready(taskrunning);
 	// 设置t->state
	taskstate("yield");
    // 切换
	taskswitch();
	return tasknswitch - n - 1;
}

// 切换协程,taskrunning是正在执行的协程，taskschedcontext是调度协程（主线程）的上下文，
// 切换到调度中心(主协程)，并保存当前上下文到taskrunning->context
void taskswitch(void)
{
    needstack(0);
    contextswitch(&taskrunning->context, &taskschedcontext);
}

// 真正切换协程的逻辑
static void contextswitch(Context *from, Context *to)
{
    // swap其实就是调用了一次get和set
    // 首先get当前上下文保存到第一个参数
    // 然后set第二个参数指向的上下文到当前上下文
    if(swapcontext(&from->uc, &to->uc) < 0){
        fprint(2, "swapcontext failed: %r\n");
        assert(0);
    }
}
```

我们需要自己实现taskmain函数, 其在taskmainstart中被调用, 作用是开始第一个协程

```c
static void
taskmainstart(void *v)
{
	taskname("taskmain");
	taskmain(taskargc, taskargv);
}
```

## 网络部分

调用netannounce可以启动一个tcp服务器, 其封装了socket建立, bind, listen

```c
// 参数: 协议类型, IP地址, 端口号
// 返回值: socketfd
int
netannounce(int istcp, char *server, int port)
{
	int fd, n, proto;
	struct sockaddr_in sa;
	socklen_t sn;
	uint32_t ip;

	taskstate("netannounce");
	proto = istcp ? SOCK_STREAM : SOCK_DGRAM;
	memset(&sa, 0, sizeof sa);
	sa.sin_family = AF_INET;
	if(server != nil && strcmp(server, "*") != 0){
		if(netlookup(server, &ip) < 0){
			taskstate("netlookup failed");
			return -1;
		}
		memmove(&sa.sin_addr, &ip, 4);
	}
	sa.sin_port = htons(port);
	if((fd = socket(AF_INET, proto, 0)) < 0){
		taskstate("socket failed");
		return -1;
	}
	
	/* set reuse flag for tcp */
    // 允许地址重用
	if(istcp && getsockopt(fd, SOL_SOCKET, SO_TYPE, (void*)&n, &sn) >= 0){
		n = 1;
		setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (char*)&n, sizeof n);
	}

	if(bind(fd, (struct sockaddr*)&sa, sizeof sa) < 0){
		taskstate("bind failed");
		close(fd);
		return -1;
	}
	
	if(proto == SOCK_STREAM)
		listen(fd, 16);
	// 设置fd为非阻塞. 
    // 在后面调用accept的时候，如果是阻塞的文件描述符，那么就会引起进程挂起，
    // 而非阻塞模式下，操作系统会返回EAGAIN的错误码，通过这个错误码我们可以决定下一步做什么.
	fdnoblock(fd);
	taskstate("netannounce succeeded");
	return fd;
}
```

有listen就有accept

```c
// 处理连接
// 参数: sockfd, ip, port
// 返回值: 和客户端通信的fd
int
netaccept(int fd, char *server, int *port)
{
    int cfd, one;
    struct sockaddr_in sa;
    uchar *ip;
    socklen_t len;
    // 注册事件到epoll，等待事件触发
    fdwait(fd, 'r');
    len = sizeof sa;
    // 触发后说明有连接了，则执行accept
    if((cfd = accept(fd, (void*)&sa, &len)) < 0){
        return -1;
    }
    // 和客户端通信的fd也改成非阻塞模式
    fdnoblock(cfd);
    one = 1;
    setsockopt(cfd, IPPROTO_TCP, TCP_NODELAY, (char*)&one, sizeof one);
    return cfd;
}
// 参数: 要注册的fd, 读事件or写事件
// 因为等待IO就绪, 切换协程
void
fdwait(int fd, int rw)
{
    // 是否已经初始化epoll
	if(!startedfdtask){
		startedfdtask = 1;
        epfd = epoll_create(1);
        assert(epfd >= 0);
        // 未初始化, 则创建一个协程做IO管理
		taskcreate(fdtask, 0, 32768);
	}

	taskstate("fdwait for %s", rw=='r' ? "read" : rw=='w' ? "write" : "error");
    struct epoll_event ev = {0};
    // 记录事件对应协程和感兴趣的事件
    ev.data.ptr = taskrunning;
	switch(rw){
	case 'r':
		ev.events |= EPOLLIN | EPOLLPRI;
		break;
	case 'w':
		ev.events |= EPOLLOUT;
		break;
	}
	// 注册协程, fd, 读写事件到epoll上
    int r = epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
    // 切换到其他协程,等待被唤醒
	taskswitch();
    // 唤醒后删除刚刚注册的事件
    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, &ev);
}
```

fdwait首先把fd注册到epoll中，然后把协程切换到下一个待执行的协程。这里有个细节，当协程T被调度执行的时候，他是脱离了就绪队列的，而taskswitch函数只是实现了切换上下文到调度中心，调度中心会从就绪队列从选择下一个协程执行，那么这时候，脱离就绪队列的协程T就处于孤儿状态，看起来再也无法给调度中心选中执行，这个问题的处理方式是，把协程、fd和感兴趣的事件信息一起注册到epoll中，当epoll监听到某个fd的事件发生时，就会把对应的协程加入就绪队列，这样协程就可以被调度执行了。

在fdwait函数一开始那里处理了epoll相关的逻辑。epoll的逻辑也是在一个协程中执行的，但是epoll所在协程和一般协程不一样，类似于操作系统的内核线程一样，epoll所在的协程成为系统协程，即不是用户定义的，而是系统定义的。我们看一下实现

```c
void
fdtask(void *v)
{
	int i, ms;
	Task *t;
	uvlong now;
	// 更改属性为系统协程
	tasksystem();
	taskname("fdtask");
    struct epoll_event events[1000];
	for(;;){
		/* let everyone else run */
        // 让其他协程先执行, 直到只剩当前协程
		while(taskyield() > 0)
			;
		/* we're the only one runnable - poll for i/o */
        // 开始执行
		errno = 0;
		taskstate("epoll");
        // 没有定时事件则一直阻塞
        // sleeping静态Tasklist变量, 用来跟踪所有当前正在等待某个事件（如 I/O 操作或超时）完成的任务。
		if((t=sleeping.head) == nil)
			ms = -1;
		else{
			/* sleep at most 5s */
            // 获取当前纳秒事件
			now = nsec();
			if(now >= t->alarmtime)
				ms = 0;
			else if(now+5*1000*1000*1000LL >= t->alarmtime)
				ms = (t->alarmtime - now)/1000000;
			else
				ms = 5000;
		}
        // 就绪的事件集合
        int nevents;
        // 等待事件发生, ms是等待的超时时间
		if((nevents = epoll_wait(epfd, events, 1000, ms)) < 0){
			if(errno == EINTR)
				continue;
			fprint(2, "epoll: %s\n", strerror(errno));
			taskexitall(0);
		}

		/* wake up the guys who deserve it */
        // 事件触发, 依次插入到就绪队列
		for(i=0; i<nevents; i++){
            taskready((Task *)events[i].data.ptr);
		}
		
		now = nsec();
        // 处理超时任务，如果当前时间超过了任务的闹钟时间，将其加入就绪队列。并从等待队列删除任务
		while((t=sleeping.head) && now >= t->alarmtime){
			deltask(&sleeping, t);
			if(!t->system && --sleepingcounted == 0)
				taskcount--;
			taskready(t);
		}
	}
}
```

epoll的处理逻辑: 通过epoll_wait阻塞，然后epoll_wait返回时，处理每一个发生的事件，而且libtask还支持超时事件。另外libtask中当还有其他就绪协程的时候，是不会进入epoll_wait的，它会把cpu让给就绪的协程（通过循环taskyield函数），当就绪队列只有epoll所在的协程时才会进入epoll的逻辑。 至此，我们看到了libtask中如何把异步变成同步的。

当用户要调用一个可能会引起进程挂起的接口时，就可以调用libtask提供的一个相应的API，比如我们想读一个文件，我们可以调用libtask的fdread。

```
int
fdread(int fd, void *buf, int n)
{
    int m;
    // 非阻塞读，如果不满足则再注册到epoll，参考fdread1
    while((m=read(fd, buf, n)) < 0 && errno == EAGAIN)
        fdwait(fd, 'r');
    return m;
}
```

这样就不需要担心进程被挂起，同时也不需要处理epoll相关的逻辑（注册事件，事件触发时的处理等等）。异步转同步，libtask的方式就是通过提供对应的API，先把用户的fd注册到epoll中，然后切换到其他协程，等epoll监听到事件触发时，就会把对应的协程插入就绪队列，当该协程被调度中心选中执行时，就会继续执行剩下的逻辑而不会引起进程挂起，因为这时候所等待的条件已经满足。

