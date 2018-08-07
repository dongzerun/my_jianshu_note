霸爷博客，干货满满。有两篇文章现在还记得，[《Linux下如何知道文件被哪个进程写》](http://blog.yufeng.info/archives/2581#more-2581)和[《巧用Systemtap注入延迟模拟IO设备抖动》](http://blog.yufeng.info/archives/2935)，周末突然想起来，发现能看懂了:)

### 什么是 systemtap
Systemtap is a tool that allows developers and administrators to write and reuse simple scripts to deeply examine the activities of a live Linux system. Data may be extracted, filtered, and summarized quickly and safely, to enable diagnoses of complex performance or functional problems.

我们一般调试程序，业务程序加日志，打 log, 基本能满足需求。再不济，使用 strace、lsof、perf 足够看到性能瓶劲，可以参考我 [go gc](https://www.jianshu.com/p/0791c35d3609) 的文章。但是系统编程，就不能狂打日志，而且很多调用栈都处于 kernel space，那么普通的调试手段就显得捉襟见肘了。

此时 systemtap 就能派上用场，他会在内核函数加 probe 探针，对 kernel space 函数调用进行统计汇总，甚至可以对其进行干预。但是对 user space 调试支持不是很好。
### 安装
本机环境：DELL R720, Ubuntu 14.04 3.19.0-25-generic x86_64 
```
apt-get  install systemtap systemtap-client systemtap-common systemtap-runtime systemtap-server -y
```
对于 centos 系统也一样，yum install 即可。此时还要使用执行 stap-prep 安装缺失的内核镜像调试包。比如我的就是 
```
linux-image-3.19.0-25-generic-dbgsym_3.19.0-25.26~14.04.1_amd64.ddeb
```
遇到缺什么包直接安装，或是从网上下载。systemtap 最好不要用源码安装，涉及内核的包都很恶心，***版本必须匹配***，可以用 uname -r 查看。

###1. Linux下如何知道文件被哪个进程写
有个文件，不定期的被修改，如果只是瞬间的写入，lsof 也没有办法，就算定期执行，也可能有缺失。那么此时 systemtap 就该大显身手了，先上代码：
```
#!/usr/bin/env stap

probe vfs.write, vfs.read
{
  if (@defined($file->f_path->dentry)) {
	dev_nr=$file->f_path->dentry->d_inode->i_sb->s_dev
	inode_nr = $file->f_path->dentry->d_inode->i_ino
	} else {
	dev_nr=$file->f_dentry->d_inode->i_sb->s_dev
	inode_nr = $file->f_dentry->d_inode->i_ino
	}
  # dev and ino are defined by vfs.write and vfs.read
  if (dev_nr == MKDEV($1,$2) && inode_nr==$3){
    printf ("%s(%d) %s 0x%x/%u\n",execname(), pid(), ppfunc(), dev_nr, inode_nr)
   }
}

probe timer.ms(10) {
	exit()
}
```
语法类似 awk 代码很简单，probe 定义探针，后面紧跟着探测点，可以是具体的函数名，支持 * 匹配，大括号定义探针触发动作。

file 是函数 vfs.read, vfs.write 的参数，dev_nr,inode_nr 根据 file 结构体获取设备号和 inode，探测点是针对内核函数的，所以可以获取函数所有参数。

execname 执行 vfs.write 或 vfs.read 程序名

pid 执行 vfs.write 或 vfs.read 进程号

ppfunc 是控测点函数名，这个内置函数在不同版本可能不一样，比如霸爷文章里是 probefuc

#### 1.1 开终端执行 dd
打开终端执行 dd 不断的写入数据，并查看文件 inode 号
```
dd if=/dev/zero of=test.dat
```
```
stat -c "%i" /disk1/test.dat
ls -al /dev/sdb1
```
这里 /dev/sdb1 是挂载在 /disk1 目录下的设备

#### 1.2 执行 stap 探测
```
stap -v inodewatch.stp 8 17 15
Pass 1: parsed user script and 95 library script(s) using 84976virt/30204res/5152shr/25852data kb, in 200usr/0sys/456real ms.
Pass 2: analyzed script: 3 probe(s), 7 function(s), 5 embed(s), 0 global(s) using 610884virt/195324res/12432shr/180716data kb, in 1810usr/290sys/3605real ms.
Pass 3: translated to C into "/tmp/stapJEOYcQ/stap_20c430109956cd1ffc28c7ceaf0aa2f1_6899_src.c" using 599240virt/188844res/8908shr/180712data kb, in 0usr/0sys/73real ms.
Pass 4: compiled C into "stap_20c430109956cd1ffc28c7ceaf0aa2f1_6899.ko" in 1840usr/320sys/4180real ms.
Pass 5: starting run.
dd(25763) vfs_write 0x800011/15
dd(25763) vfs_write 0x800011/15
dd(25763) vfs_write 0x800011/15
dd(25763) vfs_write 0x800011/15
dd(25763) vfs_write 0x800011/15
Pass 5: run completed in 0usr/40sys/724real ms.
```
stap 执行脚本需要 5 个步骤，解析脚本，分析，生成 c 代码，编绎成内核模块 ko 文件。最后执行模块，可以看到 dd 任务在写文件，调用 vfs_write

###2. 巧用Systemtap注入延迟模拟IO设备抖动
霸爷的这篇例子很有意思，systemtap 模拟磁盘 IO 抖动，对于一些存储系统，压测时可以试一下。原理还是很简单的，在 vfs_write, vfs_read 时 sleep 一小段时间即可，时间可以随机。先上代码
```
cat inject_ka.stp
global inject, ka_cnt

probe procfs("cnt").read {
  $value = sprintf("%d\n", ka_cnt);
}
probe procfs("inject").write {
  inject= $value;
  printf("inject count %d, ka %s", ka_cnt, inject);
}

probe vfs.read.return,
      vfs.write.return {
  if (@defined($file->f_path->dentry)) {
	dev_nr=$file->f_path->dentry->d_inode->i_sb->s_dev
	inode_nr = $file->f_path->dentry->d_inode->i_ino
	} else {
	dev_nr=$file->f_dentry->d_inode->i_sb->s_dev
	inode_nr = $file->f_dentry->d_inode->i_ino
	}

  if ($return &&
      dev_nr == MKDEV($1,$2) &&
      inject == "on\n")
  {
#	printf("dev %x func: %s\n", dev_nr, ppfunc())
    ka_cnt++;
    udelay($3);
  }
}

probe begin{
  println("ik module begin:)");
}
```
代码有些略长，先看探针 probe vfs.read.return, vfs.write.return 表示在退出前执行探针代码，判断 dev_nr 是不是目标设备，并且打开了 ineject, 如果打开，那么 udelay 一小段时间。

至于另外两个探针，procfs("cnt"), procfs("inject") 读取 /proc/systemtap 时触发，修改全局变量 inject 来决定是否打开 IO 注入。

####2.1 执行代码
这个脚本执行可能会遇到 vfs_lookup_path 报错，很恶心，我把 procfs.c 升级了一个版本，并注释掉 vfs_lookup_path 部份才解决 ... 
```
stap -DMAXSKIPPED=9999 -m ik -g inject_ka.stp 8 17 400
ik module begin:)
```
8，17 表示磁盘设备号，400表示 udelay 时间，此时脚本阻塞在这里，并没有开始执行 IO 注入。打开另一个终端执行注入，持续 30 秒。
```
echo on| tee /proc/systemtap/ik/inject  && sleep 30 && echo off| tee /proc/systemtap/ik/inject
```
此时可以看到 stap 有输出。

####2.2 测试磁盘性能
简单的使用 dd 来测试 IO 延迟对顺序写的影响

注入前
```
dd if=/dev/zero of=test.dat  bs=8k count=1000000
1000000+0 records in
1000000+0 records out
8192000000 bytes (8.2 GB) copied, 34.8372 s, 235 MB/s
```
注入后
```
dd if=/dev/zero of=test.dat  bs=8k count=1000000
1000000+0 records in
1000000+0 records out
8192000000 bytes (8.2 GB) copied, 79.5475 s, 103 MB/s
```
可以看到 dd 性能下降很大，通过调整  udelay 时间可以模拟不同延迟下的性能。如果是随机的，或是符合正态分布的可能更好。

###小结
[官网](https://sourceware.org/systemtap/examples/) 有很多 systemtap 使用例子和介绍，还可以抓网络协义栈，性能很强大。同时需要有一定的内核功底，至少要知道探针埋在哪里，openresty 大量使用 systemtap 进行调试，可以参考学习。

另外，安装是个很大问题，一定要注意版本，太新也不可以，ubuntu 系统 apt 源是 2.3，尝试过源码安装高版本，各种报错。

