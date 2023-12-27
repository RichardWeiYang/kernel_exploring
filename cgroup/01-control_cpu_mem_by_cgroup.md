刚开始接触cgroup时，第一个任务就是用cgroup限制cpu/mem的使用。可能这也是大多数人接触到cgroup时，需要做的任务。

# 使用cgroup限制进程能运行的cpu

在这个例子中，我们探寻一下如何用cgroup限制进程能够在哪些cpu上调度。在云计算场景中，我们通常会用这个功能来隔离不同的用户/业务，保证对应资源的使用情况。

这个过程分为两步：

  * 创建一个限制了cpu调度的cgroup
  * 将指定进程加入到该cgroup中

## 新建限制cpu的cgroup

```
mkdir /sys/fs/cgroup/cpuset/test
echo "0-1" > /sys/fs/cgroup/cpuset/test/cpuset.cpus
```

这样就创建了一个cgroup，并且限制了该cgroup中的进程只能运行在0-1号cpu上。

## 将指定进程加入cgoup

```
echo $pid > /sys/fs/cgroup/cpuset/test/cgroup.procs
```

这么看坑你还是有点空，我们来看一个实际的例子

## 完整例子

```
mkdir /sys/fs/cgroup/cpuset/test

echo "0-1" > /sys/fs/cgroup/cpuset/test/cpuset.cpus
echo 0 > /sys/fs/cgroup/cpuset/test/cpuset.mems

# echo current shell process number to cgroup.procs
# so all child will inherit this attribute
echo $$ > /sys/fs/cgroup/cpuset/test/cgroup.procs

# stress on 4 cpus
$ stress -c 4

# while only 2 cpu is working
$ top
```

正常情况下，stress -c 4会使用4块cpu。但是因为将测试进程放进了cgroup，所以只有两个cpu在实际使用。

# 使用cgroup限制内存使用

在这个例子中，我们探寻一下如何用cgroup限制进程能够使用的内存。在云计算场景中，我们通常会用这个功能来限制用户/业务的内存，保证对应资源的使用情况，避免有恶意用户/业务抢占资源。

这个过程分为两步：

  * 创建一个限制内存使用的cgroup
  * 将指定进程加入到该cgroup中

## 创建限制内存使用的cgroup

```
mkdir /sys/fs/cgroup/memory/test

# limit 4M
echo 4M > /sys/fs/cgroup/memory/test/memory.limit_in_bytes
```

## 将指定进程加入cgoup

```
echo $pid > /sys/fs/cgroup/cpuset/test/cgroup.procs
```

这一步完全一样。。。

## 完整例子

以例服人

```
# create a cgroup with memory limitation
mkdir /sys/fs/cgroup/memory/test
# limit 4M
echo 4M > /sys/fs/cgroup/memory/test/memory.limit_in_bytes
# disable swap, otherwise it will use swap to get more "memory"
echo 0 > /sys/fs/cgroup/cpuset/rabbit/memory.swappiness

# put ourself into this cgroup
echo $$ > /sys/fs/cgroup/cpuset/rabbit/cgroup.procs

# prepare a program eats memory
cat eat_mem.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define MB (1024 * 1024)

int main(int argc, char *argv[])
{
    char *p;
    int i = 0;
    while(1) {
        p = (char *)malloc(MB);
        memset(p, 0, MB);
        printf("%dM memory allocated\n", ++i);
        sleep(1);
    }

    return 0;
}

# run to trigger oom
./eat_mem
```

上面的例子中，有一点值得注意的是这个cgroup的swap被关掉了。否则的话，当进程内存不够，内核会将进程的内存交换到磁盘上。虽然进程的内存实际上是被限制了，但是无法触发oom观察到实验结果。

好了，希望上面两个例子能够让你亲眼看到cgroup的操作和效果。接下来我们就开始探索其中的奥秘了～
