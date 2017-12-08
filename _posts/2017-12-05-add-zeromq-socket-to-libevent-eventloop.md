---
layout: post
title:  "ZeroMQ socket与libevent的融合使用"
date:   2017-12-05 
author: 王可
categories: network libevent zeromq
permalink: /add-zeromq-socket-to-libevent-eventloop
---



有时候我们既需要`ZeroMQ`带来的特性，也需要`libevent`的`Reactor框架`，这时就需要将两者结合使用。特别是，我们要能将`ZeroMQ`创建的zmq socket加入到`libevent`的event loop中，其后端的I/O multiplex机制根据操作系统的不同而不同，如select/poll/kqueue/epoll等，但它们都是将OS socket对应的file discriptor加入到这个event loop中。

而ZeroMQ的socket从API使用者的角度看来，是一个opaque object；我们可以根据`zmq_getsockopt`的`ZMQ_FD`选项来获取其对应的OS socket fd。其文档说明如下：

> The ZMQ_FD option shall retrieve the file descriptor associated with the specified socket. The returned file descriptor can be used to integrate the socket into an existing event loop; the ØMQ library shall signal any pending events on the socket in an edge-triggered fashion by making the file descriptor become ready for reading.

> The ability to read from the returned file descriptor does not necessarily indicate that messages are available to be read from, or can be written to, the underlying socket; applications must retrieve the actual event state with a subsequent retrieval of the ZMQ_EVENTS option.

`ZeroMQ`有两种方式向用户通知某个事件ready for read：
* 通过`int zmq_poll (zmq_pollitem_t *items, int nitems, long timeout);`，函数返回后，检测items数组中每个对象的`revent` flag，检测其是否有`ZMQ_POLLIN`标记； 
    
    ```
    typedef struct
    {
        void *socket;
        int fd;
        short events;
        short revents;
    } zmq_pollitem_t;
    ```


* 通过`zmq_getsockopt`的`ZMQ_EVENTS`选项获取flag，检测其是否有`ZMQ_POLLIN`标记；

不管哪种方式，`ZeroMQ`都是以边缘触发（edge trigger）的方式来触发事件。
此外，由于`ZeroMQ`提供的是”基于消息“的语意，因此，对于TCP这样的stream协议来说，即便某个zmq socket的底层OS socket在某个时刻的状态是`ready for read`，这也只是以为着这个fd上有数据可读，但是这些数据未必已经构成了一个业务上的消息（即发送端是以“业务消息”为维度来发送端）。因此，必须继续通过`zmq_getsockopt`的`ZMQ_EVENTS`选项获取flag，检测其是否有`ZMQ_POLLIN`标记。如果有该标记，才表明至少已经有一个业务消息可供读取。

同时，由于是边缘触发，我们在读取业务消息的时候，一定要一直读取，直到遇到了`EAGAIN`（`EWOULDBLOCK`)才能结束，否则即便该fd上有更新的数据到达，该fd也不会再触发可读事件。

下面用代码来示例：
发送者：
```
#include <memory>
#include <iostream>
#include <zmq.hpp>
#include <chrono>
#include <thread>
int main() {

    std::shared_ptr<zmq::context_t> ctx;
    std::shared_ptr<zmq::socket_t> socket;

    try {
        ctx = std::make_shared<zmq::context_t>(1);
        socket = std::make_shared<zmq::socket_t>(*ctx, ZMQ_DEALER);
        socket->connect("ipc://libevent_integration_server");

    } catch (std::exception & e) {
        std::cerr << e.what() << std::endl;
    }

    std::string data("hello world");

    std::chrono::seconds interval(1);
    while(1) {
        zmq::message_t msg(data.c_str(), data.size());
        socket->send(msg);
        std::cout << "sending msg..." << std::endl;
        std::this_thread::sleep_for(interval);
    }


    return 0;
}

```



接收者：
```
 #include <memory>
 #include <iostream>
 #include <event2/event.h>
 #include <zmq.hpp>

void business_logic(evutil_socket_t fd, short flags, void * context) {
    printf("Got an event on socket %d:%s%s%s%s\n",
           fd,
           (flags&EV_TIMEOUT) ? " timeout" : "",
           (flags&EV_READ)    ? " read" : "",
           (flags&EV_WRITE)   ? " write" : "",
           (flags&EV_SIGNAL)  ? " signal" : "");

    std::shared_ptr<zmq::socket_t> socket =  *(std::shared_ptr<zmq::socket_t>*)context;
    int flag = socket->getsockopt<int>(ZMQ_EVENTS);
    if (flag & ZMQ_POLLIN) {
        std::cout << "ZMQ_POLLIN effective" << std::endl;
        bool should_continue;
        do {
            zmq::message_t msg;
            should_continue = socket->recv(&msg, ZMQ_DONTWAIT);
            if (should_continue)
                std::cout << "ZMQ_POLLIN  msg size is:" << msg.size() << ",msg is:" << (char*)msg.data() << std::endl;
        }while(should_continue);
    } else {
        std::cout << "msg is not complete" << std::endl;
    }
}

int main() {

    std::shared_ptr<zmq::context_t> ctx;
    std::shared_ptr<zmq::socket_t> socket;

    try {
        ctx = std::make_shared<zmq::context_t>(1);
        socket = std::make_shared<zmq::socket_t>(*ctx, ZMQ_DEALER);
        socket->bind("ipc://libevent_integration_server");
    } catch (std::exception & e) {
        std::cerr << e.what() << std::endl;
    }

    struct event_base * base = event_base_new();
    if (!base)
        return -1;


    struct event * read_event = event_new(base, socket->getsockopt<int>(ZMQ_FD), EV_READ | EV_PERSIST, business_logic, (void*)&socket);

    if (!read_event)
        return -1;

    event_add(read_event, nullptr);
    event_base_dispatch(base);

    return 0;
}
```

特别要注意的是其中循环调用`recv`的部分：
```
socket->recv(&msg, ZMQ_DONTWAIT);
```
直到其返回false，即遇到`EAGAIN`之后才停止。

这样，我们就成功地将`ZeroMQ`的zmq socket融入到了`libevent`的event loop中。






