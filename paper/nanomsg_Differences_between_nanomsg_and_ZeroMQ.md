http://nanomsg.org/documentation-zeromq.html

Differences between nanomsg and ZeroMQ
======================================

Much has changed since this document was written, both in nanomsg and ZeroMQ. Nonetheless this document may be of interest to understand the historical motivations behind nanomsg in the words of the original author of both ZeroMQ and nanomsg.

Licensing
---------
nanomsg library is MIT-licensed. What it means is that, unlike with ZeroMQ, you can modify the source code and re-release it under a different license, as a proprietary product, etc. More reasoning about the licensing can be found here.

POSIX Compliance
----------------
ZeroMQ API, while modeled on BSD socket API, doesn’t match the API fully. nanomsg aims for full POSIX compliance.

- Sockets are represented as ints, not void pointers.

- Contexts, as known in ZeroMQ, don’t exist in nanomsg. This means simpler API (sockets can be created in a single step) as well as the possibility of using the library for communication between different modules in a single process (think of plugins implemented in different languages speaking each to another). More discussion can be found here.

- Sending and receiving functions (nn_send, nn_sendmsg, nn_recv and nn_recvmsg) fully match POSIX syntax and semantics.

Implementation Language
-----------------------
The library is implemented in C instead of C++.

- From user’s point of view it means that there’s no dependency on C runtime (libstdc or similar) which may be handy in constrained and embedded environments.

- From nanomsg developer’s point of view it makes life easier.

- Number of memory allocations is drastically reduced as intrusive containers are used instead of C++ STL containers.

- The above also means less memory fragmentation, less cache misses, etc.

- More discussion on the C vs. C++ topic can be found here and here.

Pluggable Transports and Protocols
----------------------------------
In ZeroMQ there was no formal API for plugging in new transports (think WebSockets, DCCP, SCTP) and new protocols (counterparts to REQ/REP, PUB/SUB, etc.) As a consequence there were no new transports added since 2008. No new protocols were implemented either. The formal internal transport API (see transport.h and protocol.h) are meant to mitigate the problem and serve as a base for creating and experimenting with new transports and protocols.

Please, be aware that the two APIs are still new and may experience some tweaking in the future to make them usable in wide variety of scenarios.

- nanomsg implements a new SURVEY protocol. The idea is to send a message ("survey") to multiple peers and wait for responses from all of them. For more details check the article here. Also look here.</li>

- In financial services it is quite common to use "deliver messages from anyone to everyone else" kind of messaging. To address this use case, there’s a new BUS protocol implemented in nanomsg. Check the details here.

Threading Model
---------------
One of the big architectural blunders I’ve done in ZeroMQ is its threading model. Each individual object is managed exclusively by a single thread. That works well for async objects handled by worker threads, however, it becomes a trouble for objects managed by user threads. The thread may be used to do unrelated work for arbitrary time span, e.g. an hour, and during that time the object being managed by it is completely stuck. Some unfortunate consequences are: inability to implement request resending in REQ/REP protocol, PUB/SUB subscriptions not being applied while application is doing other work, and similar. In nanomsg the objects are not tightly bound to particular threads and thus these problems don’t exist.

- REQ socket in ZeroMQ cannot be really used in real-world environments, as they get stuck if message is lost due to service failure or similar. Users have to use XREQ instead and implement the request re-trying themselves. With nanomsg, the re-try functionality is built into REQ socket.

- In nanomsg, both REQ and REP support cancelling the ongoing processing. Simply send a new request without waiting for a reply (in the case of REQ socket) or grab a new request without replying to the previous one (in the case of REP socket).

- In ZeroMQ, due to its threading model, bind-first-then-connect-second scenario doesn’t work for inproc transport. It is fixed in nanomsg.

- For similar reasons auto-reconnect doesn’t work for inproc transport in ZeroMQ. This problem is fixed in nanomsg as well.

- Finally, nanomsg attempts to make nanomsg sockets thread-safe. While using a single socket from multiple threads in parallel is still discouraged, the way in which ZeroMQ sockets failed randomly in such circumstances proved to be painful and hard to debug.

State Machines
--------------
Internal interactions inside the nanomsg library are modeled as a set of state machines. The goal is to avoid the incomprehensible shutdown mechanism as seen in ZeroMQ and thus make the development of the library easier.

- For more discussion see http://250bpm.com/blog:24]here and here.

IOCP Support
------------
One of the long-standing problems in ZeroMQ was that internally it uses BSD socket API even on Windows platform where it is a second class citizen. Using IOCP instead, as appropriate, would require major rewrite of the codebase and thus, in spite of multiple attempts, was never implemented. IOCP is supposed to have better performance characteristics and, even more importantly, it allows to use additional transport mechanisms such as NamedPipes which are not accessible via BSD socket API. For these reasons nanomsg uses IOCP internally on Windows platforms.

Level-triggered Polling
-----------------------
One of the aspects of ZeroMQ that proved really confusing for users was the ability to integrate ZeroMQ sockets into an external event loops by using ZMQ_FD file descriptor. The main source of confusion was that the descriptor is edge-triggered, i.e. it signals only when there were no messages before and a new one arrived. nanomsg uses level-triggered file descriptors instead that simply signal when there’s a message available irrespective of whether it was available in the past.

Routing Priorities
------------------
nanomsg implements priorities for outbound traffic. You may decide that messages are to be routed to a particular destination in preference, and fall back to an alternative destination only if the primary one is not available.

- For more discussion see here.

Asynchronous DNS
----------------
DNS queries (e.g. converting hostnames to IP addresses) are done in asynchronous manner. In ZeroMQ such queries were done synchronously, which meant that when DNS was unavailable, the whole library, including the sockets that haven’t used DNS, just hung.

Zero-Copy
---------
While ZeroMQ offers a "zero-copy" API, it’s not true zero-copy. Rather it’s "zero-copy till the message gets to the kernel boundary". From that point on data is copied as with standard TCP. nanomsg, on the other hand, aims at supporting true zero-copy mechanisms such as RDMA (CPU bypass, direct memory-to-memory copying) and shmem (transfer of data between processes on the same box by using shared memory). The API entry points for zero-copy messaging are nn_allocmsg and nn_freemsg functions in combination with NN_MSG option passed to send/recv functions.

Efficient Subscription Matching
-------------------------------
In ZeroMQ, simple tries are used to store and match PUB/SUB subscriptions. The subscription mechanism was intended for up to 10,000 subscriptions where simple trie works well. However, there are users who use as much as 150,000,000 subscriptions. In such cases there’s a need for a more efficient data structure. Thus, nanomsg uses memory-efficient version of Patricia trie instead of simple trie.

- For more details check this [http://250bpm.com/blog:19]article.

Unified Buffer Model
--------------------
ZeroMQ has a strange double-buffering behaviour. Both the outgoing and incoming data is stored in a message queue and in TCP’s tx/rx buffers. What it means, for example, is that if you want to limit the amount of outgoing data, you have to set both ZMQ_SNDBUF and ZMQ_SNDHWM socket options. Given that there’s no semantic difference between the two, nanomsg uses only TCP’s (or equivalent’s) buffers to store the data.

Scalability Protocols
---------------------
Finally, on philosophical level, nanomsg aims at implementing different "scalability protocols" rather than being a generic networking library. Specifically:

- Different protocols are fully separated, you cannot connect REQ socket to SUB socket or similar.

- Each protocol embodies a distributed algorithm with well-defined prerequisites (e.g. "the service has to be stateless" in case of REQ/REP) and guarantees (if REQ socket stays alive request will be ultimately processed).

- Partial failure is handled by the protocol, not by the user. In fact, it is transparent to the user.

- The specifications of the protocols are in /rfc subdirectory.

- The goal is to standardise the protocols via IETF.

- There’s no generic UDP-like socket (ZMQ_ROUTER), you should use L4 protocols for that kind of functionality.