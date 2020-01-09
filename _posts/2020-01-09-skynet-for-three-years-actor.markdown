---
layout: post
title:  "三年skynet有感——集群间通讯"
date:   2020-01-09 22:00:00 +0800
categories: 代码分析
---

集群间通讯

实际上，整个skynet的核心部分都是十分精简的，我们可以把它看成是一个消息驱动的Actor运行容器，由一个消息队列和若干个工作线程组成。而要实现skynet节点之间的沟通，则需要额外的名为harbor的service提供支持，这就像是每个Actor容器之间都是相互独立的运行个体，那要如何实现Actor容器的Actor消息发送呢？那就是借助一个专门负责Actor容器间通讯的Actor来处理转发的工作，这就是skynet的集群实现逻辑。

skynet的skynet_harbor.h/c，是分析skynet集群通讯的重要入口之一。上文提及到，skynet_send函数的执行过程，会先调用skynet_harbor_message_isremote函数，判断该handle id是为本机的service还是远程的service。判断的方法很简单，handle id是一个32位整数，该数字的高8位是集群通讯的harbor的id，其余的24位是本机的service的id。而这也暗示了，skynet的集群最大节点数不多于256个，单个进程的service数不多于16,777,216个，而skynet官方实现，规定集群最多允许255个skynet节点，因为harbor id为0，要留给识别本skynet的service。

从代码来看，skynet_harbor_message_isremote通过位运算，得出handle id是本机还是远程的结果，如果是远程的handle id，则调用skynet_context_send函数，而该函数的内部，还是把消息添加到消息队列处理了。而实际上，这是交给了skynet内置的harbor服务处理了。而skynet的harbor服务，也负责和建立TCP连接到集群其它的skynet节点。

```C
#define HANDLE_MASK 0xffffff
static struct skynet_context * REMOTE = 0;

void skynet_harbor_send(struct remote_message *rmsg, uint32_t source, int session) {
    int type = rmsg->sz >> MESSAGE_TYPE_SHIFT;
    rmsg->sz &= MESSAGE_TYPE_MASK;
    assert(type != PTYPE_SYSTEM && type != PTYPE_HARBOR && REMOTE);
    skynet_context_send(REMOTE, rmsg, sizeof(*rmsg) , source, type , session);
}

int skynet_harbor_message_isremote(uint32_t handle) {
    assert(HARBOR != ~0);
    int h = (handle & ~HANDLE_MASK);
    return h != HARBOR && h !=0;
}

void skynet_context_send(struct skynet_context * ctx, void * msg, size_t sz, uint32_t source, int type, int session) {
    ...
    smsg.sz = sz | (size_t)type << MESSAGE_TYPE_SHIFT;
    skynet_mq_push(ctx->queue, &smsg);
}
```

skynet实现的集群，其每个skynet的节点都有一个唯一的id，id的范围是从1~255的整形。而整个skynet的集群，有且仅有一个master服务，它有一个简单的内存数据库，记录了harbor id和对应的skynet节点的通讯地址。另外，该master服务也负责全局service的信息同步，例如需要启动一个集群全局的服务“x”，该服务“x”的启动和退出信息，会通过master服务同步到集群其它的skynet节点。

因为这些服务的逻辑大部分都是由lua代码编写的，因此我们从skynet的config开始看起，skynet的启动过程，会调用到skynet_start.c下的bootstrap函数，这个函数会解析config文件的bootstrap配置项，并创建skynet的上下文context对象，实际上这就是之前提到过的“符合skynet规范的C模块”。如以下配置，这将会启动snlua模块，并对snlua模块传bootstrap这个参数，这个参数的意思是加载skynet提供的bootstrap.lua文件，作为入口。

```lua
—- config file

include "config.path"
-- preload = "./examples/preload.lua"    -- run preload.lua before every lua service run
thread = 8
logger = nil
logpath = "."
harbor = 1
address = "127.0.0.1:2526"
master = "127.0.0.1:2013"
start = "main"    -- main script
bootstrap = "snlua bootstrap"    -- The service for bootstrap
standalone = "0.0.0.0:2013"
-- snax_interface_g = "snax_g"
cpath = root.."cservice/?.so"
-- daemon = "./skynet.pid"
```

在bootstrap.lua中我们可以看出来，skynet是通过配置文件的standalone项来设置该节点是否为master，以及如果是master的话，监听的端口是多少。如果不是master节点，则以slave的角色来启动service。而接下来的重点，则是cmaster.lua和cslave.lua两个内置service的实现了。

```lua
-- booststrap.lua

local skynet = require "skynet"
local harbor = require "skynet.harbor"
require "skynet.manager"    -- import skynet.launch, ...
local memory = require "memory"

skynet.start(function()
    local sharestring = tonumber(skynet.getenv "sharestring" or 4096)
    memory.ssexpand(sharestring)

    local standalone = skynet.getenv "standalone"

    local launcher = assert(skynet.launch("snlua","launcher"))
    skynet.name(".launcher", launcher)

    local harbor_id = tonumber(skynet.getenv "harbor" or 0)
    if harbor_id == 0 then
        assert(standalone ==  nil)
        standalone = true
        skynet.setenv("standalone", "true")

        local ok, slave = pcall(skynet.newservice, "cdummy")
        if not ok then
            skynet.abort()
        end
        skynet.name(".cslave", slave)

    else
        if standalone then
            if not pcall(skynet.newservice,"cmaster") then
                skynet.abort()
            end
        end


        local ok, slave = pcall(skynet.newservice, "cslave")
        if not ok then
            skynet.abort()
        end
        skynet.name(".cslave", slave)
    end


    if standalone then
        local datacenter = skynet.newservice "datacenterd"
        skynet.name("DATACENTER", datacenter)
    end
    skynet.newservice "service_mgr"
    pcall(skynet.newservice,skynet.getenv "start" or "main")
    skynet.exit()
end)
```

cmaster.lua和cslave.lua的实现，我们可以快速地看下master服务的启动代码，说白了就是监听并处理slave服务的请求。

```lua
-- cmaster.lua

skynet.start(function()
    local master_addr = skynet.getenv "standalone"
    skynet.error("master listen socket " .. tostring(master_addr))
    local fd = socket.listen(master_addr)
    socket.start(fd , function(id, addr)
        skynet.error("connect from " .. addr .. " " .. id)
        socket.start(id)
        local ok, slave, slave_addr = pcall(handshake, id)
        if ok then
            skynet.fork(monitor_slave, slave, slave_addr)
        else
            skynet.error(string.format("disconnect fd = %d, error = %s", id, slave))
            socket.close(id)
        end
    end)
end)
```


而slave服务，在启动的过程中，slave服务尝试连接集群的master服务，并获取其它slave服务的信息并连接，另外，开启一个新的协程来对本slave的请求进行监听，处理新的连接和连接断开的事件的处理。另外，slave也提供全局service命名注册等功能。

```lua
local function monitor_master(master_fd)
    while true do
        local ok, t, id_name, address = pcall(read_package,master_fd)
        if ok then
            if t == 'C' then
                if connect_queue then
                    connect_queue[id_name] = address
                else
                    connect_slave(id_name, address)
                end
            elseif t == 'N' then
                globalname[id_name] = address
                response_name(id_name)
                if connect_queue == nil then
                    skynet.redirect(harbor_service, address, "harbor", 0, "N " .. id_name)
                end
            elseif t == 'D' then
                local fd = slaves[id_name]
                slaves[id_name] = false
                if fd then
                    monitor_clear(id_name)
                    socket.close(fd)
                end
            end
        else
            skynet.error("Master disconnect")
            for _, v in ipairs(monitor_master_set) do
                v(true)
            end
            socket.close(master_fd)
            break
        end
    end
end
skynet.start(function()
    local master_addr = skynet.getenv "master"
    local harbor_id = tonumber(skynet.getenv "harbor")
    local slave_address = assert(skynet.getenv "address")
    local slave_fd = socket.listen(slave_address)
    skynet.error("slave connect to master " .. tostring(master_addr))
    local master_fd = assert(socket.open(master_addr), "Can't connect to master")

    skynet.dispatch("lua", function (_,_,command,...)
        local f = assert(harbor[command])
        f(master_fd, ...)
    end)
    skynet.dispatch("text", monitor_harbor(master_fd))

    harbor_service = assert(skynet.launch("harbor", harbor_id, skynet.self()))

    local hs_message = pack_package("H", harbor_id, slave_address)
    socket.write(master_fd, hs_message)
    local t, n = read_package(master_fd)
    assert(t == "W" and type(n) == "number", "slave shakehand failed")
    skynet.error(string.format("Waiting for %d harbors", n))
    skynet.fork(monitor_master, master_fd)
    if n > 0 then
        local co = coroutine.running()
        socket.start(slave_fd, function(fd, addr)
            skynet.error(string.format("New connection (fd = %d, %s)",fd, addr))
            socketdriver.nodelay(fd)
            if pcall(accept_slave,fd) then
                local s = 0
                for k,v in pairs(slaves) do
                    s = s + 1
                end
                if s >= n then
                    skynet.wakeup(co)
                end
            end
        end)
        skynet.wait()
        socket.close(slave_fd)
    else
        -- slave_fd does not start, so use close_fd.
        socket.close_fd(slave_fd)
    end
    skynet.error("Shakehand ready")
    skynet.fork(ready)
end)
```

有了cmaster和cslave这两个集群的基础支持服务，接下来就是clusterd服务的事情了，这个服务提供了skynet集群的跨节点通讯功能。暂且不一一分析论述了。

三年在这个公司，实际上我们公司的skynet的使用习惯，并没有用到太多这些skynet内部提供的master-slave功能，也没有用到skynet内部提供的集群命名服务机制。服务器的主备和service的命名服务是肯定要做的，但我们的做法是借助redis数据库，通过在redis模拟争夺锁的方式，来定夺唯一的master节点是哪个。另外，service的命名也是注册在redis上。其实我不是很认可这种做法，虽然直到目前看起来也没有什么问题，但我总是觉得，不应该把锁的争夺放在redis上去做，因为redis模拟锁机制这种做法，不是十分可靠。
