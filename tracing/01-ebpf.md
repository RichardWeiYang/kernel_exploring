eBPF -- extended Berkeley Packet Filter 是一种能在内核中抓包的机制。（嗯，感觉自己都不认可这句话。）

# 整体架构

从架构上来看大概是这个样子的。

```
                            Kernel
                             +------------------------------+
                             |                              |
                             | .............                |
                             | . Map       .        * * * * |
                             | .............         BPF    |
                             |      ^               *     * |
                             |      |                       |
    +------------+   bpf()   |      v               *     * |
    | Userspace  |  syscall  | +--------------+      Prog   |
    |            |< - - - - >| | bpf_map::ops |< - >*     * |
    |  Process   |           | +--------------+             |
    +------------+           |                      * * * * |
                             +------------------------------+
```

在内核中有两部分：

  * 叫map的数据
  * 叫prog的代码

整个机制的运行过程为：

> 用户态将prog(包含了map定义)注入到内核中，用户态和内核态(prog)通过各自的api来访问、修改map数据。最终用户态将相关信息打印获得输出。

# 例子及使用

有效学习的途径之一就是通过例子，而在内核源码中就有很多eBPF的例子代码供我们学习。他们在samples/bpf目录下。

## 编译

虽然找到了例子，但是要跑起来还要再花些功夫。在ebpf例子的编译上和普通的程序编译略有不同。

首先，为了能使ebpf发挥效能，还需要内核这边的一些配置。比如我在网上找到的这么些内核配置的要求。

```
  CONFIG_BPF=y
  CONFIG_BPF_SYSCALL=y
  # [optional, for tc filters]
  CONFIG_NET_CLS_BPF=m
  # [optional, for tc actions]
  CONFIG_NET_ACT_BPF=m
  CONFIG_BPF_JIT=y
  # [for Linux kernel versions 4.1 through 4.6]
  CONFIG_HAVE_BPF_JIT=y
  # [for Linux kernel versions 4.7 and later]
  CONFIG_HAVE_EBPF_JIT=y
  # [optional, for kprobes]
  CONFIG_BPF_EVENTS=y

  CONFIG_NET_SCH_SFQ=m
  CONFIG_NET_ACT_POLICE=m
  CONFIG_NET_ACT_GACT=m
  CONFIG_DUMMY=m
  CONFIG_VXLAN=m
```

除此之外，执行make之前还要安装两个编译工具， clang/llvm。因为当前的gcc不支持生成eBPF的字节码。

在Fedora上运行以下命令安装 clang/llvm。

> yum install clang llvm

好了，准备好了一切，我们就可以编译ebpf的例子了。

```
cd samples/bpf
make
```

每一个可执行文件就可以拿来做实验了。

## 示例 -- sockex1

此时我们挑一个看着比较简单的例子来学习以下，sockex1这个例子看上去代码比较绍，那就挑这个吧。

这个例子有两个文件组成： sockex1_user.c sockex1_kern.c。后者看上去短，那就先看kern.c。

```
struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__type(key, u32);
	__type(value, long);
	__uint(max_entries, 256);
} my_map SEC(".maps");

SEC("socket1")
int bpf_prog1(struct __sk_buff *skb)
{
	int index = load_byte(skb, ETH_HLEN + offsetof(struct iphdr, protocol));
	long *value;

	if (skb->pkt_type != PACKET_OUTGOING)
		return 0;

	value = bpf_map_lookup_elem(&my_map, &index);
	if (value)
		__sync_fetch_and_add(value, skb->len);

	return 0;
}
```

如之前整体架构中所述，eBPF由两部分组成： 作为数据的map, 作为代码的prog。我们这个例子中数据和代码各一份。

虽然有很多函数看不懂，但是大致的意思就是在my_map中做了某个查找，符合条件时增加skb->len。那我们再来看看用户态都做了些什么。

```
int main(int ac, char **argv)
{
  [...]

	snprintf(filename, sizeof(filename), "%s_kern.o", argv[0]);

	if (bpf_prog_load(filename, BPF_PROG_TYPE_SOCKET_FILTER,
			  &obj, &prog_fd))
		return 1;

	map_fd = bpf_object__find_map_fd_by_name(obj, "my_map");

	sock = open_raw_sock("lo");

	assert(setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, &prog_fd,
			  sizeof(prog_fd)) == 0);

	f = popen("ping -4 -c5 localhost", "r");
	(void) f;

	for (i = 0; i < 5; i++) {
		long long tcp_cnt, udp_cnt, icmp_cnt;
		int key;

		key = IPPROTO_TCP;
		assert(bpf_map_lookup_elem(map_fd, &key, &tcp_cnt) == 0);

		key = IPPROTO_UDP;
		assert(bpf_map_lookup_elem(map_fd, &key, &udp_cnt) == 0);

		key = IPPROTO_ICMP;
		assert(bpf_map_lookup_elem(map_fd, &key, &icmp_cnt) == 0);

		printf("TCP %lld UDP %lld ICMP %lld bytes\n",
		       tcp_cnt, udp_cnt, icmp_cnt);
		sleep(1);
	}

	return 0;
}
```

其中大致的流程为：

  * 加载sockex1_kern.o
  * 打开了loop的socket，并且和加载的prog做了关联
  * 接着在my_map中查找协议对应的数据

看着好像不难，但是依然一头雾水。不着急，我们一点点来。

# 里应外合

从架构和上面的例子中看出，eBPF包含两部分，内核态的prog和用户态的程序。两者之间如何通信则成了我心中的一块疑团。

好在踏(误)破(打)铁(误)鞋(撞)， 也算是找到了些端倪。

## 内核中的bpf helper

在插入内核的prog中，我们可以看到这样的函数

> bpf_map_lookup_elem()

这个就是在操作map数据。因为这段代码需要在内核中调用，那么在内核中这个函数又在哪里呢？对了，这个东西就叫BPF_CALL。

```
BPF_CALL_2(bpf_map_lookup_elem, struct bpf_map *, map, void *, key)
{
	WARN_ON_ONCE(!rcu_read_lock_held());
	return (unsigned long) map->ops->map_lookup_elem(map, key);
}
```

怎么样，感觉也还行吧。

## 用户态的libbpf

内核态的函数在内核中直接定义了，还比较好找。但是在用户态的程序又如何和内核中的map打交道？

这个东西叫 libbpf， 在内核源码中tools/lib/bpf目录下。当我们仔细看samples/bpf目录下的Makefile，我们可以看到用户态程序在链接时使用了这个库。

```
# Libbpf dependencies
LIBBPF = $(TOOLS_PATH)/lib/bpf/libbpf.a

TPROGS_LDLIBS			+= $(LIBBPF) -lelf -lz

# Link an executable based on list of .o files, all plain c
# tprog-cmulti -> executable
quiet_cmd_tprog-cmulti	= LD  $@
      cmd_tprog-cmulti	= $(CC) $(tprogc_flags) $(TPROGS_LDFLAGS) -o $@ \
			                     $(addprefix $(obj)/,$($(@F)-objs)) \
			                     $(TPROGS_LDLIBS) $(TPROGLDLIBS_$(@F))
$(tprog-cmulti): $(tprog-cobjs) FORCE
        $(call if_changed,tprog-cmulti)
$(call multi_depend, $(tprog-cmulti), , -objs)
```

知道了这点，我们就来看看函数bpf_map_lookup_elem在用户态中它的定义：

```
  bpf_map_lookup_elem()
      sys_bpf(BPF_MAP_LOOKUP_ELEM, &attr, sizeof(attr));
          syscall(__NR_bpf, cmd, attr, size);
```

啊，原来如此。用户态是通过bpf这个系统调用来和内核通信的。再回过去看看整体架构中的那个图，是不是又发现了什么？

让我们还是进到bpf系调用看一下：

```
SYSCALL_DEFINE3(bpf, int, cmd, union bpf_attr __user *, uattr, unsigned int, size)
{
	union bpf_attr attr;
	int err;

  ...

	switch (cmd) {
	case BPF_MAP_CREATE:
		err = map_create(&attr);
		break;
	case BPF_MAP_LOOKUP_ELEM:
		err = map_lookup_elem(&attr);
		break;
	case BPF_MAP_UPDATE_ELEM:
		err = map_update_elem(&attr);
		break;
	case BPF_MAP_DELETE_ELEM:
		err = map_delete_elem(&attr);
		break;

  ...

```

从函数原型和实现上我们可以看到，sys_bpf为了满足不同的需求，按照cmd类型做了分类处理。

比如我们在用户态执行的bpf_prog_load，在这里对应的就是BPF_MAP_CREATE；bpf_map_lookup_elem对应的是BPF_MAP_LOOKUP_ELEM。

沿着这个思路，我们是不是就能找到用户态中如何创建，访问，修改内核中的map数据了呢？Bingo!

# Where Where Where?

到这里看似我们已经了解了一切，但是突然有个非常关键的问题冒了出来 -- 添加到内核中的prog究竟什么时候会被触发？

比如在samples/bpf/cpustat_kern.c添加的这么一段prog

```
SEC("tracepoint/power/cpu_frequency")
int bpf_prog2(struct cpu_args *ctx)
```

随之而来的有两个问题：

  * 这个prog是在内核什么地方会被调用？
  * 为什么传进来的参数是 struct cpu_args 这样的结构体？

当然现在这个整体的疑惑我还是没有完全打开，不过找到了一把小小的钥匙。这个钥匙还是在对应的用户态程序中。

让我们从用户态samples/bpf/cpustat_user.c中的 load_bpf_file()开始

```
  load_bpf_file(file)
      do_load_bpf_file(file, NULL)
          load_and_attach()
              is_tracepoint = strncmp(event, "tracepoint/", 11) == 0;
              if (is_tracepoint) {
                  strcpy(buf, DEBUGFS);
                  strcat(buf, "events/");
                  strcat(buf, event);
                  strcat(buf, "/id");
              }
              ...
              efd = sys_perf_event_open(&attr, -1/*pid*/, 0/*cpu*/, -1/*group_fd*/, 0);
              err = ioctl(efd, PERF_EVENT_IOC_ENABLE, 0);
              err = ioctl(efd, PERF_EVENT_IOC_SET_BPF, fd);
```

具体细节还有很多，可以看到的是，SEC的定义影响了prog的加载过程。比如对tracepoint开头的SEC，加载时回去找到debugfs中对应的事件id。并通过后续一系列的操作将这个事件和prog联系起来。

对于tracepoint类型的bpf，当前找到的两个信息是：

  * 内核中通过函数trace_call_bpf()来调用到prog
  * 调用时的参数结构可以在 /sys/kernel/debug/tracing/events/event/format 文件中找到

好了，eBPF的基础学习就到这里。后续的内容已经超出了eBPF本身的范围，而是包括perf event在内的其他内核组件如何和eBPF互动的内容了。

到这里，先到这里。
