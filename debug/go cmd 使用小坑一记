先说下使用场景，某服务在每台服务器上启动 agent, 用户会在指定机器上执行任务，并将结果返回到网页上。执行任务由用户自定义脚本，一般也都是 shell 或是python，会不断的产生子进程，孙进程，直到执行完毕或是超时被 kill
***
最近发现经常有任务，一直处于运行中，但实际上己经超时被 kill，并未将输出写到系统，看不到任务的执行情况。登录机器，发现执行脚本进程己经杀掉，但是有子脚本卡在某个 http 调用。我司的网滥到无法直视，内网还有不通的 case，并且还有很多公网机器，再看下这个脚本，python requests 默认没有设置超时... 

总结一下现象：agent 用 go cmd 启动子进程，子进程还会启动孙进程，孙进程因某种原因阻塞。此时，如果子进程因超时被 agent kill 杀掉,  agent 却仍然处于 wait 状态。

### 复现 case
agent 使用 exec.CommandContext 启动任务，设置 ctx 超时 30s，并将结果写到 bytes.Buffer, 最后打印。简化例子如下：

```
func main() {
	ctx, cancelFn := context.WithTimeout(context.Background(), time.Second*30)
	defer cancelFn()
	cmd := exec.CommandContext(ctx, "/root/dongzerun/sleep.sh")
	var b bytes.Buffer
	cmd.Stdout = &b //剧透，坑在这里
	cmd.Stderr = &b
	cmd.Start()
	cmd.Wait()
	fmt.Println("recive: ", b.String())
}
```
这个是 sleep.sh，模拟子进程
```
#!/bin/sh
echo "in sleep"
sh /root/dongzerun/sleep1.sh
```
这是 sleep1.sh 模拟孙进程，sleep 1000 阻塞在这里
```
#!/bin/sh
sleep 1000
```

###现象
启动测试 go 程序，查看 ps axjf | less
```
ppid  pid   pgid
 2468 32690 32690 32690 ?           -1 Ss       0   0:00  \_ sshd: root@pts/6
32690 32818 32818 32818 pts/6    28746 Ss       0   0:00  |   \_ -bash
32818 28531 28531 32818 pts/6    28746 S        0   0:00  |       \_ strace ./wait
28531 28543 28531 32818 pts/6    28746 Sl       0   0:00  |       |   \_ ./wait
28543 28559 28531 32818 pts/6    28746 S        0   0:00  |       |       \_ /bin/sh /root/dongzerun/sleep.sh
28559 28560 28531 32818 pts/6    28746 S        0   0:00  |       |           \_ sh /root/dongzerun/sleep1.sh
28560 28563 28531 32818 pts/6    28746 S        0   0:00  |       |               \_ sleep 1000
```
等过了 30s，通过 ps axjf | less 查看
```
 2468 32690 32690 32690 ?           -1 Ss       0   0:00  \_ sshd: root@pts/6
32690 32818 32818 32818 pts/6    36192 Ss       0   0:00  |   \_ -bash
32818 28531 28531 32818 pts/6    36192 S        0   0:00  |       \_ strace ./wait
28531 28543 28531 32818 pts/6    36192 Sl       0   0:00  |       |   \_ ./wait
```
```
    1 28560 28531 32818 pts/6    36192 S        0   0:00 sh /root/dongzerun/sleep1.sh
28560 28563 28531 32818 pts/6    36192 S        0   0:00  \_ sleep 1000
```
通过上面的 case，可以看到 sleep1.sh 成了孤儿进程，被 init 1 认领，但是 28543 wait 并没有退出，那他在做什么？？？

### 分析

使用 strace 查看 wait  程序
```
epoll_ctl(4, EPOLL_CTL_DEL, 6, {0, {u32=0, u64=0}}) = 0
close(6)                                = 0
futex(0xc420054938, FUTEX_WAKE, 1)      = 1
waitid(P_PID, 28559, {si_signo=SIGCHLD, si_code=CLD_KILLED, si_pid=28559, si_status=SIGKILL, si_utime=0, si_stime=0}, WEXITED|WNOWAIT, NULL) = 0
卡在这里约 30s
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_KILLED, si_pid=28559, si_status=SIGKILL, si_utime=0, si_stime=0} ---
rt_sigreturn()                          = 0
futex(0x9a0378, FUTEX_WAKE, 1)          = 1
futex(0x9a02b0, FUTEX_WAKE, 1)          = 1
wait4(28559, [{WIFSIGNALED(s) && WTERMSIG(s) == SIGKILL}], 0, {ru_utime={0, 0}, ru_stime={0, 0}, ...}) = 28559
futex(0x9a0b78, FUTEX_WAIT, 0, NULL
```
通过 go 源码可以看到 go exec wait 时，会先执行 waitid，阻塞在这里，然后再来一次 wait4 等待最终退出结果。不太明白为什么两次 wait... 但是最后卡在了 futex 这里，看着像在等待什么资源？？？

打开 golang pprof, 再次运行程序，并 pprof 
```
	go func() {
		err := http.ListenAndServe(":6060", nil)
		if err != nil {
			fmt.Printf("failed to start pprof monitor:%s", err)
		}
	}()
```
```
curl http://127.0.0.1:6060/debug/pprof/goroutine?debug=2

goroutine 1 [chan receive]:
os/exec.(*Cmd).Wait(0xc42017a000, 0x7c3d40, 0x0)
	/usr/local/go/src/os/exec/exec.go:454 +0x135
main.main()
	/root/dongzerun/wait.go:32 +0x167
```
程序没有退出，并不可思义的卡在了 exec.go:454 行代码，查看 go1.9.6 原码：
```
// Wait releases any resources associated with the Cmd.
func (c *Cmd) Wait() error {
      ......
	state, err := c.Process.Wait()
	if c.waitDone != nil {
		close(c.waitDone)
	}
	c.ProcessState = state

	var copyError error
	for range c.goroutine {
        //卡在了这里
		if err := <-c.errch; err != nil && copyError == nil {
			copyError = err
		}
	}

	c.closeDescriptors(c.closeAfterWait)
    ......
	return copyError
}
```
通过源代码分析，程序 wait 卡在了 <-c.errch 获取 chan 数据。那么 errch 是如何生成的呢？
查看 cmd.Start 源码，go 将 cmd.Stdin, cmd.Stdout, cmd.Stderr 组织成 *os.File，并依次写到数组 childFiles 中，这个数组索引就对应子进程的0，1，2 文描术符，即子进程的标准输入，输出，错误。
```
	type F func(*Cmd) (*os.File, error)
	for _, setupFd := range []F{(*Cmd).stdin, (*Cmd).stdout, (*Cmd).stderr} {
		fd, err := setupFd(c)
		if err != nil {
			c.closeDescriptors(c.closeAfterStart)
			c.closeDescriptors(c.closeAfterWait)
			return err
		}
		c.childFiles = append(c.childFiles, fd)
	}
	c.childFiles = append(c.childFiles, c.ExtraFiles...)

	var err error
	c.Process, err = os.StartProcess(c.Path, c.argv(), &os.ProcAttr{
		Dir:   c.Dir,
		Files: c.childFiles,
		Env:   dedupEnv(c.envv()),
		Sys:   c.SysProcAttr,
	})
```
在执行 setupFd 时，会有一个关键的操作，打开 pipe 管道，封装一个匿名 func，功能就是将子进程的输出结果写到 pipe 或是将 pipe 数据写到子进程标准输入，最后关闭 pipe. 这个匿名函数最终在 Start 时执行
```
func (c *Cmd) stdin() (f *os.File, err error) {
	if c.Stdin == nil {
		f, err = os.Open(os.DevNull)
		if err != nil {
			return
		}
		c.closeAfterStart = append(c.closeAfterStart, f)
		return
	}

	if f, ok := c.Stdin.(*os.File); ok {
		return f, nil
	}

	pr, pw, err := os.Pipe()
	if err != nil {
		return
	}

	c.closeAfterStart = append(c.closeAfterStart, pr)
	c.closeAfterWait = append(c.closeAfterWait, pw)
	c.goroutine = append(c.goroutine, func() error {
		_, err := io.Copy(pw, c.Stdin)
		if skip := skipStdinCopyError; skip != nil && skip(err) {
			err = nil
		}
		if err1 := pw.Close(); err == nil {
			err = err1
		}
		return err
	})
	return pr, nil
}
```

重新运行测试 case，并用 lsof 查看进程打开了哪些资源
```
root@nb1963:~/dongzerun# ps aux |grep wait
root      4531  0.0  0.0 122180  6520 pts/6    Sl   17:24   0:00 ./wait
root      4726  0.0  0.0  10484  2144 pts/6    S+   17:24   0:00 grep --color=auto wait
root@nb1963:~/dongzerun#
root@nb1963:~/dongzerun# ps aux |grep sleep
root      4543  0.0  0.0   4456   688 pts/6    S    17:24   0:00 /bin/sh /root/dongzerun/sleep.sh
root      4548  0.0  0.0   4456   760 pts/6    S    17:24   0:00 sh /root/dongzerun/sleep1.sh
root      4550  0.0  0.0   5928   748 pts/6    S    17:24   0:00 sleep 1000
root      4784  0.0  0.0  10480  2188 pts/6    S+   17:24   0:00 grep --color=auto sleep
root@nb1963:~/dongzerun#
root@nb1963:~/dongzerun# lsof -p 4531
COMMAND  PID USER   FD   TYPE     DEVICE SIZE/OFF       NODE NAME
wait    4531 root    0w   CHR        1,3      0t0       1029 /dev/null
wait    4531 root    1w   REG        8,1    94371    4991345 /root/dongzerun/nohup.out
wait    4531 root    2w   REG        8,1    94371    4991345 /root/dongzerun/nohup.out
wait    4531 root    3u  IPv6 2005568215      0t0        TCP *:6060 (LISTEN)
wait    4531 root    4u  0000       0,10        0       9076 anon_inode
wait    4531 root    5r  FIFO        0,9      0t0 2005473170 pipe
root@nb1963:~/dongzerun# lsof -p 4543
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF       NODE NAME
sleep.sh 4543 root    0r   CHR    1,3      0t0       1029 /dev/null
sleep.sh 4543 root    1w  FIFO    0,9      0t0 2005473170 pipe
sleep.sh 4543 root    2w  FIFO    0,9      0t0 2005473170 pipe
sleep.sh 4543 root   10r   REG    8,1       55    4993949 /root/dongzerun/sleep.sh
root@nb1963:~/dongzerun# lsof -p 4550
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF       NODE NAME
sleep   4550 root  mem    REG    8,1  1607664    9179617 /usr/lib/locale/locale-archive
sleep   4550 root    0r   CHR    1,3      0t0       1029 /dev/null
sleep   4550 root    1w  FIFO    0,9      0t0 2005473170 pipe
sleep   4550 root    2w  FIFO    0,9      0t0 2005473170 pipe
```

###原因总结
孙进程，启动后，默认会继承父进程打开的文件描述符，即 node 2005473170  的 pipe，那么当父进程被 kill -9 后会清理资源，关闭打开的文件，但是 close 只是引用计数减 1。实际上 孙进程 仍然打开着 pipe。回头看 agent 代码
```
	c.goroutine = append(c.goroutine, func() error {
		_, err := io.Copy(pw, c.Stdin)
		if skip := skipStdinCopyError; skip != nil && skip(err) {
			err = nil
		}
		if err1 := pw.Close(); err == nil {
			err = err1
		}
		return err
	})
```
那么当子进程执行结束后，go cmd  执行这个匿名函数的 io.Copy 来读取子进程输出数据，永远没有数据可读，也没有超时，阻塞在 copy 这里。

### 解决方案
原因找到了，解决方法也就有了。
1. 子进程启动孙进程时，增加 CloseOnEec 标记，但不现实，还要看孙进程的输出日志
2. io.Copy 改写，增加超时调用，理论上可行，但是要改源
3. 超时 kill 时，不单杀子进程，而是杀掉进程组

最终采用方案 3，简化代码如下，主要改动点有两处：
1. SysProcAttr 配置 Setpgid，让子进程与孙进程，拥有独立的进程组id，即子进程的 pid
2. syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL) 杀进程时指定进程组
```
func Run(instance string, env map[string]string) bool {
	var (
		cmd         *exec.Cmd
		proc        *Process
		sysProcAttr *syscall.SysProcAttr
	)

	t := time.Now()
	sysProcAttr = &syscall.SysProcAttr{
		Setpgid: true, // 使子进程拥有自己的 pgid，等同于子进程的 pid
		Credential: &syscall.Credential{
			Uid: uint32(uid),
			Gid: uint32(gid),
		},
	}

	// 超时控制
	ctx, cancel := context.WithTimeout(context.Background(), time.Duration(j.Timeout)*time.Second)
	defer cancel()

	if j.ShellMode {
		cmd = exec.Command("/bin/bash", "-c", j.Command)
	} else {
		cmd = exec.Command(j.cmd[0], j.cmd[1:]...)
	}

	cmd.SysProcAttr = sysProcAttr
	var b bytes.Buffer
	cmd.Stdout = &b
	cmd.Stderr = &b

	if err := cmd.Start(); err != nil {
		j.Fail(t, instance, fmt.Sprintf("%s\n%s", b.String(), err.Error()), env)
		return false
	}

	waitChan := make(chan struct{}, 1)
	defer close(waitChan)

	// 超时杀掉进程组 或正常退出
	go func() {
		select {
		case <-ctx.Done():
			log.Warnf("timeout kill job %s-%s %s ppid:%d", j.Group, j.ID, j.Name, cmd.Process.Pid)
			syscall.Kill(-cmd.Process.Pid, syscall.SIGKILL)
		case <-waitChan:
		}
	}()

	if err := cmd.Wait(); err != nil {
		j.Fail(t, instance, fmt.Sprintf("%s\n%s", b.String(), err.Error()), env)
		return false
	}
	return true
}
```




