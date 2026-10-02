先来看看hugetlb都有哪些使用方式。

# 内核配置

首先内核中需要配置了hugetlb，启动后才能使用。主要的配置有：

```
CONFIG_HUGETLBFS=y
CONFIG_HUGETLB_PAGE=y
```

如果启动后，目录/sys/kernel/mm/hugepages/存在，则表示基本的配置已经生效。

# 预留/分配

hugetlb是一个基于内存的存在，所以在真正使用前，要从系统内存中“挖”出这么一段内存，而且是要连续的。

有两种方式

  * 启动参数
  * 动态分配

因为要求内存是连续的大页，所以第一种在实际使用中更常见。

## 启动参数

启动时，添加内核参数

```
hugepagesz=2M hugepages=1024
hugepagesz=1G hugepages=4
```

前者是页面大小，后者是预留的页面数量。

## 动态分配

系统启动后也可以预留或者调整大页的数量。

```
# 2MB
echo 1024 > /proc/sys/vm/nr_hugepages

# 1GB
echo 4 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages
```

甚至可以指定numa节点

```
echo 4 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
```

# 观察状态

不论使用上述哪种方式，都可以通过grep "^Huge" /proc/meminfo来查看当前的使用状态。

```
grep "^Huge" /proc/meminfo
HugePages_Total:    1024
HugePages_Free:     1024
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:         2097152 kB
```

但是这个信息有点“老”， 其中除了Hugetlb，都描述的是默认大小2048kB的。

要不用尺寸hugetlb的状态信息，要到对应的目录下去看

```
# 2M
cat /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
cat /sys/kernel/mm/hugepages/hugepages-2048kB/free_hugepages

# 1G
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages
cat /sys/kernel/mm/hugepages/hugepages-1048576kB/free_hugepages
```

另外还能从numa的角度去观察

```
/sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
/sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages
```

所以观察系统所有hugetlb占用内存的情况时，需要注意这点。

# 使用方式

在使用上，我没有想到也能有两种方式。

  * 匿名映射
  * 文件映射

但最后实际上背后都是文件支持。

## 匿名映射

下面这个操作和匿名私有映射看上去很像，不过内核后端会创建一个/anon_hugepage作为支持。

```
	mmap(NULL, pmd_pagesize, PROT_READ | PROT_WRITE,
		  MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
```

## 文件映射

文件映射稍微麻烦点，需要先挂载一个hugetlbfs的目录。

```
 sudo mkdir -p /mnt/huge
 sudo mount -t hugetlbfs -o pagesize=2M,size=128M,mode=0777 none /mnt/huge
```

然后代码中在这个目录下打开这个文件。

```
	char file_name[] = "/mnt/huge/demo";

	fd = open(file_name, O_CREAT | O_RDWR, 0600);
	if (fd < 0) {
		perror("open");
		return;
	}

	if (ftruncate(fd, pmd_pagesize) < 0) {
		perror("ftruncate");
		goto trunc_error;
	}

	hp = mmap(NULL, pmd_pagesize, PROT_READ | PROT_WRITE,
		  MAP_SHARED, fd, 0);
```

好了，总体上使用还是简单的。
