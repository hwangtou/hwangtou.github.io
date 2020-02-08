---
layout: post
title:  "Golang随谈——浅瞰底层：Go的并发调度模型"
date:   2020-02-08 03:00:00 +0800
categories: Golang
---

## 调度器分析

国内喜欢把Go的并发模型称为G-M-P模型，但在网上一查，貌似国外并没有这样的定义，他们喜欢直接称其为Go Scheduler——Go的调度器。不管如何，G-M-P都是Go调度器中的重要概念，它们都定义在sys/runtime/runtime2.go文件中，让我们看看它们都代表什么吧：

1. G for Goroutine，定义于struct g，其存放着Goroutine的状态信息，如保存着Goroutine的执行堆栈信息、Goroutine的等待信息和变量的GC信息等信息。我们每用关键字go创建一个Goroutine，其在go程序的底层都创建了一个对应的G对象。
2. M for Machine，定义于struct m，对应着操作系统的工作线程，其和物理处理器线程对应，它负责任务的调度，是实际驱动G运行的实体，但M不负责G的状态的管理，需要切换执行的G时，M会把G的堆栈状态写回到G。实际上，在Go1.1之前，Go只有G-M模型，此前还没有P这个概念，直到Dmitry Vyukov在2012年发表了《Scalable Go Scheduler Design Doc》文章，改文章指出当时G-M模型的问题——用全局唯一的锁和中心化的状态来保护Goroutine相关的操作，Goroutine之间的切换可能会过载，每个M之间的内存缓存问题，以及抢占式线程的阻塞和非阻塞增加了很多开销。因此提出引入Processors的概念到runtime中。
3. P for Processor，定义于struct p，实现了逻辑上的处理器，它的责任是负责提供相关的上下文环境，负责内存缓存的管理，负责Goroutine任务队列等。P是G和M的中间层，M会和P先绑定，然后M会不断地从P的任务队列中取出G并恢复执行（取出操作无锁，因为没有资源竞争的问题），当P的任务队列都处理完，P再从全局队列中返回一个G来执行（取出操作有锁，因为这可能会和其它的P竞争），当全局队列也没有G时，则从其它的P窃取G来执行。当再也没有G可以被执行时，M和P会被解绑，进入休眠状态。P的个数默认为物理线程数。

Go的调度器才是概念的重点，而G-M-P则是Go调度器组成的重要部分。P和M通常是对应的，简单来说，P管理着一组G，并负责把G挂载到M上运行。当一个M长时间在运行同一个M时，runtime会创建一个新的M，阻塞的G所在的P会把其余的G挂载到新的M上，当这个阻塞的G阻塞完成或者结束时，该旧M会被回收。

关于G的运行，因为M在运行G的过程中，会遇到需要上下文切换的情况——当一个被运行的G要被切换时，需要对G的执行现场进行保护，以便下次被调度执行时进行现场恢复，Go调度器的做法是，把M的堆栈和M所需的寄存器（SP、PC等）保存到G中，就可以实现现场保护了。当调度器再次运行该G的时候，M通过访问G中保存的寄存器进行现场恢复，即可从上次中断的位置继续执行。

Go这种调度器使用了m:n调度的技术，即复用或调度m个goroutine到n个OS线程。其中m的调度由Go程序的运行时负责，n的调度由OS负责。这让m的调度可以在用户态下完成，不会造成内核态和用户态见的频繁切换。同时，内存的分配和释放，文件的IO等，Go也通过内存池和netpoll等技术，尽量减少内核态的调用。

## G的状态

Goroutine在生命周期的不断的阶段，会有不同的G状态。而通过分析G的状态，有助于我们了解Goroutine的调度。在runtime2.go文件中定义了，G有以下几种状态——idle, runnable, running, syscall, waiting, dead, copystack六种非GC状态，以及scan, scanrunnable, scan running, scansyscall, scanwaiting六种对应的GC状态，而moribund_unused和enqueue_unused两种状态已经被废弃了：

1. _Gidle for idle，意思是这个goroutine刚被创建出来，还未被进行初始化。因为它的值为0，所以刚被创建出来的g对象都是_Gidle。但在runtime库仅有的两处调用中，创建出来的g都马上被赋值为_Gdead，这是为了g在添加到被GC观察之前，用于躲避trackbacks和stack scan，因为这个g对象在必要的处理前，还不是一个真正的goroutine。
2. _Grunnable for runnable，意思是这个goroutine已经在运行队列，在这种情况下，goroutine还未执行用户代码，M的执行栈还不是goroutine自己的。
3. _Grunning for running，意思是goroutine可能正在执行用户代码，M的执行栈已经由该goroutine所拥有，此时对象g不在运行队列中。这个状态值要待分配给M和P之后，交由M和P来设定。
4. _Gsyscall for system scall，意思是这个goroutine正在执行系统调用，而不是正在执行用户代码。和_Grunning一样，goroutine拥有了执行栈，也不在运行队列中。这个状态值只能由分配给的M来设定。
5. _Gwaiting for waiting，意思是goroutine在运行时被阻塞，它既不执行用户代码，也不在运行队列。它被记录在其它的地方，例如管道等待队列——channel wait queue，因此当需要该goroutine的时候，该goroutine可以马上就绪，这也是goroutine和channel的底层实现方式。这个时候，执行栈不被该g对象所拥有，除非一个管道正在做读或者写执行栈里面数据的操作。除了以上这类型的情况，在一个goroutine进入_Gwaiting之后尝试获取其执行栈，都是不安全的。
6. _Gdead for dead，意思是这个goroutine在当前不被使用，这种情况可能是goroutine刚被创建出来，或者已经执行完毕退出并被放到释放列表中。当一个G执行完毕并正在退出时，和G被添加到释放列表时，G和G的执行栈都是M所拥有的。
7. _Gcopystack for copy stack，意思是这个goroutine的执行栈已经被移动，这个goroutine即不执行用户代码，也不在运行队列。这种状态是_Grunning的时候，出现了执行栈空间不足或者过大，需要扩容或者GC的情况下发生，是进行执行栈扩容或者收缩时的中间状态。
8. _Gscan系列，用于标记正在被GC扫描的状态，这些状态是由_Gscan=0x1000再加上_GRunnable, _Grunning, _Gsyscall和_Gwaiting的枚举值所产生的，这么做的好处是直接通过简单的运算即可知道被Scan之前的状态。当被标记为这系列的状态时，这些goroutine都不会执行用户代码，并且它们的执行栈都是被做该GC的goroutine所拥有。不过_Gscanrunning状态有点特别，这个标记是为了阻止正在运行的goroutine切换成其它状态，并告诉这个G自己扫描自己的堆栈。正是这种巧妙的方式，使得Go语言的GC十分高效。

从以上列举的状态可以分析出，无论是处理waiting的业务，还是处理GC，goroutine是高效的。但当要调用system call的时候则不然，低效的系统调用业务代码，会影响Go应用的运行性能，幸好Go语言中已经封装了很多能代替用户低效的系统调用的工具，例如网络调用看似是系统调用，但Go实际上已经在底层封装了netpoll，我们应该尽量使用这些库来避免系统调用。不合理的设计导致频繁copy stack和会导致频繁GC的设计等设计，这些都是我们需要注意的。

2020.02

tou.hwang



---
延伸阅读：
1. [The Go scheduler](http://morsmachine.dk/go-scheduler)
2. [Scheduling In Go : Part I - OS Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
3. [Scheduling In Go : Part II - Go Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)
4. [Scheduling In Go : Part III - Concurrency](https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html)

参考资料：

1. [Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#heading=h.mmq8lm48qfcw)
2. Go 1.13.1 源代码

虽然有参考以下文章，但我觉得里面的概念和说法未必都是对的，所以我对其内容我作重新思考和甄别，再作参考。

1. [Go 调度模型](https://wudaijun.com/2018/01/go-scheduler/)
2. [Go 调度模型](https://www.liwenzhou.com/posts/Go/14_concurrence/)
3. [深入Golang调度器之GMP模型](https://www.cnblogs.com/sunsky303/p/9705727.html)
4. [Go语言——goroutine并发模型](https://www.jianshu.com/p/f9024e250ac6)

