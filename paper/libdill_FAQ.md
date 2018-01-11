http://libdill.org/documentation.html

FAQ: How can I report a bug?
----------------------------
Report a bug here:

https://github.com/sustrik/libdill/issues

FAQ: What's structured concurrency?
-----------------------------------
Check this short introductory article about structured concurrency.

FAQ: What's libdill's performance?
----------------------------------
The best way to find out is to check for yourself. You will find some small performance benchmarks in the perf subdirectory.

最好的办法是自己检查一下。您将在 perf 子目录中找到一些小的性能基准。

Generally speaking, though, libdill's concurrency primitives are only a bit slower than basic C flow control statements. A context switch has been seen to execute in as little as 6 ns, with coroutine creation taking 26 ns. Passing a message through a channel takes about 40 ns.

不过，一般来说，libdill 的并发原语只比基本的 C 流控制语句慢一点。上下文切换被认为在 6ns 的范围内执行，协程创建则需要 26 ns。通过一个通道传递消息大约需要 40 ns。

FAQ: How does libdill's concurrency differ from Go's concurrency?
-----------------------------------------------------------------
1. No interaction between threads. Each thread is treated as a separate process.
2. Channels are always unbuffered.
3. choose, unlike select, is deterministic. If multiple clauses can be executed, the clause closest to the beginning of the pollset wins.
4. chdone signals the closing of a channel to both senders and receivers.
5. Coroutines can be canceled.

1. 线程之间没有交互。每个线程被视为一个单独的进程。
2. 信道总是不缓冲的。
3. 与 select 不同，choose 是确定性的。如果可以执行多个子句，则最接近 pollset 开头的子句将获胜。
4. chdone 向发送者和接收者发出关闭通道的信号。
5. 协程可以被取消。

FAQ: How does libdill differ from libmill?
------------------------------------------
libmill was a project that aimed to copy Go's concurrency model to C 1:1 without introducing any innovations or experiments. The project is finished now. It will be maintained but won't change in the future.

libmill 是一个致力于将 Go 的并发模型 1:1 复制到 C 的项目，而不引入任何创新或实验。这个项目现在已经完成了。它将被维持，但在未来不会改变。

libdill is a follow-up project that experiments with structured concurrency and diverges from the Go model.

libdill 是一个后续的项目，它用结构化的并发性进行实验，并偏离 Go 模型。

Technically, these are the differences:

从技术上讲，这些是不同之处：

1. Libdill is idiomatic C. Whereas libmill takes Go's concurrency API and implements it in an almost identical manner in C, libdill tries to provide the same functionality via a more C-like and POSIX-like API. For example, choose is a function in libdill rather than a language construct, or Go's panic is replaced with error returns.
2. Coroutines can be canceled. This creates a foundation for "structured concurrency".
3. chdone causes blocked recv on the channel to return the EPIPE error rather than a value.
4. chdone will signal both the senders and the receivers of a channel. This allows for scenarios such as multiple senders and a single receiver communicating via a single channel. The receiver can use chdone to let the senders know that it's terminating.
5. libmill's fdwait was replaced by fdin and fdout. The idea is that if we want data to flow through the connection in both directions in parallel, we should use two coroutines rather than one.

1. libdill 是惯用的 C，而 libmill 采用 Go 的并发 API 并以几乎相同的方式在 C 中实现它，libdill 则试图通过一个更类似 C 和POSIX 的 API 提供相同的功能。例如，select 是 libdill 中的一个函数，而不是一个语言结构，或者 Go 的 panic 被错误返回所代替。
2. 协程是可取消的。这为“结构化并发”奠定了基础。
3. chdone 导致通道上阻塞的 recv 返回 EPIPE 错误，而不是一个值。
4. chdone 将向信道的发送者和接收者发送信号。这允许使用多个发送者和单个接收器等场景通过单个信道进行通信。接收者可以使用chdone 让发送者知道它正在终止。
5. 用 fdin 和 fdout 代替 libmil 的 fdwait。如果我们希望数据在两个方向上并行地流动，我们应该使用两个协程，而不是一个。

FAQ: How can I access the source code?
--------------------------------------
To clone the repository, run:

```
$ git clone https://github.com/sustrik/libdill.git
```

To build from the source (you'll need automake and libtool), run:

```
$ ./autogen.sh
$ ./configure
$ make
$ make check
$ sudo make install
```

For easy debugging, use the following configure options:

```
$ ./configure --disable-shared --enable-debug --enable-valgrind
```

The above will turn optimization off, generate debug symbols, and link all the tests with the static version of the library. The second option will cause executables in the tests subdirectory to be actual debuggable binaries rather that wrapper shell scripts. The last option instructs valgrind about where the coroutine stacks are located, thereby preventing valgrind from generating spurious warnings.

FAQ: Is continuous integration available for libdill?
-----------------------------------------------------
Travis: https://travis-ci.org/sustrik/libdill

FAQ: How can I contribute?
--------------------------
To contribute to libdill, create a GitHub pull request. You have to state that your patch is submitted under the MIT/X11 license, so that it can be incorporated into the mainline codebase without licensing issues.

If you make a substantial contribution to a file, add your copyright to the file header. Irrespective of whether you do so or not, your name will be added to the AUTHORS file to indicate you own copyright to part of the codebase.

FAQ: How can I see coverage report for the tests?
-------------------------------------------------

```
$ ./configure --enable-gcov
$ make clean
$ make check
$ lcov -t "libdill" -o libdill.info -c -d .
$ genhtml -o lcov libdill.info
```

After doing the steps above open lcov/index.html in your browser.

FAQ: What is the release process?
---------------------------------
These instructions are intended for project maintainers:

- Run make distcheck to check whether the packaging process still works.
- Bump the ABI version appropriately (see here: http://250bpm.com/blog:41).
- Commit and push your commits back to the master branch on GitHub.
- Tag the new version and push the tag to GitHub (e.g. git tag -a 0.3-beta; git push origin 0.3-beta).
- Clone a clean repo from GitHub.
- Build the package (./autogen.sh; ./configure; make distcheck).
- Add the package to the gh-pages branch.
- Adjust download.md in gh-pages branch.
- Regenerate the documentation using ./gendocs.sh <version-number>.
- Commit and push to gh-pages.
- Announce the release on twitter, etc.