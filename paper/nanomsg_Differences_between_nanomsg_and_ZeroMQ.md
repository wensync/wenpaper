http://nanomsg.org/documentation-zeromq.html

Differences between nanomsg and ZeroMQ
======================================

Much has changed since this document was written, both in nanomsg and ZeroMQ. Nonetheless this document may be of interest to understand the historical motivations behind nanomsg in the words of the original author of both ZeroMQ and nanomsg.

自从用 nanomsg 和 ZeroMQ 编写这个文档以来，已经发生了很大的变化。尽管如此，用 ZeroMQ 和 nanomsg 的原始作者的话来说，理解 nanomsg 背后的历史动机可能很有意义。

Licensing
---------
nanomsg library is MIT-licensed. What it means is that, unlike with ZeroMQ, you can modify the source code and re-release it under a different license, as a proprietary product, etc. More reasoning about the licensing can be found here.

nanomsg 库是 MIT 授权的。它的意思是，与 ZeroMQ 不同，您可以修改源代码并在不同的许可下重新发布它，作为专有产品等等。在这里可以找到更多关于许可的推理。

POSIX Compliance
----------------
ZeroMQ API, while modeled on BSD socket API, doesn’t match the API fully. nanomsg aims for full POSIX compliance.

ZeroMQ API 虽然建模于 BSD 套接字API，但并不完全匹配该 API。nanomsg 的目标是完全遵守 POSIX。

- Sockets are represented as ints, not void pointers.

- 套接字表示为 int，而不是 void 指针。

- Contexts, as known in ZeroMQ, don’t exist in nanomsg. This means simpler API (sockets can be created in a single step) as well as the possibility of using the library for communication between different modules in a single process (think of plugins implemented in different languages speaking each to another). More discussion can be found here.

- 在 ZeroMQ 中的上下文并不存在于 nanomsg 中。这意味着更简单的 API (可以在一个步骤中创建套接字)，以及在一个进程中使用这个库在不同模块之间进行通信的可能性(想想用不同语言实现的插件，相互交流)。更多的讨论可以在这里找到。

- Sending and receiving functions (nn_send, nn_sendmsg, nn_recv and nn_recvmsg) fully match POSIX syntax and semantics.

- 发送和接收函数(nn_send、nn_sendmsg、nn_recv 和 nn_recvmsg)完全匹配 POSIX 语法和语义。

Implementation Language
-----------------------
The library is implemented in C instead of C++.

这个库是用 C 而不是 C++ 实现的。

- From user’s point of view it means that there’s no dependency on C runtime (libstdc or similar) which may be handy in constrained and embedded environments.

- 从用户的角度来看，这意味着不依赖于 C 运行时(libstdc 或类似的)，这在受限和嵌入式环境中可能很方便。

- From nanomsg developer’s point of view it makes life easier.

- 从 nanomsg 开发人员的角度来看，它使生活变得更容易。

- Number of memory allocations is drastically reduced as intrusive containers are used instead of C++ STL containers.

- 内存分配的数量急剧减少，因为使用侵入式容器，而不是 C++ STL 容器。

- The above also means less memory fragmentation, less cache misses, etc.

- 以上还意味着更少的内存碎片，更少的缓存丢失，等等。

- More discussion on the C vs. C++ topic can be found here and here.

- 关于 C 与 C++ 主题的更多讨论可以在这里和这里找到。

Pluggable Transports and Protocols
----------------------------------
In ZeroMQ there was no formal API for plugging in new transports (think WebSockets, DCCP, SCTP) and new protocols (counterparts to REQ/REP, PUB/SUB, etc.) As a consequence there were no new transports added since 2008. No new protocols were implemented either. The formal internal transport API (see transport.h and protocol.h) are meant to mitigate the problem and serve as a base for creating and experimenting with new transports and protocols.

在 ZeroMQ 中，没有用于插入新传输的正式 API(比如 WebSocket、DCCP、SCTP)和新协议(REQ/REP、PUB/SUB等的对应方)，因此自2008以来没有添加新的传输。也没有实施任何新的协议。正式的内部传输 API (参见 transport.h 和 protocol.h)旨在缓解问题，并作为创建和试验新传输和协议的基础。

Please, be aware that the two APIs are still new and may experience some tweaking in the future to make them usable in wide variety of scenarios.

请注意，这两个 API 仍然是新的，将来可能会经历一些调整，以使它们在各种不同的场景中可用。

- nanomsg implements a new SURVEY protocol. The idea is to send a message ("survey") to multiple peers and wait for responses from all of them. For more details check the article here. Also look here.

- nanomsg 实现了一项新的 SURVEY 协议。这样做的目的是向多个对等点发送一个信息(“调查”)，等待他们所有人的回复。要了解更多细节，请看这里的文章。还有看这里。

- In financial services it is quite common to use "deliver messages from anyone to everyone else" kind of messaging. To address this use case, there’s a new BUS protocol implemented in nanomsg. Check the details here.

- 在金融服务中，使用“任何人向其他人传递信息”这类信息是相当普遍的。为了解决这个用例，有一个用 nanomsg 实现的新总线协议。看看这里的细节。

Threading Model
---------------
One of the big architectural blunders I’ve done in ZeroMQ is its threading model. Each individual object is managed exclusively by a single thread. That works well for async objects handled by worker threads, however, it becomes a trouble for objects managed by user threads. The thread may be used to do unrelated work for arbitrary time span, e.g. an hour, and during that time the object being managed by it is completely stuck. Some unfortunate consequences are: inability to implement request resending in REQ/REP protocol, PUB/SUB subscriptions not being applied while application is doing other work, and similar. In nanomsg the objects are not tightly bound to particular threads and thus these problems don’t exist.

我在 ZeroMQ 中做过的一个重大架构错误是它的线程模型。每个单独的对象都由单个线程独占管理。这对于由工作线程处理的异步对象来说很好，但是对于用户线程管理的对象来说，这就成了一个麻烦。线程可以用于在任意时间跨度(例如一个小时)中执行无关的工作，而在这段时间内，由线程管理的对象被完全卡住了。一些不幸的后果是：无法在 REQ/REP 协议中实现请求重发，在应用程序进行其他工作时没有应用发布/订阅，等等。在 nanomsg 中，对象与特定的线程没有紧密的绑定，因此这些问题并不存在。

- REQ socket in ZeroMQ cannot be really used in real-world environments, as they get stuck if message is lost due to service failure or similar. Users have to use XREQ instead and implement the request re-trying themselves. With nanomsg, the re-try functionality is built into REQ socket.

- ZeroMQ 中的 REQ 套接字不能真正用于现实世界的环境，因为如果由于服务失败或类似的原因而丢失消息，它们就会陷入困境。用户必须使用 XREQ 代替，并实现请求重试自己。对于 nanomsg，重试功能被内置到 REQ 套接字中。

- In nanomsg, both REQ and REP support cancelling the ongoing processing. Simply send a new request without waiting for a reply (in the case of REQ socket) or grab a new request without replying to the previous one (in the case of REP socket).

- 在 nanomsg，REQ 和 REP 都支持取消正在进行的处理。只需发送一个新请求，而不等待回复(在 REQ 套接字情况下)，或者获取一个新请求，而不对前一个请求作出答复(在 REP 套件字的情况下)。

- In ZeroMQ, due to its threading model, bind-first-then-connect-second scenario doesn’t work for inproc transport. It is fixed in nanomsg.

- 在 ZeroMQ 中，由于其线程模型，先绑定再连接-场景不适用于 inproc 传输。它在 nanomsg 被解决了。

- For similar reasons auto-reconnect doesn’t work for inproc transport in ZeroMQ. This problem is fixed in nanomsg as well.

- 出于类似的原因，在 ZeroMQ 中，自动重新连接不适用于 inproc 传输。这个问题在 nanomsg 也解决了。

- Finally, nanomsg attempts to make nanomsg sockets thread-safe. While using a single socket from multiple threads in parallel is still discouraged, the way in which ZeroMQ sockets failed randomly in such circumstances proved to be painful and hard to debug.

- 最后，nanomsg 试图使 nanomsg 套接字线程安全。虽然仍然不鼓励并行地使用多个线程的单个套接字，但在这种情况下 ZeroMQ 套接字随机失败的方式证明是痛苦的，很难调试。

State Machines
--------------
Internal interactions inside the nanomsg library are modeled as a set of state machines. The goal is to avoid the incomprehensible shutdown mechanism as seen in ZeroMQ and thus make the development of the library easier.

nanomsg 库中的内部交互被建模为一组状态机。目标是避免 ZeroMQ 中所显示的不可理解的关闭机制，从而使库的开发更加容易。

- For more discussion see http://250bpm.com/blog:24]here and here.

IOCP Support
------------
One of the long-standing problems in ZeroMQ was that internally it uses BSD socket API even on Windows platform where it is a second class citizen. Using IOCP instead, as appropriate, would require major rewrite of the codebase and thus, in spite of multiple attempts, was never implemented. IOCP is supposed to have better performance characteristics and, even more importantly, it allows to use additional transport mechanisms such as NamedPipes which are not accessible via BSD socket API. For these reasons nanomsg uses IOCP internally on Windows platforms.

ZeroMQ 长期存在的问题之一是在内部使用 BSD 套接字 API，甚至在 Windows 平台上也是如此，因为它是二等公民。如果适当地使用IOCP，则需要对代码基进行重大重写，因此，尽管多次尝试，但从未实现。IOCP 应该具有更好的性能特性，更重要的是，它允许使用额外的传输机制，如 NamedPipes，这些机制是通过 BSD 套接字 API 无法访问的。出于这些原因，nanomsg 在 Windows 平台上内部使用 IOCP。

Level-triggered Polling
-----------------------
One of the aspects of ZeroMQ that proved really confusing for users was the ability to integrate ZeroMQ sockets into an external event loops by using ZMQ_FD file descriptor. The main source of confusion was that the descriptor is edge-triggered, i.e. it signals only when there were no messages before and a new one arrived. nanomsg uses level-triggered file descriptors instead that simply signal when there’s a message available irrespective of whether it was available in the past.

ZeroMQ 的一个方面对用户来说确实令人困惑，那就是能够使用 ZMQ_FD 文件描述符将 ZeroMQ 套接字集成到外部事件循环中。造成混乱的主要原因是描述符是边缘触发的，也就是说，只有在之前没有消息并且有新消息到达时才发出信号。nanomsg 使用级别触发的文件描述符，而只是在消息可用时发出信号，而不管消息在过去是否可用。

Routing Priorities
------------------
nanomsg implements priorities for outbound traffic. You may decide that messages are to be routed to a particular destination in preference, and fall back to an alternative destination only if the primary one is not available.

nanomsg 实现出站流量的优先级。您可以决定将消息优先路由到特定目的地，并且只有在主目的地不可用时才返回到其他目的地。

- For more discussion see here.

Asynchronous DNS
----------------
DNS queries (e.g. converting hostnames to IP addresses) are done in asynchronous manner. In ZeroMQ such queries were done synchronously, which meant that when DNS was unavailable, the whole library, including the sockets that haven’t used DNS, just hung.

DNS 查询(例如，将主机名转换为IP地址)是以异步方式完成的。在 ZeroMQ 中，这样的查询是同步进行的，这意味着当 DNS 不可用时，整个库(包括未使用 DNS 的套接字)只挂起。

Zero-Copy
---------
While ZeroMQ offers a "zero-copy" API, it’s not true zero-copy. Rather it’s "zero-copy till the message gets to the kernel boundary". From that point on data is copied as with standard TCP. nanomsg, on the other hand, aims at supporting true zero-copy mechanisms such as RDMA (CPU bypass, direct memory-to-memory copying) and shmem (transfer of data between processes on the same box by using shared memory). The API entry points for zero-copy messaging are nn_allocmsg and nn_freemsg functions in combination with NN_MSG option passed to send/recv functions.

虽然 ZeroMQ 提供了一个“零拷贝” API，但它不是真正的零拷贝。相反，它是“零复制，直到消息到达内核边界”。从那时起，数据就像标准 TCP 一样被复制。另一方面，nanomsg 的目标是支持真正的零拷贝机制，如 RDMA (CPU 旁路，直接内存到内存复制)和 shmem (使用共享内存在同一框中的进程之间传输数据)。零拷贝消息传递的 API 入口点是nn_allocmsg 和 nn_freemsg 函数，结合 NN_MSG 选项传递给 send/recv 函数。

Efficient Subscription Matching
-------------------------------
In ZeroMQ, simple tries are used to store and match PUB/SUB subscriptions. The subscription mechanism was intended for up to 10,000 subscriptions where simple trie works well. However, there are users who use as much as 150,000,000 subscriptions. In such cases there’s a need for a more efficient data structure. Thus, nanomsg uses memory-efficient version of Patricia trie instead of simple trie.

在 ZeroMQ 中，简单尝试用于存储和匹配 PUB/SUB。订阅机制是为多达 10,000 个订阅而设计的，在这些订阅中，简单的 trie 工作得很好。然而，有一些用户使用多达 150,000,000 订阅。在这种情况下，需要一个更有效的数据结构。因此，nanomsg 使用的是节省内存的Patricia trie 版本，而不是简单的 trie。

- For more details check this [http://250bpm.com/blog:19]article.

Unified Buffer Model
--------------------
ZeroMQ has a strange double-buffering behaviour. Both the outgoing and incoming data is stored in a message queue and in TCP’s tx/rx buffers. What it means, for example, is that if you want to limit the amount of outgoing data, you have to set both ZMQ_SNDBUF and ZMQ_SNDHWM socket options. Given that there’s no semantic difference between the two, nanomsg uses only TCP’s (or equivalent’s) buffers to store the data.

ZeroMQ 有一种奇怪的双缓冲行为。传出数据和传入数据都存储在消息队列和 TCP 的 tx/rx 缓冲区中。例如，它的意思是，如果您想限制传出数据量，您必须同时设置 ZMQ_SNDBUF 和 ZMQ_SNDHWM 套接字选项。考虑到两者之间没有语义差异，Nanomsg 只使用 TCP(或等效的)缓冲区来存储数据。

Scalability Protocols
---------------------
Finally, on philosophical level, nanomsg aims at implementing different "scalability protocols" rather than being a generic networking library. Specifically:

最后，在哲学层面上，nanomsg 的目标是实现不同的“可伸缩性协议”，而不是作为一个通用的网络库。具体而言：

- Different protocols are fully separated, you cannot connect REQ socket to SUB socket or similar.

- 不同的协议完全分开，不能将 REQ 套接字连接到 SUB 套接字或类似的套接字。

- Each protocol embodies a distributed algorithm with well-defined prerequisites (e.g. "the service has to be stateless" in case of REQ/REP) and guarantees (if REQ socket stays alive request will be ultimately processed).

- 每个协议都包含一个分布式算法，具有定义良好的先决条件(例如，REQ/REP 中的“服务必须是无状态的”)和保证(如果 REQ 套接字保持活动状态，最终将处理请求)。

- Partial failure is handled by the protocol, not by the user. In fact, it is transparent to the user.

- 部分故障由协议处理，而不是由用户处理。事实上，它对用户是透明的。

- The specifications of the protocols are in /rfc subdirectory.

- 协议的规范在 /rfc 子目录中。

- The goal is to standardise the protocols via IETF.

- 目标是通过 IETF 实现协议标准化。

- There’s no generic UDP-like socket (ZMQ_ROUTER), you should use L4 protocols for that kind of functionality.

- 没有通用的类似 UDP 的套接字(ZMQ_ROUTER)，您应该使用 L4 协议来实现这种功能。