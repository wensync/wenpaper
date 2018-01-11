http://libdill.org/tutorial-basics.html


Tutorial: libdill basics
=========================

Introduction
------------

In this tutorial, you will develop a simple TCP "greet" server. Clients are meant to connect to it by telnet. After a client has connected, the server will ask for their name, reply with a greeting, and then proceed to close the connection.

在本教程中，您将开发一个简单的 TCP “greet”服务器。客户端应该通过 telnet 连接到它。在客户端连接后，服务器将询问它们的名称，以问候语回复，然后继续关闭连接。

An interaction with the server will look like this:

与服务器的交互如下所示：

```
$ telnet 127.0.0.1 5555
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
What's your name?
Bartholomaeus
Hello, Bartholomaeus!
Connection closed by foreign host.
```

Throughout the tutorial, you will learn how to use coroutines, channels, and sockets.

在本教程中，您将学习如何使用协程、通道和套接字。

Step 1: Setting up the stage
----------------------------

Start by including the libdill header file. Later we'll need some functionality from the standard library, so include those headers as well:

首先包括 libdill 头文件。稍后，我们将需要标准库中的一些功能，因此还包括这些标题：

```
#include <libdill.h>
#include <assert.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
```

Add a main function. We'll assume that the first argument, if present, will be the port number to be used by the server. If not specified, the port number will default to 5555:

添加一个主函数。我们假设第一个参数(如果存在)将是服务器使用的端口号。如果未指定端口号，端口号将默认为 5555：

```
int main(int argc, char *argv[]) {
    int port = 5555;
    if(argc > 1) port = atoi(argv[1]);

    return 0;
}
```

Now we can move on to the actual interesting stuff.

现在我们可以继续讨论真正有趣的事情了。

The tcp_listen() function creates a listening TCP socket. The socket can be used to accept new TCP connections from clients:

函数创建侦听 TCP 套接字。套接字可用于接受来自客户端的新 TCP 连接：

```
struct ipaddr addr;
int rc = ipaddr_local(&addr, NULL, port, 0);
if (rc < 0) {
    perror("Can't open listening socket");
    return 1;
}
int ls = tcp_listen(&addr, 10);
assert( ls >= 0 );
```

The ipaddr_local() function converts the textual representation of a local IP address to the actual address. Its second argument can be used to specify a local network interface to bind to. This is an advanced feature and you likely won't need it. For now, you can simply ignore it by setting it to NULL. This will cause the server to bind to all local network interfaces available.

函数将本地 IP 地址的文本表示转换为实际地址。它的第二个参数可用于指定要绑定到的本地网络接口。这是一个高级特性，您可能不需要它。现在，您只需将其设置为 NULL 就可以忽略它。这将导致服务器绑定到所有可用的本地网络接口。

The third argument is, unsurprisingly, the port that clients will connect to. When testing the program, keep in mind that valid port numbers range from 1 to 65535 and that binding to ports 1 through 1023 will typically require superuser privileges.

第三个参数是，毫不奇怪，客户端将连接到的端口。在测试程序时，请记住，有效的端口号从 1 到 65535 不等，绑定到端口 1 到 1023 通常需要超级用户权限。

If tcp_listen() fails, it will return -1 and set errno to the appropriate error code. The libdill API is in this respect very similar to standard POSIX APIs. Consequently, we can use standard POSIX error-handling mechanisms such as perror() in this case.

如果 tcp_listen() 失败，它将返回 -1 并将 errno 设置为适当的错误代码。在这方面，libdill API 非常类似于标准的 POSIX API。因此，在这种情况下，我们可以使用标准的 POSIX 错误处理机制，比如 perror()。

As for unlikely errors, the tutorial will simply use asserts to catch them so as to stay succinct and readable.

对于不太可能出现的错误，本教程将使用断言来捕获它们，以保持简洁和可读性。

If you run the program at this stage, you'll find out that it terminates immediately without pausing or waiting for a client to connect. That is what the tcp_accept() function is for:

如果您在此阶段运行该程序，您将发现它立即终止，而无需暂停或等待客户端连接。这就是 tcp_accept() 函数的用途：

```
int s = tcp_accept(ls, NULL, -1);
assert(s >= 0);
```

The function returns the newly established connection.

函数返回新建立的连接。

Its third argument is a deadline. We'll cover deadlines later on in this tutorial. For now, remember that the constant -1 can be used to mean 'no deadline' — if there is no incoming connection, the call will block forever.

它的第三个参数是最后期限。我们将在本教程后面讨论最后期限。现在，请记住，常数 -1 可以用来表示“没有截止日期” -- 如果没有传入连接，则呼叫将永远阻塞。

Finally, we want to handle any number of client connections instead of just one so we put the tcp_accept() call into an infinite loop. For now we'll just print a message when a new connection is established. We will close it immediately:

最后，我们希望处理任意数量的客户端连接，而不是仅仅处理一个连接，因此我们将 tcp_accept() 调用放入一个无限循环中。现在，我们将在建立新连接时打印一条消息。我们将立即关闭：

```
while(1) {
    int s = tcp_accept(ls, NULL, -1);
    assert(s >= 0);
    printf("New connection!\n");
    rc = hclose(s);
    assert(rc == 0);
}
```

The source code for this step can be found in the libdill repository as tutorial/step1.c. All the steps that follow can be found in the same directory.

这个步骤的源代码可以在 libdill 仓库中找到，如 tutorial/step1.c。下面的所有步骤都可以在同一个目录中找到。

Build the program like this:

像这样构建程序：

```
$ gcc -o greetserver step1.c -ldill
```

Then run the resulting executable:

然后运行生成的可执行文件：

```
$ ./greetserver
```

The server is now waiting for a new connection. Use telnet at a different terminal to establish one and then check whether the program works as expected:

服务器现在正在等待一个新的连接。在不同的终端上使用 telnet 建立一个终端，然后检查程序是否按预期工作：

```
$ telnet 127.0.0.1 5555
```

To test whether error handling works, try using an invalid port number:

若要测试错误处理是否有效，请尝试使用无效的端口号：

```
$ ./greetserver 70000
Can't open listening socket: Invalid argument
$
```

Everything seems to work as expected. Let's now move on to the step 2.

一切似乎都像预期的那样工作。现在我们来看第 2 步。

Step 2: The business logic
--------------------------

When a new connection arrives, the first thing that we want to do is to establish the network protocol we'll be using. libdill contains a small library of easily composable microprotocols that allows you to compose a wide range of protocols just by plugging different microprotocols into each other in a lego brick fashion. In this tutorial, however, we are going to limit ourselves to just a very simple setup. On top of the TCP connection that we've just created, we'll have a simple protocol that will split the TCP bytestream into discrete messages, using line breaks (CR+LF) as delimiters:

当一个新连接到达时，我们要做的第一件事就是建立我们将要使用的网络协议。libdill 包含一个易于组合的微协议的小型库，它允许您通过以乐高砖的方式将不同的微协议插入到彼此之间，就可以组成范围广泛的协议。然而，在本教程中，我们将把自己限制在一个非常简单的设置上。在我们刚刚创建的 TCP 连接之上，我们将有一个简单的协议，将 TCP 字节流拆分为离散消息，使用换行(CR+LF)作为分隔符：

```
int s = crlf_attach(s);
assert(s >= 0);
```

Note that hclose() works recursively. Given that our CRLF socket wraps an underlying TCP socket, a single call to hclose() will close both of them.

注意，hclose()是递归工作的。考虑到我们的 CRLF 套接字封装了一个底层 TCP 套接字，一个对 hclose() 的调用将关闭这两个套接字。

Once finished with the setup, we can send the "What's your name?" question to the client:

设置完成后，我们可以向客户发送“您的名字是什么？”问题：

```
rc = msend(s, "What's your name?", 17, -1);
if(rc != 0) goto cleanup;
```

Note that msend() works with messages, not bytes (the name stands for "message send"). Consequently, the data will act as a single unit: either all of it is received or none of it is. Also, we don't have to care about message delimiters. That's done for us by the CRLF protocol.

请注意，msend() 使用的是消息，而不是字节(名称代表“消息发送”)。因此，数据将作为一个单一的单位：要么全部收到，要么没有收到。此外，我们不必关心消息分隔符。这是 CRLF 协议为我们做的。

To handle possible errors from msend() (such as when the client has closed the connection), add a cleanup label before the hclose line and jump to it whenever you want to close the connection and proceed without crashing the server.

若要处理 msend() 中可能出现的错误(例如客户端关闭连接时)，请在 hclose 行之前添加一个清理标签，并在您想要关闭连接并在不使服务器崩溃的情况下跳转到它。

```
char inbuf[256];
ssize_t sz = mrecv(s, inbuf, sizeof(inbuf), -1);
if(sz < 0) goto cleanup;
```

The above piece of code simply reads the reply from the client. The reply is a single message, which in the case of the CRLF protocol translates to a single line of text. The mrecv function returns the number of bytes in the message.

上面的代码只是读取来自客户端的答复。答复是一条消息，在CRLF协议中，该消息转换为一行文本。函数返回消息中的字节数。

Having received a reply, we can now construct the greeting and send it to the client. The analysis of this code is left as an exercise to the reader:

收到回复后，我们现在可以构造问候语并将其发送给客户端。对这段代码的分析留给读者来练习：

```
inbuf[sz] = 0;
char outbuf[256];
rc = snprintf(outbuf, sizeof(outbuf), "Hello, %s!", inbuf);
rc = msend(s, outbuf, rc, -1);
if(rc != 0) goto cleanup;
```

Compile the program and check whether it works as expected.

编译程序并检查它是否按预期工作。

Step 3: Making it parallel
--------------------------

At this point, the client cannot crash the server, but it can block it. Do the following experiment:

此时，客户机不能使服务器崩溃，但它可以阻止它。做以下实验：

Start the server.
At a different terminal, start a telnet session and, without entering your name, let it hang.
At yet another terminal, open a new telnet session.
Observe that the second session hangs without even asking you for your name.
The reason for this behavior is that the program doesn't even start accepting new connections until the entire dialog with the client has finished. What we want instead is to run any number of dialogs with clients in parallel. And that is where coroutines kick in.

启动服务器。
在另一个终端，启动 telnet 会话，不输入您的名字，让它挂起。
在另一个终端，打开一个新的 telnet 会话。
请注意，第二次会议挂起，甚至没有问你的名字。
造成这种行为的原因是，在与客户端的整个对话框完成之前，程序甚至不会开始接受新的连接。相反，我们希望与客户端并行运行任意数量的对话框。这就是协同效应的开始。

Coroutines are defined using the coroutine keyword and launched with the go() construct.

协程使用 coroutine 关键字定义，并使用 go() 构造启动。

In our case, we can move the code performing the dialog with the client into a separate function and launch it as a coroutine:

在我们的例子中，我们可以将执行与客户端对话的代码移动到一个单独的函数中，并作为一个协程启动它：

```
coroutine void dialog(int s) {
    int rc = msend(s, "What's your name?", 17, -1);

    ...

cleanup:
    rc = hclose(s);
    assert(rc == 0);
}

int main(int argc, char *argv[]) {

    ...

    while(1) {
        int s = tcp_accept(ls, NULL, -1);
        assert(s >= 0);
        s = crlf_attach(s);
        assert(s >= 0);
        int cr = go(dialog(s));
        assert(cr >= 0);
    }
}
```

Let's compile it and try the initial experiment once again. As can be seen, one client now cannot block another one. Excellent. Let's move on.

让我们编译它，再试一次最初的实验。可以看到，一个客户现在无法阻止另一个客户端。太棒了。让我们继续前进。

NOTE: Please note that for the sake of simplicity the program above doesn't track the running coroutines and close them when they are finished. This results in memory leaks. To understand how should this kind of thing be done properly check the article about structured concurrency.

注意：请注意，为了简单起见，上面的程序不跟踪运行的协同线，并在它们完成时关闭它们。这会导致内存泄漏。要理解如何正确地完成这类事情，请查看关于结构化并发的文章。

Step 4: Deadlines
-----------------

File descriptors can be a scarce resource. If a client connects to the greetserver and lets the dialog hang without entering a name, one file descriptor on the server side is, for all intents and purposes, wasted.

文件描述符可能是一种稀缺资源。如果客户端连接到 greetserver 并让对话框挂起而不输入名称，那么服务器端的一个文件描述符就会被浪费掉。

To deal with the problem, we are going to timeout the whole client/server dialog. If it takes more than 10 seconds, the server will kill the connection at once.

为了解决这个问题，我们将超时整个客户机/服务器对话框。如果超过 10 秒，服务器将立即关闭连接。

One thing to note is that libdill uses deadlines rather than the more conventional timeouts. In other words, you specify the time instant by which you want the operation to finish rather than the maximum time it should take to run it. To construct deadlines easily, libdill provides the now() function. The deadline is expressed in milliseconds, which means you can create a deadline 10 seconds in the future as follows:

值得注意的是，libdill 使用的是截止日期，而不是更为传统的超时。换句话说，您指定希望操作完成的时间瞬间，而不是运行操作所需的最大时间。为了方便地构造最后期限，libdill 提供了 now() 函数。最后期限以毫秒为单位表示，这意味着您可以在将来创建一个截止日期 10秒，如下所示：

```
int64_t deadline = now() + 10000;
```

Furthermore, you have to modify all potentially blocking function calls in the program to take the deadline parameter. In our case:

此外，您还必须修改程序中所有可能阻塞的函数调用，以接受截止日期参数。就我们而言：

```
int64_t deadline = now() + 10000;
int rc = msend(s, "What's your name?", 17, deadline);
if(rc != 0) goto cleanup;
char inbuf[256];
ssize_t sz = mrecv(s, inbuf, sizeof(inbuf), deadline);
if(sz < 0) goto cleanup;
```

Note that errno is set to ETIMEDOUT if the deadline is reached. Since we're treating all errors the same (by closing the connection), we don't make any specific provisions for deadline expiries.

注意，如果到达截止日期，errno 将设置为 ETIMEDOUT。由于我们对所有错误的处理都是一样的(通过关闭连接)，所以我们不对截止日期规定做出任何具体规定。

Step 5: Communication among coroutines
--------------------------------------

Suppose we want the greetserver to keep statistics: The overall number of connections made, the number of those that are active at the moment and the number of those that have failed.

假设我们希望 greetserver 保存统计信息：连接的总数、当前活动的连接数和已失败连接的数量。

In a classic, thread-based application, we would keep the statistics in global variables and synchronize access to them using mutexes.

在一个经典的、基于线程的应用程序中，我们将把统计信息保存在全局变量中，并使用互斥变量同步访问它们。

With libdill, however, we aim at "concurrency by message passing", and so we are going to implement the feature in a different way.

然而，使用libdill，我们的目标是“通过消息传递实现并发性”，因此我们将以不同的方式实现该特性。

We will create a new coroutine that will keep track of the statistics and a channel that will be used by the dialog() coroutines to communicate with it:

我们将创建一个跟踪统计数据的新协程，并创建一个用于 dialog() 协程与其通信的通道：

[image]

First, we define the values that will be passed through the channel:

首先，我们定义将通过通道传递的值：

```
#define CONN_ESTABLISHED 1
#define CONN_SUCCEEDED 2
#define CONN_FAILED 3
```

Now we can create the channel and pass it to the dialog() coroutines:

现在，我们可以创建这个通道并将它传递给 dialog() 协程：

```
coroutine void dialog(int s, int ch) {
    ...
}

int main(int argc, char *argv[]) {

    ...

    int ch = chmake(sizeof(int));
    assert(ch >= 0);

    while(1) {
        int s = tcp_accept(ls, NULL, -1);
        assert(s >= 0);
        s = crlf_attach(s);
        assert(s >= 0);
        int cr = go(dialog(s, ch));
        assert(cr >= 0);
    }
}
```

The argument to chmake() is the type of values that will be passed through the channel. In our case, the type is simply an integer.

chmake()的参数是将通过通道传递的值的类型。在我们的例子中，类型只是一个整数。

Libdill channels are "unbuffered". In other words, the sending coroutine will block each time until the receiving coroutine can process the message.

Libdill 通道“没有缓冲”。换句话说，每次发送协程都会阻塞，直到接收协程处理消息为止。

This kind of behavior could, in theory, become a bottleneck, however, in our case we assume that the statistics() coroutine will be extremely fast and not turn into one.

从理论上讲，这种行为可能成为瓶颈，然而，在我们的例子中，我们假设 statistics() 协程将非常快，不会变成一个瓶颈。

At this point we can implement the statistics() coroutine, which will run forever in a busy loop and collect statistics from all the dialog() coroutines. Each time the statistics change, it will print them to stdout:

此时，我们可以实现 statistics() 协程，它将在一个繁忙的循环中永远运行，并从所有的 dialog() 协程中收集统计信息。每次统计数据发生变化时，它都会将它们打印到 stdout：

```
coroutine void statistics(int ch) {
    int active = 0;
    int succeeded = 0;
    int failed = 0;
    
    while(1) {
        int op;
        int rc = chrecv(ch, &op, sizeof(op), -1);
        assert(rc == 0);

        switch(op) {
        case CONN_ESTABLISHED:
            ++active;
            break;
        case CONN_SUCCEEDED:
            --active;
            ++succeeded;
            break;
        case CONN_FAILED:
            --active;
            ++failed;
            break;
        }

        printf("active: %-5d  succeeded: %-5d  failed: %-5d\n",
            active, succeeded, failed);
    }
}

int main(int argc, char *argv[]) {

    ...

    int ch = chmake(sizeof(int));
    assert(ch >= 0);
    int cr = go(statistics(ch));
    assert(cr >= 0);

    ...

}
```

The chrecv() function will retrieve one message from the channel or block if there is none available. At the moment we are not sending anything to it, so the coroutine will simply block forever.

如果没有可用的消息，chrecv() 函数将从通道或块中检索一条消息。目前我们并没有向它发送任何东西，所以协程将永远阻塞。

To fix that, let's modify the dialog() coroutine to send some messages to the channel. The chsend() function will be used to do that:

为了解决这个问题，让我们修改 dialog() 协程，以便向通道发送一些消息。将使用 chsend() 函数来执行此操作：

```
coroutine void dialog(int s, int ch) {
    int op = CONN_ESTABLISHED;
    int rc = chsend(ch, &op, sizeof(op), -1);
    assert(rc == 0);

    ...

cleanup:
    op = errno == 0 ? CONN_SUCCEEDED : CONN_FAILED;
    rc = chsend(ch, &op, sizeof(op), -1);
    assert(rc == 0);
    rc = hclose(s);
    assert(rc == 0);
}
```

Now recompile the server and run it. Create a telnet session and let it time out. The output on the server side will look like this:

现在重新编译服务器并运行它。创建一个 telnet 会话并让它超时。服务器端的输出如下所示：

```
$ ./greetserver
active: 1      succeeded: 0      failed: 0
active: 0      succeeded: 0      failed: 1
```

The first line is displayed when the connection is established: There is one active connection and no connection has succeeded or failed yet.

建立连接时将显示第一行：有一个活动连接，但尚未成功或失败。

The second line shows up when the connection times out: There are no active connection anymore and one connection has failed so far.

当连接超时时，第二行出现：不再有活动连接，到目前为止，有一个连接已失败。

Now try pressing enter in telnet when asked for your name. The connection will be terminated by the server immediately, without sending the greeting, and the server log will report one failed connection. What's going on here?

现在，当询问您的姓名时，尝试在 telnet 中按 enter。服务器将立即终止连接，而不发送问候语，服务器日志将报告一个连接失败。这里发生了什么事？

The reason for the behavior is that the CRLF protocol treats an empty line as a connection termination request. Thus, when you press enter in telnet, you send an empty line which causes mrecv() on the server side to return the EPIPE error, which represents "connection terminated by peer". The server will jump directly into to the cleanup code.

这种行为的原因是 CRLF 协议将空行视为连接终止请求。因此，当您在 telnet 中按 enter 键时，就会发送一条空行，这将导致服务器端的mrecv() 返回 EPIPE 错误，该错误表示“由对等节点终止的连接”。服务器将直接跳入清理代码。

And that's the end of the tutorial. Enjoy your time with the library!

本教程到此结束。好好享受这个库的时光吧！
