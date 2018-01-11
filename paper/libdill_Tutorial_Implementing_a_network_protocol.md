Tutorial: Implementing a network protocol
=========================================

Introduction
------------

In this tutorial you will learn how to implement a simple network protocol using libdill.

在本教程中，您将学习如何使用libdill实现简单的网络协议。

The source code for individual steps of this tutorial can be found in tutorial/protocol subdirectory.

本教程的各个步骤的源代码可以在 tutorial/protocol 子目录中找到。

Step 1: Creating a handle
-------------------------

Handles are libdill's equivalent of file descriptors. A socket implementing our protocol will be pointed to by a handle. Therefore, we have to learn how to create custom handle types.

句柄是 libdill 的文件描述符。实现我们协议的套接字将由句柄指向。因此，我们必须学习如何创建自定义句柄类型。

Standard include file for libdill is libdill.h . In this case, however, we will include libdillimpl.h which defines all the functions libdill.h does but also adds some extra stuff that can be used to implement different plugins to libdill, such as new handle types or new socket types:

标准的 libdill 文件是 libdill.h。但是，在本例中，我们将包括 libdilimpl 定义了所有 libdill.h 定义的函数，而且还添加了一些额外的东西，可以用来实现 libdill 的不同插件，例如新句柄类型或新套接字类型：


```
#include <libdillimpl.h> 
```

To make it clear what API we are trying to implement, let's start with a simple test program. We'll call our protocol quux and at this point we will just open and close the handle:

为了弄清楚我们要实现什么 API，让我们从一个简单的测试程序开始。我们将调用我们的协议 quux，此时我们将打开并关闭句柄：

```
int main(void) {
    int h = quux_open();
    assert(h >= 0);
    int rc = hclose(h);
    assert(rc == 0);
    return 0;
}
```

To start with the implementation we need a structure to hold data for the handle. At the moment it will contain only handle's virtual function table:

为了开始实现，我们需要一个结构来保存句柄的数据。目前，它将只包含句柄的虚拟函数表：

```
struct quux {
    struct hvfs hvfs;
};
```

Let's add forward declarations for functions that will be filled into the virtual function table. We'll learn what each of them is good for shortly:

让我们为将被填充到虚拟函数表中的函数添加前向声明。我们将很快了解它们各自的优点：

```
static void *quux_hquery(struct hvfs *hvfs, const void *id);
static void quux_hclose(struct hvfs *hvfs);
static int quux_hdone(struct hvfs *hvfs, int64_t deadline);
```

The quux_open function itself won't do much except for allocating the object, filling in the table of virtual functions and registering it with libdill runtime:

quux_open 函数本身除了分配对象、填充虚拟函数表并向 libdill 运行时注册外，不会做太多事情：

```
int quux_open(void) {
    int err;
    struct quux *self = malloc(sizeof(struct quux));
    if(!self) {err = ENOMEM; goto error1;}
    self->hvfs.query = quux_hquery;
    self->hvfs.close = quux_hclose;
    self->hvfs.done = quux_hdone;
    int h = hmake(&self->hvfs);
    if(h < 0) {err = errno; goto error2;}
    return h;
error2:
    free(self);
error1:
    errno = err;
    return -1;
}
```

Function hmake() does the trick. You pass it a virtual function table and it returns a handle. When the standard function like hclose() is called on the handle libdill will forward the call to the corresponding virtual function, in this particular case to quux_hclose() . The interesting part is that the first argument to the virtual function is no longer the handle but rather pointer to the virtual function table. And given that virtual function table is a member of struct quux it's easy to convert it to the pointer to the quux object itself.

函数 hmake() 起了作用。传递给它一个虚拟函数表，它返回一个句柄。当在句柄上调用标准函数(如 hclose())时，libdill 将将调用转发到相应的虚拟函数，在这种情况下，调用将转到 quux_hclose()。有趣的是，虚拟函数的第一个参数不再是句柄，而是指向虚拟函数表的指针。并且，由于虚拟函数表是 struct quux 的成员，因此很容易将其转换为指向 quux 对象本身的指针。

quux_hclose() virtual function will deallocate the quux object:

quux_hclose() 虚拟函数将释放 quux 对象：

```
static void quux_hclose(struct hvfs *hvfs) {
    struct quux *self = (struct quux*)hvfs;
    free(self);
}
```

At the moment we can just return ENOTSUP from the other two virtual functions.

目前，我们可以从其他两个虚拟函数返回 ENOTSUP。

Compile the file and run it to test whether it works as expected:

编译该文件并运行它，以测试它是否按预期工作：

```
$ gcc -o step1 step1.c -ldill
```

Step 2: Implementing an operation on the handle
-----------------------------------------------

Consider a UDP socket. It is actually multiple things. It's a handle and as such it exposes functions like hclose() . It's a message socket and as such it exposes functions like msend() and mrecv() . It's also a UDP socket per se and as such it exposes functions like udp_send() or udp_recv() .

考虑一个 UDP 套接字。其实是多方面的。它是一个句柄，因此它公开类似 hclose() 之类的函数。它是一个消息套接字，因此它公开像 msend() 和 mrecv() 这样的函数。它本身也是一个 UDP 套接字，因此它公开了像 udp_send() 或 udp_recv() 这样的函数。

libdill provides an extremely generic mechanism to address this multi-faceted nature of handles. In fact, the mechanism is so generic that it's almost silly. There's a hquery() function which takes ID as an argument and returns a void pointer. No assumptions are made about the nature of the ID or nature of the returned pointer. Everything is completely opaque.

libdill 提供了一个非常通用的机制来解决这种多面处理的性质。事实上，这种机制是如此的普遍，以至于它几乎是愚蠢的。有一个 hquery() 函数，它以 ID 作为参数并返回一个空指针。对于 ID 的性质或返回指针的性质不作任何假设。一切都是完全不透明的。

To see how that can be useful let's implement a new quux function.

为了了解这是如何有用的，让我们实现一个新的 quux 函数。

First, we have to define an ID for quux object type. Now, this may be a bit confusing, but the ID is actually a void pointer. The advantage of using a pointer as an ID is that if it was an integer you would have to worry about ID collisions, especially if you defined IDs in different libraries that were then linked together. With pointers there's no such problem. You can take a pointer to a global variable and it is guaranteed to be unique as two pieces of data can't live at the same memory location.

首先，我们必须为 quux 对象类型定义一个 ID。现在，这可能有点混乱，但 ID 实际上是一个空指针。使用指针作为 ID 的优点是，如果指针是整数，则必须担心 ID 冲突，特别是如果在不同库中定义了 ID，然后将它们链接在一起。有了指针，就没有这样的问题了。您可以使用指向全局变量的指针，并且保证它是唯一的，因为两个数据不能驻留在同一个内存位置。

```
static const int quux_type_placeholder = 0;
static const void *quux_type = &quux_type_placeholder;
```

Second, let's implement hquery() virtual function, empty implementation of which has been created in previous step of this tutorial:

其次，让我们实现 hquery() 虚拟函数，该函数的空实现已经在本教程的前面步骤中创建：

```
static void *quux_hquery(struct hvfs *hvfs, const void *type) {
    struct quux *self = (struct quux*)hvfs;
    if(type == quux_type) return self;
    errno = ENOTSUP;
    return NULL;
}
```

To understand what this is good for think of it from user's perspective: You call hquery() function and pass it a handle and the ID of quux handle type. The function will fail with ENOTSUP if the handle is not a quux handle. It will return pointer to quux structure if it is. You can use the returned pointer to perform useful work on the object.

要理解这对用户的好处是什么，请从用户的角度考虑：调用 hquery() 函数并传递给它一个句柄和 quux 句柄类型的ID。如果句柄不是 quux 句柄，函数将以 ENOTSUP 失败。如果是，它将返回指向 quux 结构的指针。可以使用返回的指针对对象执行有用的工作。

But wait! Doesn't that break encapsulation? Anyone can call hquery() function, get the pointer to raw quux object and mess with it in unforeseen ways.

但是等等！这不是破坏了封装吗？任何人都可以调用 hquery() 函数，获取指向原始 quux 对象的指针，并以不可预见的方式处理它。

But no. Note that quux_type is defined as static. The ID is not available anywhere except in the file that implements quux handle. No external code will be able to get the raw object pointer. The encapsulation works as expected after all.

但是没有。请注意 quux_type 被定义为静态的。除了实现 quux 句柄的文件外，任何地方都不能使用 ID。没有外部代码能够获得原始对象指针。毕竟，封装的工作方式与预期的一样。

All that being said, we can finally implement our new user-facing function:

尽管如此，我们终于可以实现我们新的面向用户的函数：

```
int quux_frobnicate(int h) {
    struct quux *self = hquery(h, quux_type);
    if(!self) return -1;
    printf("Kilroy was here!\n");
    return 0;
}
```

Modify the test to call the new function, compile it and run it:

修改测试以调用新函数，编译它并运行它：

```
int main(void) {
    ...
    int rc = quux_frobnicate(h);
    assert(rc == 0);
    ...
}
```

Step 3: Attaching and detaching a socket
----------------------------------------

Let's turn the quux handle that we have implemented into quux network socket now.

让我们现在把我们已经实现的quux句柄转换成 quux 网络套接字。

We won't implement the entire network stack from scratch. The user will layer the quux socket on top of an existing bytestream protocol, such as TCP. The layering will be done via quux_attach() and quux_detach() functions.

我们不会从头开始实现整个网络栈。用户将在现有的字节协议(如 TCP)之上层上层间套接字。分层将通过 quux_attach() 和 quux_detach() 函数完成。

In the test program we are going to layer quux protocol on top of ipc protocol (UNIX domain sockets):

在测试程序中，我们将在 ipc 协议(UNIX 域套接字)之上进行分层协议：

```
coroutine void client(int s) {
    int q = quux_attach(s);
    assert(q >= 0);
    /* Do something useful here! */
    s = quux_detach(q);
    assert(s >= 0);
    int rc = hclose(s);
    assert(rc == 0);
}

int main(void) {
    int ss[2];
    int rc = ipc_pair(ss);
    assert(rc == 0);
    go(client(ss[0]));
    int q = quux_attach(ss[1]);
    assert(q >= 0);
    /* Do something useful here! */
    int s = quux_detach(q);
    assert(s >= 0);
    rc = hclose(s);
    assert(rc == 0);
    return 0;
}
```

To implement it the socket first need to remember the handle of the underlying protocol so that it can use it when sending and receiving data:

要实现它，套接字首先需要记住底层协议的句柄，这样它就可以在发送和接收数据时使用它：

```
struct quux {
    ...
    int u;
};
```

To accomodate libdill's naming conventions for protocols running on top of other protocols, we have to rename quux_open() to quux_attach() . Attach function will accept a handle of the underlying bytestream protocol and create quux protocol on top of it:

为了适应 libdill 对运行在其他协议之上的协议的命名约定，我们必须将 quux_open() 重命名为 quux_attach()。attach 函数将接受底层字节流协议的句柄，并在其之上创建 quux 协议：


```
int quux_attach(int u) {
    int err;
    struct quux *self = malloc(sizeof(struct quux));
    if(!self) {err = ENOMEM; goto error1;}
    self->hvfs.query = quux_hquery;
    self->hvfs.close = quux_hclose;
    self->hvfs.done = quux_hdone;
    self->u = u;
    int h = hmake(&self->hvfs);
    if(h < 0) {int err = errno; goto error2;}
    return h;
error2:
    free(self);
error1:
    errno = err;
    return -1;
}
```

We can reuse quux_frobnicate() and rename it to quux_detach() . It will terminate the quux protocol and return the handle of the underlying protocol:

我们可以重用 quux_frobnicate() 并将其重命名为 quux_detach()。它将终止 quux 协议并返回底层协议的句柄：

```
int quux_detach(int h) {
    struct quux *self = hquery(h, quux_type);
    if(!self) return -1;
    int u = self->u;
    free(self);
    return u;
}
```

Step 4: Exposing the socket interface
-------------------------------------


libdill recognizes two kinds of network sockets: bytestream sockets and messages sockets. The main difference between the two is that the latter preserves the message boundaries while the former does not.

libdill识别两种网络套接字：字节流套接字和消息套接字。两者的主要区别是后者保留消息边界，而前者不保留消息边界。

Let's say quux will be a message-based protocol. As such, it should expose functions like msend and mrecv to the user.

假设 quux 将是一个基于消息的协议。因此，它应该向用户公开 msend 和 mrecv 之类的函数。

Do you remember the trick with adding new function to a handle? If not so, re-read step 2 of this tutorial. For adding message socket functions to quux we'll use the same mechanism except that instead of defining our own type ID we will use an existing one ( msock_type ) defined by libdill. Passing msock_type to hquery function will return pointer to a virtual function table of type msock_vfs . The table is once again defined by libdill:

您还记得在句柄中添加新函数的技巧吗？如果不是这样，请重新阅读本教程的步骤 2。对于向 quux 添加消息套接字函数，我们将使用相同的机制，除了使用 libdill 定义的现有类型 ID (msock_type)而不是定义自己的类型 ID。将 msock_type 传递给 hquery 函数将返回指向msock_vfs 类型的虚拟函数表的指针。该表再次由 libdill 定义：

```
struct msock_vfs {
    int (*msendl)(struct msock_vfs *vfs,
        struct iolist *first, struct iolist *last, int64_t deadline);
    ssize_t (*mrecvl)(struct msock_vfs *vfs,
        struct iolist *first, struct iolist *last, int64_t deadline);
};
```

To implement the functions, we have to first add the virtual function table to quux socket object:

要实现这些函数，我们必须首先将虚拟函数表添加到 quux 套接字对象：

```
struct quux {
    struct hvfs hvfs;
    struct msock_vfs mvfs;
    int u;
};
```

We'll need forward declarations for msock functions:

我们需要对 msock 函数进行前向声明：

```
static int quux_msendl(struct msock_vfs *mvfs,
    struct iolist *first, struct iolist *last, int64_t deadline);
static ssize_t quux_mrecvl(struct msock_vfs *mvfs,
    struct iolist *first, struct iolist *last, int64_t deadline);
```

And we have to fill in the virtual function table inside quux_attach() function:

我们必须在 quux_attach() 函数中填充虚拟函数表：

```
int quux_attach(int u) {
    ...
    self->mvfs.msendl = quux_msendl;
    self->mvfs.mrecvl = quux_mrecvl;
    ...
}
```

Return the pointer to the msock virtual function table when hquery() is called with msock_type type ID:

当使用 msock_type 类型 ID 调用 hquery() 时，返回指向 msock 虚拟函数表的指针：

```
static void *quux_hquery(struct hvfs *hvfs, const void *type) {
    struct quux *self = (struct quux*)hvfs;
    if(type == msock_type) return &self->mvfs;
    if(type == quux_type) return self;
    errno = ENOTSUP;
    return NULL;
}
```

Note that quux_msendl() and quux_mrecvl() receive pointer to msock virtual function table which is not the same as the pointer to quux object. We'll have to convert it. To make that easier let's define this handy macro. It will convert pointer to an embedded structure to a pointer of the enclosing structure:

注意，quux_msendl() 和 quux_mrecvl() 接收指向 msock 虚拟函数表的指针，这与指向 quux 对象的指针不同。我们得把它改过来。为了使这更容易，让我们定义这个方便的宏。它将将指向嵌入式结构的指针转换为包围结构的指针：

```
#define cont(ptr, type, member) \
    ((type*)(((char*) ptr) - offsetof(type, member)))
```

Using the cont macro we can convert the mvfs into pointer to quux object. Here, for instance, is a stub implementation of quux_msendl() function:

使用 cont 宏，我们可以将 mvfs 转换为指向 quux 对象的指针。例如，这里是 quux_msendl() 函数的存根实现：

```
static int quux_msendl(struct msock_vfs *mvfs,
      struct iolist *first, struct iolist *last, int64_t deadline) {
    struct quux *self = cont(mvfs, struct quux, mvfs);
    errno = ENOTSUP;
    return -1;
}
```

Add a similar stub implementation for quux_mrecvl() function and proceed to the next step.

为 quux_mrecvl() 函数添加类似的存根实现，然后继续下一步。

Step 5: Sending a message
-------------------------

Now that we've spent previous four steps doing boring scaffolding work we can finally do some fun protocol stuff.

现在我们已经花了四个步骤做无聊的脚手架工作，我们终于可以做一些有趣的协议事务了。

Modify the client part of the test program to send a simple text message:

修改测试程序的客户端部分以发送简单的文本消息：

```
coroutine void client(int s) {
    ...
    int rc = msend(q, "Hello, world!", 13, -1);
    assert(rc == 0);
    ...
}
```

Function msend will forward the call to our quux_msendl() stub function which, at the moment, does nothing but returns ENOTSUP error. To implement it we have to decide how the quux network protocol will actually look like.

函数 msend 将把调用转发到 quux_msendl() 存根函数，该函数目前只返回 ENOTSUP 错误。为了实现它，我们必须决定 quux 网络协议的实际样子。

Let's say it will prefix individual messages with 8-bit size. The test message from above is going to look like this on the wire:

假设它将以 8 位大小在单个消息前加上前缀。上面的测试消息将如下所示：

[image]

Given that size field is a single byte messages can be at most 255 bytes long. That may be a problem in the real world but this is just a tutorial so let's ignore it and move on.

考虑到 size 字段是单个字节，消息最多可以有 255 个字节长。这可能是现实世界中的一个问题，但这只是一个教程，所以让我们忽略它，继续前进。

Note that payload data is passed to the quux_msendl() function in form of two pointers to a structure called iolist .

注意，有效载荷数据以两个指针的形式传递给 quux_msendl() 函数，该结构称为 iolist。

Iolists are libdill's alternative to POSIX iovecs. Where iovecs are arrays of buffers (so called scatter/gather arrays) iolists are linked lists of buffers. Very much like iovec, iolist has iol_base pointer pointing to the data and iol_len field containing the size of the data. Unlike iovec though it has also iol_next field which points to the next buffer in the list. iol_next of the last item in the list is set to NULL .

iolist 是 libdill 对 POSIX iovecs 的替代。其中 iovecs 是缓冲区数组(所谓的分散/聚集数组)，iolist 是链接的缓冲区列表。与 iovec 非常类似，iolist 有指向数据的 iol_base 指针和包含数据大小的 iol_len 字段。与 iovec 不同，它还有 iol_next 字段，它指向列表中的下一个缓冲区。列表中最后一项的 iol_next 设置为 NULL。

An iolist instance may look like this:

一个 iolist 实例可能如下所示：

[image]

Don't forget that there's an extra field in iolist, one called iol_rsvd , which should be always set to zero.

不要忘记，iolist 中有一个额外的字段，一个名为 iol_rsvd，它应该始终设置为零。

Given that we decided to prefix quux messages with message size we have to compute the message size first. We can do so by iterating over the iolist and summing all the buffer sizes. If size is greater than 254 (we'll use number 255 for a special purpose later on) it's an error.

考虑到我们决定用消息大小作为 quux 消息的前缀，我们必须首先计算消息大小。我们可以通过迭代 iolist 和所有缓冲区大小来实现这一点。如果大小大于254(稍后我们将使用数字 255 进行特殊用途)，这是一个错误。

```
size_t sz = 0;
struct iolist *it;
for(it = first; it; it = it->iol_next)
    sz += it->iol_len;
if(sz > 254) {errno = EMSGSIZE; return -1;}
```

Now we can send the size and the payload to the underlying socket:

现在，我们可以将大小和有效载荷发送到底层套接字：

```
uint8_t c = (uint8_t)sz;
int rc = bsend(self->u, &c, 1, deadline);
if(rc < 0) return -1;
rc = bsendl(self->u, first, last, deadline);
if(rc < 0) return -1;
```

To indicate success return zero from the function. Then compile and test.

若要指示成功，请从函数返回零。然后编译和测试。

However, let's suppose we want to implement a high-performance protocol. Looking at the code, it's not hard to spot that there's quite a serious performance problem: For each message, the underlying network stack is traversed twice, one for the size byte, second time for the payload. Instead of doing two send calls and traversing the underlying network stack twice we can modify the iolist to include the size byte as well as the payload and send the entire message using a single bsendl() call. The modified iolist may look as follows. Grey parts are the original iolist as passed to quux_msendl() by the user. Black part is the modification done inside quux_msendl() :

然而，让我们假设我们想要实现一个高性能的协议。看一下代码，不难发现有一个相当严重的性能问题：对于每条消息，底层的网络堆栈会被遍历两次，一次是大小字节，第二次是有效载荷。我们可以修改 iolist 以包含大小字节以及有效载荷，而不是执行两个发送调用和遍历底层网络栈，然后使用单个 bsendl() 调用发送整个消息。修改后的 iolist 可能如下所示。灰色部分是用户传递给 quux_msendl() 的原始 iolist。黑色部分是 quux_msendl() 中所做的修改：

[image]

The code will look like this:

代码将如下所示：

```
uint8_t c = (uint8_t)sz;
struct iolist hdr = {&c, 1, first, 0};
int rc = bsendl(self->u, &hdr, last, deadline);
if(rc < 0) return -1;
```


Step 6: Receiving a message
---------------------------

Add the lines to receive a message to the test. Note that mrecvl() function returns size of the message:

将接收消息的行添加到测试中。注意，mrecvl() 函数返回消息的大小：

```
int main(void) {
    ...
    char buf[256];
    ssize_t sz = mrecv(q, buf, sizeof(buf), -1);
    assert(sz >= 0);
    printf("%.*s\n", (int)sz, buf);
    ...
}
```

To implement the receive function we'll have to read the 8-bit size first:

要实现接收函数，我们必须首先读取 8 位大小：

```
uint8_t sz;
int rc = brecv(self->u, &sz, 1, deadline);
if(rc < 0) return -1;
```

Great. Now we know that the message is sz bytes long. We are going to read those bytes into user's buffers. But let's take care of one special case first.

太棒了。现在我们知道消息是 sz 字节长。我们将把这些字节读入用户的缓冲区。但让我们先处理一个特例。

libdill allows iolist passed to the receive function to be NULL . What that means is that the user wants to skip a message. This can be easily done given that the underlying protocol provides similar skipping functionality:

libdill 允许传入接收函数的 iolist 为空。这意味着用户想跳过一条消息。考虑到底层协议提供了类似的跳过功能，这很容易做到：

```
if(!first) {
    rc = brecv(self->u, NULL, sz, deadline);
    if(rc < 0) return -1;
    return sz;
}
```

If the iolist is not NULL we will find out the overall size of the buffer and check whether message will fit into it:

如果 iolist 不是 NULLL，我们将找出缓冲区的总体大小，并检查消息是否适合它：

```
size_t bufsz = 0;
struct iolist *it;
for(it = first; it; it = it->iol_next)
    bufsz += it->iol_len;
if(bufsz < sz) {errno = EMSGSIZE; return -1;}
```

It the message fits into the buffer payload can be received:

如果消息适合缓冲区有效载荷，则可以接收：

```
size_t rmn = sz;
for(it = first; it; it = it->iol_next) {
    size_t torecv = rmn < it->iol_len ? rmn : it->iol_len;
    rc = brecv(self->u, it->iol_base, torecv, deadline);
    if(rc < 0) return -1;
    rmn -= torecv;
    if(rmn == 0) break;
}
return sz;
```

We are facing the same performance problem here as we did in the send function. Calling brecv() multiple times means extra network stack traversals and thus decreased performance. And we can apply a similar solution. We can modify the iolist in such a way that it can be simply forwarded to the underlying socket.

我们在这里面临的性能问题和发送函数中的性能问题是一样的。多次调用 brecv() 意味着额外的网络堆栈遍历，从而降低性能。我们可以应用类似的解决方案。我们可以修改 iolist，使其可以简单地被转发到基础套接字。

The main problem in this case is that the size of the message may not match the size of the buffer supplied by the user. If message is larger than the buffer we will simply return an error. However, if message is smaller than the buffer there's a problem. Bytestream's brecvl() function has no size argument. It just receives data until the buffer is completely full. Therefore, we will have to shrink the buffer to match the message size.

这种情况下的主要问题是消息的大小可能与用户提供的缓冲区大小不匹配。如果消息大于缓冲区，我们只需返回一个错误。但是，如果消息小于缓冲区，则存在问题。字节流的 brecvl() 函数没有大小参数。它只接收数据，直到缓冲区完全满。因此，我们必须缩小缓冲区以匹配消息大小。

Let's say the iolist supplied by the user looks like this:

假设用户提供的 iolist 如下所示：

[image]

If the message is 5 bytes long we are going to modify the iolist like this. Grey parts are the iolist structures passed to quux socket by the user. Black parts are the modifications:

如果消息有 5 个字节长，我们将像这样修改 iolist。灰色部分是用户传递给 quux 套接字的 iolist 结构。黑色部分是改动：

[image]

In iolist-based functions the list is supposed to be unchanged when the function returns. However, it can be temporarily modified while the function is in progress. Therefore, iolists are not guaranteed to be thread- or coroutine-safe.

在基于 iolist 的函数中，当函数返回时，列表应该保持不变。但是，在函数进行过程中可以暂时修改它。因此，iolists 不一定是线程或协程安全的。

We are modifying the iolist here, which is all right, but we have to make sure to revert all our changes before the function returns. Note the orig pointer in the above diagram. It won't be passed to the underlying socket. Instead, it will be used to restore the original iolist after sending is done.

我们在这里修改 iolist，这很好，但是我们必须确保在函数返回之前还原所有的更改。注意上面图表中的原始指针。它不会传递给底层套接字。相反，它将用于在发送完成后恢复原始 iolist。

The code for the above looks like this:

上面的代码如下所示：

```
size_t rmn = sz;
it = first;
while(1) {
    if(it->iol_len >= rmn) break;
    rmn -= it->iol_len;
    it = it->iol_next;
    if(!it) {errno = EMSGSIZE; return -1;}
}
struct iolist orig = *it;
it->iol_len = rmn;
it->iol_next = NULL;
rc = brecvl(self->u, first, last, deadline);
*it = orig;
if(rc < 0) return -1;
return sz;
```

Compile and run!

编译并运行！


Step 7: Error handling
----------------------

Consider the following scenario: User wants to receive a message. The message is 200 bytes long. However, after reading 100 bytes, receive function times out. That puts you, as the protocol implementor, into an unconfortable position. There's no way to push the 100 bytes that were already received back to the underlying socket. libdill sockets provide no API for that, but even in principle, it would mean that the underlying socket would need an unlimited buffer in case the user wanted to push back one terrabyte of data.

考虑以下场景：用户希望接收消息。消息有 200 个字节长。但是，读取 100 个字节后，接收函数超时。这使您，作为协议实现者，处于一个无法控制的位置。无法将已经接收到的 100 个字节推送回基础套接字。libdill套接字没有为此提供API，但即使在原则上，这也意味着底层套接字需要一个无限的缓冲区，以防用户想要推回一兆字节的数据。

libdill solves this problem by not trying too hard to recover from errors, even from seemingly recoverable ones like ETIMEOUT.

libdill 通过不太努力地从错误中恢复，甚至是从像 ETIMEOUT 这样的看似可恢复的错误中解决了这个问题。

When building on top of a bytestream protocol -- which in unrecoverable by definition -- you thus have to track failures and once error happens return an error for any subsequent attampts to receive a message. And same reasoning and same solution applies to outbound messages.

当构建在字节流协议之上时 -- 根据定义，这是不可恢复的 -- 因此，您必须跟踪故障，一旦错误发生，返回一个错误，以便后续的攻击程序接收消息。同样的推理和同样的解决方案也适用于出站消息。

Note that this does not apply when you are building on top of a message socket. Message sockets may be recoverable. Consider UDP. If receiving one packet fails you can still receive the next packet.

请注意，当您构建在消息套接字之上时，这并不适用。消息套接字可以恢复。考虑 UDP。如果接收一个数据包失败，您仍然可以接收下一个数据包。

Anyway, to implement error handling in quux protocol, let's add two booleans to the socket to track whether sending/receiving had failed already:

无论如何，要在 quux 协议中实现错误处理，让我们向套接字中添加两个布尔值，以跟踪发送/接收是否已经失败：

```
struct quux {
    ...
    int senderr;
    int recverr;
};
```

Initialize them to false in quux_attach() function:

在quux_attach() 函数中将它们初始化为 false：

```
self->senderr = 0;
self->recverr = 0;
```

Set senderr to true every time when sending fails and recverr to true every time when receiving fails. For example:

每次发送失败时将 senderr 设置为 true，每次接收失败时将 recverr 设置为 true。例如：

```
if(sz > 254) {self->senderr = 1; errno = EMSGSIZE; return -1;}
```

Finally, fail send and receive function if the error flag is set:

最后，如果设置了错误标志，则失败发送和接收函数：

```
static int quux_msendl(struct msock_vfs *mvfs,
      struct iolist *first, struct iolist *last, int64_t deadline) {
    struct quux *self = cont(mvfs, struct quux, mvfs);
    if(self->senderr) {errno = ECONNRESET; return -1;}
    ...
}

static ssize_t quux_mrecvl(struct msock_vfs *mvfs,
      struct iolist *first, struct iolist *last, int64_t deadline) {
    struct quux *self = cont(mvfs, struct quux, mvfs);
    if(self->recverr) {errno = ECONNRESET; return -1;}
    ...
}
```


Step 8: Initial handshake
-------------------------

Let's say we want to support mutliple versions of quux protocol. When a quux connection is established peers will exchange their version numbers and if those don't match, protocol initialization will fail.

假设我们希望支持多个版本的 quux 协议。当建立了 quux 连接时，对等点将交换它们的版本号，如果它们不匹配，则协议初始化将失败。

In fact, we don't even need proper handshake. Each peer can simply send its version number and wait for version number from the other party. We'll do this work in quux_attach() function.

事实上，我们甚至不需要适当的握手。每个对等方只需发送其版本号，然后等待对方的版本号。我们将在 quux_attach() 函数中完成这项工作。

Given that sending and receiving are blocking operations quux_attach() will become a blocking operation itself and will accept a deadline parameter:

鉴于发送和接收都是阻塞操作，quux_attach() 将成为阻塞操作本身，并将接受一个截止日期参数：

```
int quux_attach(int u, int64_t deadline) {
    ...
    const int8_t local_version = 1;
    int rc = bsend(u, &local_version, 1, deadline);
    if(rc < 0) {err = errno; goto error2;}
    uint8_t remote_version;
    rc = brecv(u, &remote_version, 1, deadline);
    if(rc < 0) {err = errno; goto error2;}
    if(remote_version != local_version) {err = EPROTO; goto error2;}
    ...
error2:
    free(self);
error1:
    hclose(u);
    errno = err;
    return -1;
}
```

Note how failure of initial handshake not only prevents initialization of quux socket, it also closes the underlying socket. This is necessary because otherwise the underlying sockets will be left in undefined state, with just half of quux handshake being done.

注意，初始握手的失败不仅阻止了 quux 套接字的初始化，还关闭了底层套接字。这是必要的，因为否则底层套接字将处于未定义状态，只完成了一半的 quux 握手。

Modify the test program accordingly (add deadlines), compile and test.

相应地修改测试程序(添加截止日期)，编译和测试。

Step 9: Terminal handshake
--------------------------

Imagine that user wants to close the quux protocol and start a new protocol, say HTTP, on top of the same underlying TCP connection. For that to work both peers would have to make sure that they've received all quux-related data before proceeding. If they had left any unconsumed data in TCP buffers, the subsequent HTTP protocol would read it and get confused by it.

假设用户希望关闭 quux 协议并在相同的底层 TCP 连接之上启动一个新协议，比如 HTTP。要想做到这一点，两个对等方都必须确保在继续之前，他们已经收到了所有与 quux 相关的数据。如果它们将任何未使用的数据留在 TCP 缓冲区中，则随后的 HTTP 协议将读取该数据并将其混淆。

To achieve that the peers will send a single termination byte (255) each to another to mark the end of the stream of quux messages. After doing so they will read any receive and drop all quux messages from the peer until they receive the 255 byte. At that point all the quux data are cleaned up, both peers have consistent view of the world and HTTP protocol can be safely initiated.

为了实现这一点，对等方将向另一个节点发送单个终止字节(255)，以标记 quux 消息流的结束。这样做之后，他们将读取任何接收，并从对等端删除所有 quux 消息，直到收到 255 字节为止。此时，所有的 quux 数据都被清理干净，两个对等点对世界都有一致的看法，HTTP 协议可以安全地启动。

Let's start with addinbg two flags to the quux socket object, one meaning "termination byte was already sent", the other "termination byte was already received":

让我们从向 quux 套接字对象添加两个标志开始，一个标志表示“终止字节已经发送”，另一个标志表示“终止字节已经接收”：

```
struct quux {
    ...
    int senddone;
    int recvdone;
};
```

The flags have to be initialized in the quux_attach() function:

必须在 quux_attach() 函数中初始化这些标志：

```
int quux_attach(int u, int64_t deadline) {
    ...
    self->senddone = 0;
    self->recvdone = 0;
    ...
}
```

If termination byte was already sent send function should return EPIPE error:

如果终止字节已经发送，发送函数应该返回 EPIPE 错误：

```
static int quux_msendl(struct msock_vfs *mvfs,
      struct iolist *first, struct iolist *last, int64_t deadline) {
    ...
    if(self->senddone) {errno = EPIPE; return -1;}
    ...

}
```

If termination byte was already received receive function should return EPIPE error. Also, we should handle the case when termination byte is received from the peer:

如果终止字节已经接收到，接收函数应该返回 EPIPE 错误。此外，当从对等端接收到终止字节时，我们应该处理这种情况：

```
static ssize_t quux_mrecvl(struct msock_vfs *mvfs,
      struct iolist *first, struct iolist *last, int64_t deadline) {
    ...
    if(self->recvdone) {errno = EPIPE; return -1;}
    ...
    if(sz == 255) {self->recvdone = 1; errno = EPIPE; return -1;}
    ...
}
```

Virtual function hdone() is supposed to start the terminal handshake. However, it is not supposed to wait till it is finished. The semantics of hdone() are "user is not going to send any more data". You can think of it as of EOF marker of a kind.

虚函数 hdone() 应该启动终止握手。然而，它不应该等到它完成。hdone() 的语义是“用户不会再发送任何数据”。你可以把它看作是一种 EOF 标记。

```
static int quux_hdone(struct hvfs *hvfs, int64_t deadline) {
    struct quux *self = (struct quux*)hvfs;
    if(self->senddone) {errno = EPIPE; return -1;}
    if(self->senderr) {errno = ECONNRESET; return -1;}
    uint8_t c = 255;
    int rc = bsend(self->u, &c, 1, deadline);
    if(rc < 0) {self->senderr = 1; return -1;}
    self->senddone = 1;
    return 0;
}
```

At this point we can modify quux_detach() function so that it properly cleans up any leftover protocol data.

此时，我们可以修改 quux_detach() 函数，以便正确地清理任何剩余的协议数据。

First, it will send the termination byte if it was not already sent. Then it will receive and drop messages until it receives the termination byte:

首先，如果尚未发送终止字节，则将发送终止字节。然后，它将接收和删除消息，直到接收到终止字节：

```
int quux_detach(int h, int64_t deadline) {
    int err;
    struct quux *self = hquery(h, quux_type);
    if(!self) return -1;
    if(self->senderr || self->recverr) {err = ECONNRESET; goto error;}
    if(!self->senddone) {
        int rc = quux_hdone(&self->hvfs, deadline);
        if(rc < 0) {err = errno; goto error;}
    }
    while(1) {
        ssize_t sz = quux_mrecvl(&self->mvfs, NULL, NULL, deadline);
        if(sz < 0 && errno == EPIPE) break;
        if(sz < 0) {err = errno; goto error;}
    }
    int u = self->u;
    free(self);
    return u;
error:
    quux_hclose(&self->hvfs);
    errno = err;
    return -1;
}
```

Note how the socket, including the underlying socket, is closed when the function fails.

注意，当函数失败时，套接字(包括底层套接字)是如何关闭的。

Adjust the test, compile and run. You are done with the tutorial. Have fun writing your own network protocols!

调整测试，编译和运行。你已经完成了本教程。写你自己的网络协议玩得开心！
