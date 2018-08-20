高级语言写**业务**代码，基本不会关心什么是 `socket`, 如何创建与销毁，比如 `go` 因为语言封装好了这一系列操作。一般书里都会讲，要调用 `socket`、`bind`、`connect` 等系统调用，就可以创建。然后`read`、`write`、`close` 这些标准方法读写数据。那么背后内核是如何工作的呢？
****
###什么是 socket
两台主机通过网络传输数据，需要建立 `socket` 连接，而这个连接通过四元组来确定唯一性 `local ip`、`local port`、`remote ip`、 `remote port`, 比如使用 `ss -ant` 查看当前机器所有的 tcp 网络连接，还能查看所处的状态。
```
State      Recv-Q Send-Q        Local Address:Port          Peer Address:Port
TIME-WAIT  0      0               10.20.34.24:32708          10.20.55.42:8889
CLOSE-WAIT 5      0               10.20.34.24:8888           10.20.43.36:25722
TIME-WAIT  0      0             180.101.136.8:33572        115.231.97.35:9075
```
从数据流动态角度来看，网卡收到数据包传给二层，二层较验后，查看 `mac` 是否是本机，是的话去掉二层的 `header`, 传给三层。三层较验后，查看 `ip` 是否本机，是的话去掉三层 `header` 传给四层，四层就是我们常说的 `tcp\udp` 传输层，四层根据端口来识别唯一的 `socket`, 将数据写到对应的 `skb` 缓冲区。

###创建 socket
```
int socket(int domain, int type, int protocol);
```
内核对外提供了接口，`man socket` 查看使用详情。我们一般创建 `tcp` 连接，domain 指定 `PF_INET`, type 指定 `SOCK_STREAM`, protocol 默认 0 即可。查看内核源码，
```
int __sys_socket(int family, int type, int protocol)
{
	int retval;
	struct socket *sock;
	int flags;
        ......
	// 内核分配 socket 结构体
	retval = sock_create(family, type, protocol, &sock);
	if (retval < 0)
		return retval;
	// 申请 fd, 绑定 socket 和 fd，并返回
	return sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
}
```
1. 根据指定协义类型，内核创建 `struct socket` 数据结构
2. 分配一个文件描述符 `fd`, 将这个 `socket` 和 `fd` 建立映射关系

大家都知道 linux 一切皆文件，所有操作都可以使用类似 `file` 的接口，好处是屏弊 `socket` 底层细节，足够抽象。

###内核如何创建 socket 数据结构
稍复杂一些，首先要有一个概念，`socket` 是一个接口，能适配非常多种协义，必然会根据传入的 `family` `type` 创建不同类型的 `socket`，这是面象对象。c 语言为了模拟多态行为，使用了大量函数指针。
```
int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
{
	int err;
	struct socket *sock;
	const struct net_proto_family *pf;
    ......
	sock = sock_alloc();
	if (!sock) {
		net_warn_ratelimited("socket: no more sockets\n");
		return -ENFILE;	/* Not exactly a match, but its the
				   closest posix thing */
	}

	sock->type = type;
    ......
	rcu_read_lock();
	pf = rcu_dereference(net_families[family]);
	err = -EAFNOSUPPORT;
	if (!pf)
		goto out_release;

	/*
	 * We will call the ->create function, that possibly is in a loadable
	 * module, so we have to bump that loadable module refcnt first.
	 */
	if (!try_module_get(pf->owner))
		goto out_release;

	/* Now protected by module ref count */
	rcu_read_unlock();

	err = pf->create(net, sock, protocol, kern);
	if (err < 0)
		goto out_module_put;
	return 0;
    ......
}
```
省略了注释和较验代码，这里重点关注两个函数 `sock_alloc` 和 `pf->create`, 先创建 `socket` 结构体, 然后根据协义去真正的初始化。
####sock_alloc 函数的实现
```
struct socket *sock_alloc(void)
{
	struct inode *inode;
	struct socket *sock;
	inode = new_inode_pseudo(sock_mnt->mnt_sb);
	if (!inode)
		return NULL;
	sock = SOCKET_I(inode);
	inode->i_ino = get_next_ino();
	inode->i_mode = S_IFSOCK | S_IRWXUGO;
	inode->i_uid = current_fsuid();
	inode->i_gid = current_fsgid();
	inode->i_op = &sockfs_inode_ops; // 实现多态的数组指针
	return sock;
}
```
这里很好理解，首先 `new_inode_pseudo` 去申请 `inode`, 并通过宏 `SOCKET_I` 获取到 `socket` 指针，`inode` 设置属性，这里有两个细节
1.`SOCKET_I ` 宏非常有意思，给定结构体成员地址，根据成员偏移量来找到结构体地址
```
static inline struct socket *SOCKET_I(struct inode *inode)
{	// container_of(ptr, type, member) 
	// 当 ptr 是 type 类型 成原 member 时，根据 ptr 地址，获取 type 地址
	return &container_of(inode, struct socket_alloc, vfs_inode)->socket;
}
```
2. `sockfs_inode_ops` 是函数指针结构体，把 `socket` 当成文件操作时，回调这个函数指针，实现多态的关键。通过查看源码，相当于重写了 `listxattr` 和 `setattr` 两个函数。

####pf->create 实现
```
pf = rcu_dereference(net_families[family]);
```
再回到 `__sock_create` 函数，`net_families` 可以理解为工厂方法的数组，`family` 协义做为索引，找到对应 `net_proto_family` 结构体
```
struct net_proto_family {
	int		family;
	int		(*create)(struct net *net, struct socket *sock,
				  int protocol, int kern);
	struct module	*owner;
};
static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,
	.create = inet_create,
	.owner	= THIS_MODULE,
};
```
每种协义都要在内核启运后，注册到全局 `net_families`, 比如 `ipv4` 的就会注册 `inet_family_ops`, 而最终创建 `socket` 就由 `inet_create` 来完成。

####inet_create
核心函数，比较复杂。linux 网络协义栈为了实现扩展性，结构体层层嵌套，最让人弄晕的就是 `sock`, `inet_sock`, `tcp_sock`, `socket`.
1. 根据 `type`, `protocol` 确定函数操作接口 `inet_stream_ops`, `tcp_prot`
```
	list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {

		err = 0;
		/* Check the non-wild match. */
		if (protocol == answer->protocol) {
			if (protocol != IPPROTO_IP)
				break;
		} else {
			/* Check for the two wild cases. */
			if (IPPROTO_IP == protocol) {
				protocol = answer->protocol;
				break;
			}
			if (IPPROTO_IP == answer->protocol)
				break;
		}
		err = -EPROTONOSUPPORT;
	}
    ......
	sock->ops = answer->ops; // inet_stream_ops
	answer_prot = answer->prot; // tcp_prot
	answer_flags = answer->flags;
```
`inetsw` 是一个全局链表数组，由 `PF_INET` 协义族初始化时根据 `inetsw_array` 生成。这里保存了不同 `type`, `protocol` 的操作接口，也可以理解为工厂模式

2. 分配 `struct sock` 结构体
```
struct sock *sk_alloc(struct net *net, int family, gfp_t priority,
		      struct proto *prot, int kern)
{
	struct sock *sk;
	sk = sk_prot_alloc(prot, priority | __GFP_ZERO, family);
	if (sk) {
		sk->sk_family = family;
		/*
		 * See comment in struct sock definition to understand
		 * why we need sk_prot_creator -acme
		 */
		sk->sk_prot = sk->sk_prot_creator = prot;
		sk->sk_kern_sock = kern;
		sock_lock_init(sk);
		sk->sk_net_refcnt = kern ? 0 : 1;
    ......
	}
	return sk;
}
```
首先由内核 `slab` 分配器分配 `sock` 结构体，并将 `sk_prot` 设置为 `tcp_prot`, 这样就实现了面向对象的多态。其它设置 `refcnt`, `cgroup` 不去理会。

这里有个疑问，为什么 `sk_alloc` 返回值为 `struct sock *` 类型，最后还能当 `struct inet_sock *` 来使用呢？重点就在 `sk_prot_alloc` 函数的实现
```
sk = kmalloc(prot->obj_size, priority);
```
那么 `prot->obj_size` 大小是多少呢？我们查看文件 tcp_ipv4.c 中 `tcp_prot` 可以得知：
```
struct proto tcp_prot = {
    ......
	.obj_size		= sizeof(struct tcp_sock),
    ......
};
```
也就是说，内核根据指定大小返回一块内存区域，具体调用方把它当成什么类型，完全不关心。而 linux 通过结构体层层嵌套，来实现不同协义。
![tcp sock struct](https://upload-images.jianshu.io/upload_images/659877-f4cdab27512e8159.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图是一个简单的结构体说明，`socket` 可以理解为用户空间操作接口。c 语言是没有面继承功能的，而为了模拟这个关系，就像套娃一样。通过查看具体的结构体定义，可以得知，`tcp_sock` 是对 `inet_connection_sock` 的扩展，而 `inet_connection_sock` 又是依次对其它的扩展，被扩展的结构体必须是第一个成员变量。当引用 `struct sock *` 操作时，就实现在面向对像里，引用父指针操作子类的功能。

3. 初始化 `socket`, `sock`

调用 `sock_init_data` 初始化，将 `socket` 和 刚申请的 `struct sock` 绑定，初始化 `struct sock` 成员变量，这里涉及几个套娃结构体的赋值。如果不同 `type` 的 `sk_prot` 设置了 `init` 函数，那么调用，这里会设用 `tcp_prot.tcp_init_sock` 

###sock_map_fd 将 sock 与 fd 绑定
再加到系统调用，来看 `sock_map_fd` 的实现。
```
static int sock_map_fd(struct socket *sock, int flags)
{
	struct file *newfile;
	int fd = get_unused_fd_flags(flags);
	if (unlikely(fd < 0)) {
		sock_release(sock);
		return fd;
	}

	newfile = sock_alloc_file(sock, flags, NULL);
	if (likely(!IS_ERR(newfile))) {
		fd_install(fd, newfile);
		return fd;
	}

	put_unused_fd(fd);
	return PTR_ERR(newfile);
}
```
能过源码，大读看到，先调用 `get_unused_fd_flags ` 分配一个 `fd`，再根据己经生成的 `struct socket` 分配一个上层的 `struct file *` 结构体，最后 `fd_install` 将两个关联起来。 

1. `get_unused_fd_flags` 如何分配 `fd`
```
int get_unused_fd_flags(unsigned flags)
{
	return __alloc_fd(current->files, 0, rlimit(RLIMIT_NOFILE), flags);
}
```
内核从当前进程的打开文件表 `struct files_struct *` 分配 `fd`, 再读 `__alloc_fd` 函数，最重要的是 `files_struct.fdt` 变量，定义如下
```
struct fdtable {
	unsigned int max_fds;
	struct file __rcu **fd;      /* current fd array */
	unsigned long *close_on_exec;
	unsigned long *open_fds;
	unsigned long *full_fds_bits;
	struct rcu_head rcu;
};
```
其中 `**fd` 是当前进程打开的文件数组，`open_fds`, `close_on_exec`, `full_fds_bits` 均是位图。分配 `fd` 时，内核按照从 0 自增的顺序，如果发生了回绕，那么就查看 `open_fds` 位图，如果为 0 说明对应的 `fd` 没有被使用，那么就分配。

2. `sock_alloc_file` 如何关分配 `file`
通过阅读源码，最重要的代码是 `alloc_file`, 其中 `socket_file_ops` 将 `file` 的行为指定为 `socket_file_ops`，然后做一些其它初始化。
```
file = alloc_file(&path, FMODE_READ | FMODE_WRITE,
		  &socket_file_ops);
```
```
static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.sendpage =	sock_sendpage,
	.splice_write = generic_splice_sendpage,
	.splice_read =	sock_splice_read,
};
```

3.`fd_install` 如何关联 `fd`, `files`
```
void __fd_install(struct files_struct *files, unsigned int fd,
		struct file *file)
{
	struct fdtable *fdt;
    ......
	fdt = rcu_dereference_sched(files->fdt);
	BUG_ON(fdt->fd[fd] != NULL);
	rcu_assign_pointer(fdt->fd[fd], file);
	rcu_read_unlock_sched();
}
```
忽略其它代码，操作只有一个 `rcu_assign_pointer(fdt->fd[fd], file)` 相当于给指定索引赋值，将 `file` 关联到进程打开文件表。

###小结
暂时只分析上层，底层 `slab` 看不懂，冲突检测及 `smp` 也不用看。至此，`socket` 是如何创建的分析完。另外，源码里一切都是接口，函数指针赋来赋去，不结合上下文，很容易头晕。


