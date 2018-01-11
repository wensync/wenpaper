http://libdill.org/structured-concurrency.html


libdill: Structured Concurrency for C
=====================================

What is concurrency?
--------------------
Concurrency allows multiple functions to run independent of one another.

并发性允许多个函数彼此独立运行。

How is concurrency implemented in libdill?
------------------------------------------
Functions that are meant to run concurrently must be annotated with the coroutine modifier.

意味着并发运行的函数必须使用 coroutine 修饰符进行注解。

```C
coroutine void foo(int arg1, const char *arg2);
```

To launch a coroutine, use the go keyword:

要启动一个协程，请使用 go 关键字：

```C
go(foo(34, "ABC"));
```

Launching concurrent functions — or coroutines, in libdill terminology — using the go construct and switching between them is extremely fast. It requires only a few machine instructions. This makes coroutines a suitable basic flow control mechanism, not unlike the if or while keywords, which have comparable performance.

使用 go 构造并在它们之间切换，启动并发函数--或者用 libdill 术语来说--是非常快的。它只需要几个机器指令。这使得协程成为一种合适的基本流控制机制，与具有类似性能的 if 或 while 关键字相似。

Coroutines have one big limitation, though: All coroutines run on a single CPU core. If you want to take advantage of multiple cores, you have to launch multiple threads or processes, presumably as many of them as there are CPU cores on your machine.

但是，协程有一个很大的限制：所有协程都运行在一个 CPU 核心上。如果您想要利用多个核，您必须启动多个线程或进程，大概与机器上有 CPU 核的数量一样多。

Coroutines are scheduled cooperatively. What that means is that a coroutine has to explicitly yield control of the CPU to allow a different coroutine to run. In a typical scenario, this is done transparently to the user: When a coroutine invokes a function that would block (such as msleep orchrecv), the CPU is automatically yielded. However, if a coroutine runs without calling any blocking functions, it may hold the CPU forever. For these cases, the yield function can be used to manually relinquish the CPU to other coroutines manually.

协程是协作式调度的。这意味着一个协程必须显式地让出 CPU 的控制，以允许一个不同的协程运行。在一个典型的场景中，这是对用户透明地完成的：当一个协程调用一个会阻塞的函数(比如 mleep 或 chrecv)时，CPU 就会自动让出。但是，如果一个协程运行时没有调用任何阻塞函数，它可能永远占用 CPU。对于这些情况，可以使用 yield 函数手动将 CPU 交给其他协程。

What is structured concurrency?
-------------------------------
Structured concurrency means that lifetimes of concurrent functions are cleanly nested. If coroutine foo launches coroutine bar, then bar must finish before foo finishes.

结构化并发意味着并发函数的生存期是干净嵌套的。如果协程 foo 启动协程 bar，那么 bar 必须在 foo 完成之前完成。

This is not structured concurrency:

这不是结构化并发：

[image]

This is structured concurrency:

这是结构化并发：

[image]

The goal of structured concurrency is to guarantee encapsulation. If the main function calls foo, which in turn launches bar in a concurrent fashion, main will be guaranteed that once foo has finished, there will be no leftover functions still running in the background.

结构化并发的目标是保证封装。如果主函数调用 foo，而 foo 又以并发方式启动 bar，则会保证 foo 完成后，后台将不再有剩余的函数还在运行。

What you end up with is a tree of coroutines rooted in the main function. This tree spreads out towards the smallest worker functions, and you may think of this as a generalization of the call stack — a call tree, if you will. In it, you can walk from any particular function towards the root until you reach the main function:

你最终得到的是一棵以 main 函数为根的协程树。这棵树扩展到最小的工作函数，如果你愿意的话，你可以把它看作是调用栈的概括 -- 一个调用树。在它中，您可以从任何特定的函数一直走到根，直到到达 main 函数为止：

[image]

How is structured concurrency implemented in libdill?
-----------------------------------------------------
As with all idiomatic C, you have to do it by hand.

就像所有的惯用 C 一样，你必须手工来完成它。

The good news is that it's easy.

好消息是这很容易。

The go construct returns a handle. The handle can be closed, and thereby kill the associated concurrent function.

go 构造返回一个句柄。句柄可以关闭，从而关闭相关的并发函数。

```C
int h = go(foo());
do_work();
hclose(h);
```

What happens to a function that gets killed? It may have some resources allocated, and we want it to finish cleanly, without leaking those resources.

被杀的函数会怎么样？它可能分配了一些资源，我们希望它干净地完成，而不泄漏这些资源。

The mechanism is simple. In a function being killed by hclose, all blocking calls will start failing with the ECANCELED error. On one hand, this forces the function to finish quickly (there's not much you can do without blocking functions); on the other hand, it provides an opportunity for cleanup.

机制很简单。在被 hclose 杀死的函数中，所有阻塞调用都将开始失败，因为 ECANCELED 错误。一方面，这迫使该函数快速完成(没有阻塞功能，您无法做什么)；另一方面，它提供了清理的机会。

```C
coroutine void foo(void) {
    void *resource = malloc(1000);
    while(1) {
        int rc = msleep(now() + 100);
        if(rc == -1 && errno == ECANCELED) {
            free(resource);
            return;
        }
    }
}
```

What about asynchronous objects?
--------------------------------
Sometimes, instead of launching a coroutine, you may want to create an object that runs coroutines in the background.

有时，您可能希望创建一个在后台运行协程的对象，而不是启动协程。

For example, an object called tcp_connection may run two coroutines, one for asynchronously reading data from and one for asynchronously writing data to the network.

例如，一个名为 tcp_connection 的对象可以运行两个协程，一个用于异步从网络读取数据，另一个用于异步地将数据写入网络。

Still, it would be nice if the object was a node in the calltree, just like a coroutine is.

尽管如此，如果对象是调用树中的一个节点，就像协程一样，那就更好了。

In other words, you may want a guarantee that once the object is deallocated there will be no leftover coroutines running:

换句话说，您可能需要一个保证，即一旦对象被解除分配，就不会有剩余的协程运行：

[image]

And there's no trick there. Just do it in the most straightforward way. Launch the coroutines in the function that opens the object and close them in the function the closes the object. When the main function closes the connection object, both the sender and receiver coroutine will be stopped automatically.

也没有什么诡计。用最直截了当的方式来做。在打开对象的函数中启动协程，并在函数中关闭它们，关闭对象。当 main 函数关闭连接对象时，发送方和接收方的协同将自动停止。

```C
struct tcp_connection {
    int sender;
    int receiver;
}

void tcp_connection_open(struct tcp_connection *self) {
    self->sender = go(tcp_sender(self));
    self->receiver = go(tcp_receiver(self));
}

void tcp_connection_close(struct tcp_connection *self) {
    hclose(self->sender);
    hclose(self->receiver);
}
```