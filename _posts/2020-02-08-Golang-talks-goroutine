---
layout: post
title:  "Golang随谈——Goroutine相关概念"
date:   2020-02-08 03:00:00 +0800
categories: Golang
---

通常我们认为，并发编程是Go语言的重大特色，其并发的理念以CSP为基础，实现方式是goroutine并行方法和channel数据类型。

## Goroutine

Go的Goroutine是类似于线程的概念，但它既不是线程（Thread），也不是协程（Coroutine），而是轻量级线程（light-weight process, LWP）。线程属于系统层面，通常来说建立一个新的线程会消耗较多资源且不易管理；而协程的主要作用是提供一个线程内的并发性，却不能利用多个处理器线程；而Goroutine这种轻量级线程，一个Go程序可以执行过万个Goroutine，这是线程所不能的，而且Goroutine的性能都是原生级的，随时能够关闭和结束，并且可以运行在多个处理器线程之上，从而实现真正的并发并行，这是协程所不能的。

顺带一提，典型线程是抢占式多任务，操作系统完全决定线程调度方案，操作系统可以剥夺耗时长的线程的时间片，把它提供给其它线程。而协程是协作式多任务，协作式多任务要求每一个运行中的程序，定时放弃自己的执行权利，让给下个任务，也就是说，下一个协程被调度的前提是当前协程主动放弃时间片，这意味着协程提供并发性而非并行性。

## Goroutine和Channel

Goroutine和Channel是Go实现CSP的方式，CSP是Communicating Sequential Process模型的意思，该模型提倡通过通信共享内存，而不是通过共享内存来实现通信。Go的channel是类似于管道的概念，但它不是真的管道，而是一种语言特征。Go语言并不排斥开发者使用锁，其标准库中也提供了锁的支持，但channel是一种方便快捷的替代方案，开发者可以通过channel在goroutine之间发送消息，而且消息的收发顺序是FIFO的（在没有阻塞的情况下FIFO，因为channel底层也是通过锁实现，当阻塞的时候，并不能保证下一个是哪个goroutine能抢夺到写入的锁）。通过发送消息的方法来进行通讯有个好处，在合理的使用下，这能够避免共享可变状态的问题。

首先要理解什么是共享可变状态，这是《七天七并发模型》一书中提到的概念，是指线程之间共享的可变的数据。而对于不变的数据，多线程不使用锁就可以安全地进行访问。书中同时提到的两种可变状态，一种是隐藏的可变状态，即我们以为该状态是不可变的，但在实际的实现中却是可变的；另一种是逃逸的可变状态，即在不知情的情况下对外共享了变量的状态。我们要避免共享可变状态，也可以看作是我们要实现不同状态之间的隔离。

我认为在Go语言中，我们要把每个运行的Goroutine视为一个需要被隔离的独立的状态，因为共享可变状态导致的问题，症结在于不同的线程同时运行的时候，其中某个线程篡改了不属于它的其它线程正在使用的数据。那么让我们把一个Goroutine和其独自使用的可变状态的数据（变量）视为一个对象，这个对象是一个需要与外界隔离的独立的，这些对象之间不能相互影响，那就避免了上面说的一个线程被另一个线程篡改了数据的问题。但这些变量间终归是要通讯的，那我们就借助Go的channel的威力，让对象之间通过channel通讯，对象接收到channel的消息后自行处理，那就避免了对象之间直接互相操作的问题。

但在Go语言中有个陷阱，以上的方案看起来很美好，但是channel之间传递的信息，这个信息可能会隐含发送方对象的状态，因为Go语言并不像erlang语言规定了一次性赋值（single-assignment variable），而且很多变量也是通过复制引用的方式来传递，而不会对本来的对象进行deep copy。在以上的情况下，共享可变状态的问题又再次出现了，接收方可能会因为修改了信息中的值，导致影响了发送方的状态。要避免这种问题，有三种方法：

1. 发送方发送的消息不含有自身的可变状态，让发送的消息和发送方持有的可变状态无关。可以通过深层复制要发送的消息，来隔离消息的状态和发送方的状态。
2. 接收方接收消息之后，对消息不作任何的修改。以上两点是空间上的思考。
3. 信息的发送方在发出信息之后不再修改发送的消息，并立即终止了自己的运行。这是时序上的思考。实际上，这种时序性的程序，有点像生产车间的流水线，被处理的消息像流水线上的产品，不同的工序（不同的goroutine）之间并不会同时修改一个产品（消息）。

以上说的这些情况，都是开发者自己需要去避免的。

顺带一提，以上把Goroutine及其使用的可变状态视为一个对象的做法，类似于Actor模型。Actor模型的程序是由独立的、并发执行的实体组成，这些实体之间通过发送消息进行通信，每个Actor都有一个信箱，用于保存已经收到但尚未处理的消息。而Go的channel，关注的不是像Actor模型中发送消息的实体，而是关注发送发送消息时所使用的通道，channel是第一类对象，它不像进程那样与信箱是紧耦合的，而是可以单独创建和读写，并在进程之间传递。因此Go的channel的用法有很多，以上只是提供一种参考。但channel作为Go语言并发的一大特征，Goroutine和Channel本来就是天生一对，密不可分。

## Goroutine和WaitGroup

我们可以使用sync.WaitGroup来实现并发控制，我们很多时候想要等待所有goroutine执行完毕再执行其它的任务，但我们不一定需要goroutine返回的细节，所以channel在此时显得有点多余，如果我们能有一个类似引用计数的计数器，在goroutine开始执行的时候对计数器+1，在goroutine结束执行的时候对计数器-1，那么当等到计数器减至0的时候，我们就知道所有的goroutine都执行完毕了。

sync包的WaitGroup就是这么一种工具，它等待一篮子的goroutine完成，而要注意的是，当我们创建了一个WaitGroup之后，就不要去复制它了，这会导致错误。而WaitGroup提供三种重要的方法：

1. Add(delta int)，向WaitGroup添加一个计数。正常来说，我们都会添加一个正整数，但也可以是负数，虽然这种做法很怪异。最好的做法当然是Add(1)了，因为Done()对应的是-1，它们通常成对出现，在进入goroutine之前Add(1)，在goroutine运行的函数defer的时候Done()。另外，如果Add一个负整数导致WaitGroup计数器是负数，这会导致Add的调用panic。

2. Done()，向WaitGroup减少一个计数。

3. Wait()，阻塞当前的goroutine，直到WaitGroup的计数变为0，则恢复下文的运行。

要注意的是，要在进入子任务的goroutine之前，调用Add方法，因为我们并不能确保子任务的goroutine在和调用Wait方法的goroutine哪个先执行，因为它们有可能是并行的。

以下是官方文档的例子：

```Go
package main

import (
    "sync"
)

type httpPkg struct{}

func (httpPkg) Get(url string) {}

var http httpPkg

func main() {
    var wg sync.WaitGroup
    var urls = []string{
        "http://www.golang.org/",
        "http://www.google.com/",
        "http://www.somestupidname.com/",
    }
    for _, url := range urls {
        // Increment the WaitGroup counter.
        wg.Add(1)
        // Launch a goroutine to fetch the URL.
        go func(url string) {
            // Decrement the counter when the goroutine completes.
            defer wg.Done()
            // Fetch the URL.
            http.Get(url)
        }(url)
    }
    // Wait for all HTTP fetches to complete.
    wg.Wait()
}
```
## Goroutine和Context

Context是更加高级的并发上下文处理工具，这个工具提供了单个任务产生了多个goroutine的管理能力。因为为了完成一个任务，任务goroutine可能会创建其它的goroutine来处理，我们可以通过Context这个上下文管理的工具，来控制这些额外产生的goroutine。Context为我们提供了截止期限、取消信号、和跨API边界以及进程之间的，访问其它请求域数值的能力。

我们先结合一个例子来理解：

```Go
func Stream(ctx context.Context, out chan<- Value) error {
    for {
        v, err := DoSomething(ctx)
        if err != nil {
            return err
            }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case out <- v:
        }
    }
}
```

Stream这个函数会不断地调用DoSomething函数来生成结果，并发送到out这个channel中。这个循环退出有两个任意条件，一是DoSomething返回错误，二是ctx.Done()返回的管道被关闭。如果是第二种情况，则会返回ctx中的error信息。

要使用Context，则要实现Context interface：

1. Deadline() (deadline time.Time, ok bool)：这个方法返回这个Context应该被取消的具体时间，如果没有取消时间，则让ok变量返回false。

2. Done() <-chan struct{}：这个方法返回一个channel，这个channel会在这个context完成并需要取消其它子任务时被关闭。如果这个context永远不能被取消，那么这个channel会一直阻塞。如果Done被调用的话，则阻塞的Done会返回。

3. Err() error：如果Done()未被关闭，则返回nil；如果Done()被关闭，则会返回一个非空的error实例来解释为何被取消，或者告诉用户context的期限已经到了。

4. Value(key interface{}) interface{}：返回一个和key相关的value。这个值仅应该用在请求域数据，而且该数据必须是线程安全的。

实际上我们不一定要亲自实现Context，因为context库已经内置了Background和TODO两个Context。其中Background永远不会被取消，也没有values和时间限制，它通常用在main函数、初始化和测试中，是处理请求的最高等级context。

再看一个例子：

```Go
package main

import (
    "context"
    "fmt"
)

func main() {
    gen := func(ctx context.Context) <-chan int {
        dst := make(chan int)
        n := 1
        go func() {
            for {
                select {
                case <-ctx.Done():
                    return // returning not to leak the goroutine
                case dst <- n:
                    n++
                }
            }
        }()
        return dst
    }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    for n := range gen(ctx) {
        fmt.Println(n)
        if n == 5 {
            break
        }
    }
}
```

gen闭包会在另一个独立的goroutine中生成整数并发送到返回的channel。而gen闭包函数需要在使用完毕之后被退出，以避免闭包内的goroutine泄漏。这里的做法是当示例最后的消费者for循环使用完gen闭包函数后，main函数会被返回并执行defer，而defer会调用context.WithCancel的cancel函数，而当这个cancel函数被调用，则会触发gen闭包函数中的goroutine的select的ctx.Done()返回的管道不再阻塞，从而执行return，退出该goroutine，避免了该goroutine的泄漏。

我们可以简单地理解为，context是一个退出goroutine的辅助工具，帮我们中断一系列的相关的goroutine。

## 结束一个Goroutine

Goroutine，可能是一个Go程序运行的最基本的要素了，通过go这个syntax，我们可以轻松地创建一个Goroutine，驱动一个函数。尽管我们可以很轻易地创建大量的Goroutine，但却不一定懂得如何结束Goroutine，不同于线程等其它并发工具，Goroutine在创建成功之后，并不会返回其句柄或引用，因此我们没有办法直接关闭一个Goroutine，要结束一个Goroutine，只能够想办法让Goroutine执行的函数返回，如果一个函数是一个死循环，那么这个Goroutine就像是一个悬空的对象，只有通过结束整个程序的方式来销毁这些悬空的Goroutine了，这种bug也会导致程序的内存泄漏。

要避免“悬空”的Goroutine，我们要首先分析会让一个函数不返回的情况：例如，在for循环中没有做好结束循环的处理，或者中断处理无法被触发，导致了无限循环；一直等待一个永远不返回的channel；等待一个IO的资源，但该IO资源一直阻塞，也没有设置超时时间；函数中使用了锁，并出现了死锁的问题。尝试执行一个运算量惊人，以该电脑性能不可能算得完的函数，而且函数中没有中断的判断，这也会导致函数一直不返回，这种不算“悬空”，但也是要考虑的问题。

顺带一提，在Go test中，如果一个函数的执行阻塞超过十分钟，那么test函数就会抛出异常，中断测试，并把这次的测试视为失败的。这可能是仅针对channel等情况阻塞超时的处理，如果是正在进行运算的Go test，可能不在视为失败的范畴。这个以后有机会再验证。

2020.02

tou.hwang



---

参考资料：
1. 《七天七并发模型》· Paul Butcher
2. [Go](https://en.wikipedia.org/wiki/Go_(programming_language)#Concurrency:_goroutines_and_channels)
2. [Go](https://zh.wikipedia.org/wiki/Go)
3. [轻量级进程](https://zh.wikipedia.org/wiki/轻量级进程)
4. [协程](https://zh.wikipedia.org/wiki/协程)
5. [协作式多任务](https://zh.wikipedia.org/wiki/协作式多任务)
6. [抢占式多任务处理](https://zh.wikipedia.org/wiki/抢占式多任务处理)
7. [共享可变状态中出现的问题以及如何避免](https://segmentfault.com/a/1190000020844071)
8. [共享的可变状态值与并发](http://www.liying-cn.net/kotlin/docs/reference/coroutines/shared-mutable-state-and-concurrency.html)
9. [Communicating sequential processes](https://en.wikipedia.org/wiki/Communicating_sequential_processes)
10. [了不起的 Erlang](https://zhuanlan.zhihu.com/p/27341488)
11. [WaitGroup](https://golang.org/pkg/sync/#WaitGroup)
12. [Package context](https://golang.org/pkg/context/)

