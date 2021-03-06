### 信号
Go中的Signal发送和处理
有时候我们想在Go程序中处理Signal信号，比如收到 SIGTERM 信号后优雅的关闭程序(参看下一节的应用)。
Go信号通知机制可以通过往一个channel中发送 os.Signal 实现。
首先我们创建一个os.Signal channel，然后使用 signal.Notify 注册要接收的信号。

golang中对信号的处理主要使用os/signal包中的两个方法：
notify方法用来监听收到的信号
stop方法用来取消监听

#### 监听全部信号：
```go
// 监听全部信号
func main()  {
    //合建chan
    c := make(chan os.Signal)
    //监听所有信号
    signal.Notify(c)
    //阻塞直到有信号传入
    fmt.Println("启动")
    s := <-c
    fmt.Println("退出信号", s)
}
```
#### 监听指定的信号
```go
// 监听指定信号
func main()  {
    //合建chan
    c := make(chan os.Signal)
    //监听指定信号 ctrl+c kill
    signal.Notify(c, os.Interrupt, os.Kill, syscall.SIGUSR1, syscall.SIGUSR2)
    //阻塞直到有信号传入
    fmt.Println("启动")
    //阻塞直至有信号传入
    s := <-c
    fmt.Println("退出信号", s)
}
```

```go
func main() {
 
// Go signal notification works by sending `os.Signal`
// values on a channel. We'll create a channel to
// receive these notifications (we'll also make one to
// notify us when the program can exit).
 sigs := make(chanos.Signal,1)
 done := make(chanbool,1)
 
// `signal.Notify` registers the given channel to
// receive notifications of the specified signals.
 signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
 
// This goroutine executes a blocking receive for
// signals. When it gets one it'll print it out
// and then notify the program that it can finish.
go func() {
 sig := <-sigs
 fmt.Println()
 fmt.Println(sig)
 done <- true
 }()
 
// The program will wait here until it gets the
// expected signal (as indicated by the goroutine
// above sending a value on `done`) and then exit.
 fmt.Println("awaiting signal")
 <-done
 fmt.Println("exiting")
}
```

kill pid的作用是向进程号为pid的进程发送SIGTERM（这是kill默认发送的信号），该信号是一个结束进程的信号且可以被应用程序捕获。若应用程序没有捕获并响应该信号的逻辑代码，则该信号的默认动作是kill掉进程。这是终止指定进程的推荐做法。

kill -9 pid则是向进程号为pid的进程发送SIGKILL（该信号的编号为9），从本文上面的说明可知，SIGKILL既不能被应用程序捕获，也不能被阻塞或忽略，其动作是立即结束指定进程。通俗地说，应用程序根本无法“感知”SIGKILL信号，它在完全无准备的情况下，就被收到SIGKILL信号的操作系统给干掉了，显然，在这种“暴力”情况下，应用程序完全没有释放当前占用资源的机会。事实上，SIGKILL信号是直接发给init进程的，它收到该信号后，负责终止pid指定的进程。在某些情况下（如进程已经hang死，无法响应正常信号），就可以使用kill -9来结束进程。

若通过kill结束的进程是一个创建过子进程的父进程，则其子进程就会成为孤儿进程（Orphan Process），这种情况下，子进程的退出状态就不能再被应用进程捕获（因为作为父进程的应用程序已经不存在了），不过应该不会对整个linux系统产生什么不利影响。

#### 优雅退出
Linux Server端的应用程序经常会长时间运行，在运行过程中，可能申请了很多系统资源，也可能保存了很多状态，在这些场景下，我们希望进程在退出前，可以释放资源或将当前状态dump到磁盘上或打印一些重要的日志，也就是希望进程优雅退出（exit gracefully）。

从上面的介绍不难看出，优雅退出可以通过捕获SIGTERM来实现。具体来讲，通常只需要两步动作：
1）注册SIGTERM信号的处理函数并在处理函数中做一些进程退出的准备。信号处理函数的注册可以通过signal()或sigaction()来实现，其中，推荐使用后者来实现信号响应函数的设置。信号处理函数的逻辑越简单越好，通常的做法是在该函数中设置一个bool型的flag变量以表明进程收到了SIGTERM信号，准备退出。
2）在主进程的main()中，通过类似于while(!bQuit)的逻辑来检测那个flag变量，一旦bQuit在signal handler function中被置为true，则主进程退出while()循环，接下来就是一些释放资源或dump进程当前状态或记录日志的动作，完成这些后，主进程退出。

```go
func main() {
	//创建监听退出chan
	c := make(chan os.Signal)
	//监听指定信号 ctrl+c kill
	signal.Notify(c, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT, syscall.SIGUSR1, syscall.SIGUSR2)
	go func() {
		for s := range c {
			switch s {
			case syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT:
				fmt.Println("退出", s)
				ExitFunc()
			case syscall.SIGUSR1:
				fmt.Println("usr1", s)
			case syscall.SIGUSR2:
				fmt.Println("usr2", s)
			default:
				fmt.Println("other", s)
			}
		}
	}()
	fmt.Println("进程启动...")
	sum := 0
	for {
		sum++
		fmt.Println("sum:", sum)
		time.Sleep(time.Second)
	}
}
```

信号	值	动作	说明
SIGHUP	1	Term	终端控制进程结束(终端连接断开)
SIGINT	2	Term	用户发送INTR字符(Ctrl+C)触发
SIGQUIT	3	Core	用户发送QUIT字符(Ctrl+/)触发
SIGILL	4	Core	非法指令(程序错误、试图执行数据段、栈溢出等)
SIGABRT	6	Core	调用abort函数触发
SIGFPE	8	Core	算术运行错误(浮点运算错误、除数为零等)
SIGKILL	9	Term	无条件结束程序(不能被捕获、阻塞或忽略)
SIGSEGV	11	Core	无效内存引用(试图访问不属于自己的内存空间、对只读内存空间进行写操作)
SIGPIPE	13	Term	消息管道损坏(FIFO/Socket通信时，管道未打开而进行写操作)
SIGALRM	14	Term	时钟定时信号
SIGTERM	15	Term	结束程序(可以被捕获、阻塞或忽略)
SIGUSR1	30,10,16	Term	用户保留
SIGUSR2	31,12,17	Term	用户保留
SIGCHLD	20,17,18	Ign	子进程结束(由父进程接收)
SIGCONT	19,18,25	Cont	继续执行已经停止的进程(不能被阻塞)
SIGSTOP	17,19,23	Stop	停止进程(不能被捕获、阻塞或忽略)
SIGTSTP	18,20,24	Stop	停止进程(可以被捕获、阻塞或忽略)
SIGTTIN	21,21,26	Stop	后台程序从终端中读取数据时触发
SIGTTOU	22,22,27	Stop	后台程序向终端中写数据时触发
