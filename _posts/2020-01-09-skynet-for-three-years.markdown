三年skynet有感——基于Actor模型理念的事件驱动的并行并发框架

符合Actor模型，是skynet很重要的特征。在Actor模型中，每个Actor都是一个最基本的计算单元，其内部对外是一个黑箱，Actor不接受外部的“调用”，但Actor可以接收外部的Message，和向其它Actor发送Message。虽然作者不怎么说skynet借鉴了Actor模型的思想，但事实上，每个skynet的service，它的功能都相当于一个独立的Actor，“service可以向skynet框架注册一个callback函数，用来接收发给它的消息”，但service和service之间，并没有办法可以直接调用，因为每个service都是一个独立的。

虽然从skynet的官方实现和官方例子，每个运行的service都是一个独立的lua state，能实现类似沙箱的效果，但实际上，skynet运行的service都是符合skynet规范的C模块，如官方例子中启动的service，它们都是从动态库中启动起来的，也就是说，我们可以用C/C++语言编写符合skynet规范的service模块，而不一定只能加载官方的lua state模块。而当service启动后，其会绑定一个即使service退出了也不会重复的数字id的类似句柄的东西，skynet称其为handle。其实我认为，这个只有uint32的handle id，在service快速启动和退出的场景下，其实还是会重复啦，所以其关键在于新创建的handle id不要和正在运行的service的handle id有冲突即可。另外skynet还提供了名字服务，即给service起一个特定的名字。而这些功能的具体实现，都在skynet.handle.h/c文件下。

Actor模型有两个重要的概念，每个Actor本身，和Actor间消息的传递。以上已经提到，每个Actor都有一个唯一的句柄，以及Actor像个黑箱，其内部实现外界是不可知的。而Actor的行为只有两种——接收消息和发送消息，而Actor的外部，则是Actor间消息传递的介质。这种介质，是Actor运行容器提供的高效的消息传递环境。

实际上，Actor模型的消息传递的理念，是其作为并行计算模型的关键，也是Actor运行容器框架充分利用机器多核性能的关键。曾经，锁作为一种重要的多线程解决方案，其也因容易产生死锁的问题而被唾弃，直至“无锁”概念的引入，而所谓的“无锁”，并不是真的去掉了锁，而是转移了锁，把锁从本来的“业务锁”变成了“无锁队列”的“锁”。而在Actor模型思想指导的并行运算框架，Actor间的消息传递正好需要借力这种“无锁队列”，我们不需要担心多线程运行时，线程间争夺业务资源而出现问题，因为线程间不会争夺同一个资源，而是资源通过队列分发的方式，分发给不同的线程执行了。

以下结合skynet的实际来看：skynet的service之间发送消息，是通过skynet_send函数来实现的。从代码的实现来看，skynet_send函数可能会同时被多个工作线程调用，但因为skynet_mq_push函数中有一个队列的旋转锁，所以发送消息这一个操作，是线程安全的。这也是消息队列生产者的模式吧。

```C
int skynet_send(struct skynet_context * context, uint32_t source, uint32_t destination , int type, int session, void * data, size_t sz) {
    // 无关代码恕删
    ...
    ...
    if (skynet_harbor_message_isremote(destination)) {
        ...
        ...
        skynet_harbor_send(rmsg, source, session);
    } else {
        ...
        ...
        if (skynet_context_push(destination, &smsg)) {
            skynet_free(data);
            return -1;
        }
    }
    return session;
}

int skynet_context_push(uint32_t handle, struct skynet_message *message) {
    ...
    skynet_mq_push(ctx->queue, message);
    ...
}

void skynet_mq_push(struct message_queue *q, struct skynet_message *message) {
    ...
    SPIN_LOCK(q)
    q->queue[q->tail] = *message;
    if (++ q->tail >= q->cap) {
        q->tail = 0;
    }
    ...
    SPIN_UNLOCK(q)
}
```

而在消息队列的另一头，则有一群工作线程，它们运行着thread_worker函数，空闲的工作线程进入pthread_cond_wait状态，等待着消费新的消息。当有新消息时，则调用skynet_context_message_dispatch函数进行新消息的处理。

```C
static void * thread_worker(void *p) {
    // 无关代码恕删
    ...
    ...
    while (!m->quit) {
        q = skynet_context_message_dispatch(sm, q, weight);
        if (q == NULL) {
            if (pthread_mutex_lock(&m->mutex) == 0) {
                ++ m->sleep;
                // "spurious wakeup" is harmless,
                // because skynet_context_message_dispatch() can be call at any time.
                if (!m->quit)
                    pthread_cond_wait(&m->cond, &m->mutex);
                -- m->sleep;
                if (pthread_mutex_unlock(&m->mutex)) {
                    fprintf(stderr, "unlock mutex error");
                    exit(1);
                }
            }
        }
    }
    return NULL;
}

...
    for (i=0;i<thread;i++) {
        ...
        create_thread(&pid[i+3], thread_worker, &wp[i]);
    }
...
```

Actor模型为我们隐藏了一个事实，整个程序是通过消息队列驱动起来的，即消息驱动。在skynet这个例子中，消息队列作为整个skynet本地应用的核心，如果没有新的消息被消费，那么工作线程则等待新的消息；当有新的消息传进来，会有一个空闲的工作线程争夺到这个消息，并投递给Actor即skynet的service作处理。因此，我们也要确保当service处理完之后，不要再占用工作线程，应该告诉skynet已经处理完毕，而实际上，lua state就结合lua的coroutine，做了这样的处理。

哦对了，这么看来，skynet是事件驱动框架。

值得注意的是，这里会产生一个引申的问题，就是发送数据和接收数据的一致性，因为发送消息的service和接收消息的service可能是并行的，而skynet中实际传递的是数据的指针，因此一方对消息的数据进行改动，都可能会对另一方造成影响。因此，需要发送的消息数据，应该要做malloc和copy的工作，并在对方service使用完消息之后执行free；或者想办法确保发送的数据和接收的数据不变，如erlang的做法便是如此。而skynet中，是使用一个名为PTYPE_TAG_DONTCOPY的type，来告诉skynet到底该消息是否应该被copy。

接下来看skynet_send函数的具体参数：

```C
int skynet_send(
  struct skynet_context * context,
  uint32_t source,
  uint32_t destination,
  int type,
  int session,
  void * msg,
  size_t sz
);

typedef int (*skynet_cb)(
  struct skynet_context * context,
  void *ud,
  int type,
  int session,
  uint32_t source ,
  const void * msg,
  size_t sz
);
```

source和destination，就是service创建时分配到的句柄handle id，这是service的重要标记。

session，是用于跨进程发送消息的情况，当对方进程返回同样session消息，则意味着这个消息是发送消息的callback。

type，是用于指定消息类型的，以下是type的定义。

```C
#define PTYPE_TEXT 0
#define PTYPE_RESPONSE 1
#define PTYPE_MULTICAST 2
#define PTYPE_CLIENT 3
#define PTYPE_SYSTEM 4
#define PTYPE_HARBOR 5
#define PTYPE_SOCKET 6
#define PTYPE_ERROR 7
#define PTYPE_RESERVED_QUEUE 8
#define PTYPE_RESERVED_DEBUG 9
#define PTYPE_RESERVED_LUA 10
#define PTYPE_RESERVED_SNAX 11
#define PTYPE_TAG_DONTCOPY 0x10000
#define PTYPE_TAG_ALLOCSESSION 0x20000
```
