最近阅读 nginx, go 代码时经常看到结构体 cache line 对齐，比如 go timer 全局数组。周末 google、知呼 搜索了相关文档，梳理一下做个总结分享出来。

-----
### c语言数组
先看例子，代码很好理解。栈上创建一个二维数组 1024 * 1024 * 4 = 4M, 遍历数组初始化为 1，先行后列赋值，注释代码为先列后行。
```
#include<stdio.h>

int main(){
	int a[1024][1024];
	for (int i = 0; i < 1024; ++i){
		for(int j = 0; j < 1024;j++){
			a[i][j]=1;
			//a[j][i]=1;
		}
	}
	return 0;
}
```
这两种方式有什么差别么？先看执行时间
```
// a[i][j] = 1 先行后列
root@:~/dongzerun# gcc -std=c99 test.c && time ./a.out
real	0m0.013s
user	0m0.000s
sys	    0m0.012s
// a[j][i] = 1 先列后行
root@:~/dongzerun# gcc -std=c99 test.c && time ./a.out

real	0m0.039s
user	0m0.032s
sys	    0m0.004s
```
性能差距很明显，先行后列消耗 0.01s, 先列后行消耗 0.03s. 可以反复测试，调大 n 值，会发现性能差距更明显。为什么会有这个差距呢？gdb 看一下 c 语言二维数组的内存布局：
```
(gdb) p/x &a[0][0] # 第一行第一个元素地址
$1 = 0x7fffffbfe1f0
(gdb) p/x &a[1][0] # 第二行第一个元素地址
$2 = 0x7fffffbff1f0
(gdb) p/x &a[0][1023] # 第一行最后一个元素地址
$3 = 0x7fffffbff1ec
```
可以看到 a[1][0] 与 a[0][0]  地址相减是 4096，也就是 1024 个 int，a[0][1023] 和 
a[1][0] 地址相减是 4，也就是 1 个 int. 那么由此推断，c 语言二维数组内存布局：按行顺序存储，完全可以转换成一维数组访问。
```
root@:~/dongzerun# gcc -std=c99 test.c && perf stat -e L1-dcache-load-misses ./a.out

 Performance counter stats for './a.out':

         1,217,476      L1-dcache-load-misses

       0.042906978 seconds time elapsed

root@:~/dongzerun# gcc -std=c99 test.c && perf stat -e L1-dcache-load-misses ./a.out

 Performance counter stats for './a.out':

           162,461      L1-dcache-load-misses

       0.010097747 seconds time elapsed

```
使用 perf 来查看程序运行时的 cache miss 信息。先列后行，产生了大量的 cpu cache miss, 遍历时不得不从内存中加载数据，性能自然下降很多。cache 的存在有两个核心约束：空间局部性和时间局部性，在很短时间内，访问一个资源时，她附近的资源很可能也会被访问。

### 为什会有 cpu cache
广义来讲，为了解决快速设备和慢速设备不匹配的问题，计算机系统结构的每个层面都有 cache. 做为 DBA, 肯定知道，Raid 卡如果带有 Cache 那么磁盘写入速度相当快，所以阵列卡成为了数据库的标配。那么同样，内存访问速度相比 cpu 时钟周期，就成了慢速度备。
![Numbers every one should know](https://upload-images.jianshu.io/upload_images/659877-4e54427776c44d0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
L1 cache 访问需要 0.5ns, 但是主存 Mainmemory 居然 100ns, 慢了几个数量级。

### 如何查看 cpu cache
测试机器为公司服务器，DELL PowerEdge R720xd. 查看 lscpu
```
......
CPU(s):                32 # 32 个processors
On-line CPU(s) list:   0-31
Thread(s) per core:    2 # 每个 core 2 超线程
Core(s) per socket:    8 # 每个 cpu 8 核心
Socket(s):             2 #双 cpu
NUMA node(s):          2
Vendor ID:             GenuineIntel
CPU family:            6
......
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              20480K
NUMA node0 CPU(s):     0,2,4,6,8,10,12,14,16,18,20,22,24,26,28,30
NUMA node1 CPU(s):     1,3,5,7,9,11,13,15,17,19,21,23,25,27,29,31
```
Intel(R) Xeon(R) CPU E5-2650 v2 @ 2.60GHz，双 cpu, 每个 8 cores, 每个 core 2 个超线程(Hyper Thread)，所以 OS 会看到 2 * 8 * 2 = 32 个 processor. 当前 cpu 有 L3 cache、L2 cache、L1i 指令cache、L1d 数据cache
### cache 架构
那么具体 cpu 有多少 cache, 并且这些 cache 是如何分布的呢？
```
root@:~# ls /sys/devices/system/cpu/cpu0/cache
index0  index1  index2  index3
root@:~# ls /sys/devices/system/cpu/cpu0/cache/index0/
coherency_line_size  number_of_sets           shared_cpu_list  size  ways_of_associativity
level                physical_line_partition  shared_cpu_map   type
root@:~# ls -l /sys/devices/system/cpu/cpu0/cache/index0/
total 0
-r--r--r-- 1 root root 4096 Jun 24 17:06 coherency_line_size
-r--r--r-- 1 root root 4096 Jun 16 00:00 level
-r--r--r-- 1 root root 4096 Jun 24 17:06 number_of_sets
-r--r--r-- 1 root root 4096 Jun 24 17:06 physical_line_partition
-r--r--r-- 1 root root 4096 Jun 24 17:06 shared_cpu_list
-r--r--r-- 1 root root 4096 Jun 16 00:00 shared_cpu_map
-r--r--r-- 1 root root 4096 Jun 16 00:00 size
-r--r--r-- 1 root root 4096 Jun 16 00:00 type
-r--r--r-- 1 root root 4096 Jun 24 17:06 ways_of_associativity
```
系统目录 /sys/devices/system/cpu 有 32 个 cpu processor, 对应 cache 目录存放相关配置信息，index0, index1 分别是 L1 的数据和指令 cache, index2, index3 分别对应 L2 和 L3 cache. 通过查看得知：服务器有 2 颗 cpu, 每颗有一个 20M L3, 被 8 个 core (2 个超线程)共享，每个核有一个 256K L2, 一个 32K L1i, 一个 32K L1d 
```
coherency_line_size: cache line size 对齐大小，64 字节，cache line 是 cpu cache 使用的最小单元
number_of_sets: 一共多少个 cache line set 集合，L2 是 512 
shared_cpu_list: 被哪些 cpu processor 共享，id 列表
shared_cpu_map: 同上，十六进制表示
size: 对应 cache 大小
ways_of_associativity: 多少路关联
```
比如一个 cpu0 所使用的 L2 cache 
```
root@:~# cat /sys/devices/system/cpu/cpu0/cache/index2/number_of_sets
512
root@:~# cat /sys/devices/system/cpu/cpu0/cache/index2/shared_cpu_list
0,16
root@:~# cat /sys/devices/system/cpu/cpu0/cache/index2/size
256K
root@:~# cat /sys/devices/system/cpu/cpu0/cache/index2/ways_of_associativity
8
root@:~# cat /sys/devices/system/cpu/cpu0/cache/index2/coherency_line_size
64
```
那么有如下公式成立：number_of_sets * coherency_line_size * ways_of_associativity = size 那么问题来了，这个所谓的 ways_of_associativity 多路关联是什么意思？
### cache 与 内存的映射关系
现代 cpu 中，cache 都划分成以 cache line (cache block) 为单位，在 x86_64 体系下一般都是 64 字节，cache line 是操作的最小单元。***程序即使只想读内存中的 1 个字节数据，也要同时把附近 63 节字加载到 cache 中，如果读取超个 64 字节，那么就要加载到多个 cache line 中***。由于局部性原因，这样 cache 使用最为高效，但同时也会带来 cache contention 竞争问题。
![cache line.png](https://upload-images.jianshu.io/upload_images/659877-c3ac6d8f153e6e17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
那么问题来了，访问一个内存地址是如何映射到 cache 中的呢？首先我们的程序都有自己的内存地址空间，也就是说，存在一个虚拟地址到物理地址的转换，需要查看 page table, 这个翻译很浪费时间，会把这种映射放到 TLB cache 中。

>那么TLB和Cache有什么关系呢？可以说TLB命中是Cache命中的基本条件。TLB不命中，会更新TLB项，这个代价非常大，Cache命中的好处基本都没有了。在TLB命中的情况下，物理地址才能够被选出，Cache的命中与否才能够达成

内存和 cache 的映射主要有三种：
* 直接映射(Direct Mapping) : 故名思义，内存地址在 cache 中都有唯一的对应关系，比如按地址取余，算法简单。但是问题很大，cache 命中率极低。因为 cache 有空间局部性和时间局部性，同一时间数据都在附近，某些 cache line 会频繁换入换出。

* 全相联映射(Fully Associative Mapping): 没有地址对应关系，所有数据都可以存到 cpu cache，但是问题也比较大，很容易被大量数据污染，可以类比 MySQL Innodb Buffer Pool, 只有一个 Pool 的版本，很容易被 Batch 操作将热数据从 Pool 中刷掉，在后来的 5.6 才引入多个 Pool

* n路组相联映射(n-ways Set-Associative mapping): 一个 cache 划分成 n 个组，每个组有一定数量的 cache line, 然后把内存按地址映射，某一块物理内存映射到某个组 set 

查看系统 ways_of_associativity 得知，当前 Xeon E5-2650 cpu 的 L1、L2 均为 8 组，L3 20 组

### cache 一致性问题：
当数据被多个 cpu processors 共享时，自然会有一致性问题。L1、L2 均为 cpu  独占，同一份数据，被多个 cache 缓存，那么一个 cpu modify 修改数据，同时要 invalid 其它 cpu cache 数据，造成了其它 cpu cache miss 和一致性问题。行业内比较通用的做法是通过MESI协议来保证Cache的一致性：
* M (Modify) 这行数据有效，数据被修改了，与内存中的不一致，数据只存在于本 cache 中
* E (Exclusive) 这行数据有效，数据和内存中的一致，并且只存在于本 cache 中
* S (Shared) 这行数据有效，数据和内存中的一致，存在于多份 cache 中
* I (Invalid) 无效，读取操作会触发 cache miss
下图为 MESI 状态机
![MESI 状态机](https://upload-images.jianshu.io/upload_images/659877-bb1e08bff15c02db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
仔细看一下，真的很复杂，软件开时也不多使用多级 cache，软件很多技术都是硬件玩剩下的概念。

### cache 竞争问题: true sharing 与 false sharing
现代多核处理器，每个 core 都有自己的 cache, 比如这次测试的 Xeon E5-2650, 每个 core 拥有自己的 L1、L2, 被每个 core 的 2 个超线程共享(Hyper Thread). 这些私有 cache 就是产生竟争问题的根源。

程序一般都是多进程运行，每个进程绑定某一个 core, 典型如 Nginx master-worker 架构。当这些进程，读取同一份数据，或是数据恰巧相邻，那么 cpu 就会 copy 这些数据到自己的 cache. 当某个进程修改这份数据时，就会使其它 cpu cache 内容失效，根据 MESI 协义来 Invalid. 后续其它进程想要访问同一份数据，就会触发 cache miss, 重新从内存装载，非常耗时。大量的 cache miss 严重降低多并发程序性能。

cache contention 分为两种类型 true sharing 和 false sharing
* True sharing : 多核竞争的，是同一份将要访问的数据，比如全局变量的修改。
```
total := 0
for i := 0; i < 32; i++ {
	go func() {
		for n := 0; n < 1000000; n++ {
			total = i
		}
	}()
}
```
衡量是否有 cpu cache contention 问题的标准在于，程序性能是否随着 cpu core 的增加而线性扩展。如果不升反降，那就需要考滤优化了。
* False sharing : 多核访问的不是同一份数据，但是因为内存相邻或映射，恰巧加载到了同一个 cache line. 这个解决方案一般是加 padding 数据，使得共享数据间隔超过一个 cache line, 参考 go 1.10 定时器数组的实现：
```
var timers [timersLen]struct {
	timersBucket

	// The padding should eliminate false sharing
	// between timersBucket values.
	pad [sys.CacheLineSize - unsafe.Sizeof(timersBucket{})%sys.CacheLineSize]byte
}
```

但是不当的内存对齐，也会产生问题，可以参考 [How Misaligning Data Can Increase Performance 12x by Reducing Cache Misses](http://danluu.com/3c-conflict/#fn3)

### 小结
业务 RD 一般不关心底层原理，api 请求 1s 的比比皆是。但是做基础组件研发，尤其是网关、内核、负载均衡，必须要关注。现代 cpu 架构一直在演进，smp, numa 影响程序性能的因素很多很多。

写的有点杂，如有出入，还请指正~~

###参考文章
[关于CPU Cache -- 程序猿需要知道的那些事](https://cenalulu.github.io/linux/all-about-cpu-cache/)
[cpu cache 入门](https://zhuanlan.zhihu.com/p/33663745)
[cache 是如何组织工作的](https://zhuanlan.zhihu.com/p/31859105)
[Cache为什么有那么多级？为什么一级比一级大？是不是Cache越大越好？](https://zhuanlan.zhihu.com/p/32058808)
[Dynamic Cache Contention Detection in Multi-threaded Applications](https://www.researchgate.net/publication/221137799_Dynamic_Cache_Contention_Detection_in_Multi-threaded_Applications)
[L1 L2 L3 究竟在哪里？](https://zhuanlan.zhihu.com/p/31422201)
[X86高性能编程之洞悉CPU Cache](http://rdc.hundsun.com/portal/article/771.html)
[CPU Cache Line伪共享问题的总结和分析](https://mp.weixin.qq.com/s/y1NSE5xdh8Nt5hlmK0E8Og)

