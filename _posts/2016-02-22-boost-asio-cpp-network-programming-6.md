---
layout: post
title: "Boost.Asio－进阶话题"
description: "Boost.Asio C++ Network Programming 翻译"

category: Boost.Asio
tags: [Boost.Asio,翻译]

imagefeature: mmoaay_bg.jpg
comments: true
share: true
---

这一章对Boost.Asio的一些进阶话题进行了阐述。在日常编程中深入研究这些问题是不太可能的，但是知道这些肯定是有好处的：

* 如果调试失败，你需要看Boost.Asio能帮到你什么
* 如果你需要处理SSL，看Boost.Asio能帮你多少
* 如果你指定一个操作系统，看Boost.Asio为你准备了哪些额外的特性
* 
### Asio VS Boost.Asio

Boost.Asio的作者也保持了Asio。你可以用Asio的方式来思考，因为它在两种情况中都有：Asio（非Boost的）和Boost.Asio。作者声明过更新都会先在非Boost中出现，然后过段时间后，再加入到Boost的发布中。

不同点被归纳到下面几条：

* Asio被定义在*asio::*的命名空间中，而Boost.Asio被定义在*boost::asio::*中
* Asio的主头文件是*asio.hpp*，而Boost.Asio的头文件是*boost/asio.hpp*
* Asio也有一个启动线程的类（和*boost::thread*一样）
* Asio提供它自己的错误码类(*asio::error_code*代替*boost::system::error_code*，然后*asio:system_error*代替*boost::systrem::system_error*)

你可以在这里查阅更多Asio的信息：[http://think_async.com](http://think_async.com)

你需要自己决定你选择的版本，我选择Boost.Asio。下面是一些当你做选择时需要考虑的问题：

* Asio的新版本比Boost.Asio的新版本发布要早（因为Boost的版本更新比较少）
* Asio只有头文件（而Boost.Asio的部分依赖于其他Boost库，这些库可能需要编译）
* Asio和Boost.Asio都是非常成熟的，所以除非你非常需要一些Asio新发布的特性，Boost.Asio是非常保险的选择，而且你也可以同时拥有其他Boost库的资源

尽管我不推荐这样，你可以在一个应用中同时使用Asio和Boost.Asio。在允许的情况下这是很自然的，比如你使用Asio，然后一些第三方库是Boost.Asio，反之亦然。 

### 调试

调试同步应用往往比调试异步应用要简单。对于同步应用，如果阻塞了，你会跳转进入调试，然后你会知道你在哪（同步意味着有序的）。然而如果是异步，事件不是有序发生的，所以在调试中是非常难知道到底发生了什么的。

为了避免这种情况，首先，你需要深入了解协程。如果实现正确，基本上你一点也不会碰到异步调试的问题。

以防万一，在做异步编码的时候，Boost.Asio还是对你伸出了援手；Boost.Asio允许“句柄追踪”，当*BOOST_ASIO_ENABLE_HANDLER_TRACKING*被定义时，Boost.Asio会写很多辅助的输出到标准错误流，纪录时间，异步操作，以及操作和完成处理handler的关系。

#### 句柄追踪信息

虽然输出信息不是那么容易理解，但是有总比没有好。Boost.Asio的输出是*@asio|<timestamp>|<action>|<description>* 
。
第一个标签永远都是*@asio*，因为其他代码也会输出到标准错误流（和*std::error*相当），所以你可以非常简单的用这个标签过滤从Boost.Asio打印出来的信息。*timestamp*实例从1970年1月1号到现在的秒数和毫秒数。*action*实例可以是下面任何一种：

* *\>n*：这个在我们进入handler *n*的时候使用。*description*实例包含了我们发送给handler的参数。
* *<n*：这个在我们退出handler *n*的时候使用。
* *!n*：这个当我们因为异常退出handler *n*的时候使用。
* *-n*：这个当handler *n*在没有调用的情况就退出的时候使用；可能是因为io_service实例被删除地太快了（在*n*有机会被调用之前）
* *n*m*：这个当handler *n*创建了一个新的有完成处理hanlder * *m*的异步操作时被调用。*description*实例展示的就是异步操作开始的地方。当你看到*>m*（开始）和*<m*（结束）时*completion*句柄被调用了。
* *n*：就像在*description*中展示的一样，这个当handler *n*做了一个操作的时候使用（可能是*close*或者*cancel*操作）。你一般可以忽略这些信息。

当*n*是0时，操作是在所有（异步）handler之外被执行的；你经常会在第一个操作时看到这个，或者当你使用的信号量其中一个被触发时。

你需要特别注意类型为*!n*和*-n*的信息，这些信息大部分都意味着你的代码有错误。在第一种情形中，异步方法没有抛出异常，所以，异常一定是你自己造成的；你不能让异常跑出你的*completion*句柄。第二种情形中，你可能太早就销毁了*io_service*实例，在所有完成处理句被调用之前。

#### 一个例子

为了向你展示一个带辅助输出信息的例子，我们修改了在**第六章 Boost.Asio其他特性** 中使用的例子。你所需要做的仅仅是在包含*boost/asio.hpp*之前添加一个*#define*

```
#define BOOST_ASIO_ENABLE_HANDLER_TRACKING
#include <boost/asio.hpp>
...
```

同时，我们也在用户登录和接收到第一个客户端列表时将信息输出到控制台中。输出会如下所示：

```
@asio|1355603116.602867|0*1|socket@008D4EF8.async_connect
@asio|1355603116.604867|>1|ec=system:0
@asio|1355603116.604867|1*2|socket@008D4EF8.async_send
@asio|1355603116.604867|<1|
@asio|1355603116.604867|>2|ec=system:0,bytes_transferred=11
@asio|1355603116.604867|2*3|socket@008D4EF8.async_receive
@asio|1355603116.604867|<2|
@asio|1355603116.605867|>3|ec=system:0,bytes_transferred=9
@asio|1355603116.605867|3*4|io_service@008D4BC8.post
@asio|1355603116.605867|<3|
@asio|1355603116.605867|>4|
John logged in
@asio|1355603116.606867|4*5|io_service@008D4BC8.post
@asio|1355603116.606867|<4|
@asio|1355603116.606867|>5|
@asio|1355603116.606867|5*6|socket@008D4EF8.async_send
@asio|1355603116.606867|<5|
@asio|1355603116.606867|>6|ec=system:0,bytes_transferred=12
@asio|1355603116.606867|6*7|socket@008D4EF8.async_receive
@asio|1355603116.606867|<6|
@asio|1355603116.606867|>7|ec=system:0,bytes_transferred=14
@asio|1355603116.606867|7*8|io_service@008D4BC8.post
@asio|1355603116.607867|<7|
@asio|1355603116.607867|>8|
John, new client list: John
```

让我们一行一行分析：

* 我们进入*async_connect*，它创建了句柄1（在这个例子中，所有的句柄都是*talk_to_svr::step*）
* 句柄1被调用（当成功连接到服务端时）
* 句柄1调用*async_send*，这创建了句柄2（这里，我们发送登录信息到服务端）
* 句柄1退出
* 句柄2被调用，11个字节被发送出去（login John）
* 句柄2调用*async_receive*，这创建了句柄3（我们等待服务端返回登录的结果）
* 句柄2退出
* 句柄3被调用，我们收到了9个字节（login ok）
* 句柄3调用*on_answer_from_server*（这创建了句柄4）
* 句柄3退出
* 句柄4被调用，这会输出John logged in
* 句柄4调用了另外一个step（句柄5），这会写入*ask_clients*
* 句柄4退出
* 句柄5进入
* 句柄5，*async_send_ask_clients*，创建句柄6
* 句柄5退出
* 句柄6调用*async_receive*，这创建了句柄7（我们等待服务端发送给我们已存在的客户端列表）
* 句柄6退出
* 句柄7被调用，我们接受到了客户端列表
* 句柄7调用*on_answer_from_server*（这创建了句柄8）
* 句柄7退出
* 句柄8进去，然后输出客户端列表（*on_clients*）

这需要时间去理解，但是一旦你理解了，你就可以分辨出有问题的输出，从而找出需要被修复的那段代码。

#### 句柄追踪信息输出到文件

默认情况下，句柄的追踪信息被输出到标准错误流（相当于*std::cerr*）。而把输出重定向到其他地方的可能性是非常高的。对于控制台应用，输出和错误输出都被默认输出到相同的地方，也就是控制台。但是对于一个windows（非命令行）应用来说，默认的错误流是null。

你可以通过命令行把错误输出重定向，比如：

```
some_application 2>err.txt
```

或者，如果你不是很懒，你可以代码实现，就像下面的代码片段

```
//  对于Windows
HANDLE h = CreateFile("err.txt", GENERIC_WRITE, 0, 0, CREATE_ALWAYS,
FILE_ATTRIBUTE_NORMAL , 0);
SetStdHandle(STD_ERROR_HANDLE, h);
// 对于Unix
int err_file = open("err.txt", O_WRONLY);
dup2(err_file, STDERR_FILENO);
```

### SSL

Boost.Asio提供了一些支持基本SSL的类。它在幕后使用的其实是OpenSSL，所以，如果你想使用SSL，首先从[www.openssl.org](www.openssl.org)下载OpenSSL然后构建它。你需要注意，构建OpenSSL通常来说不是一个简单的任务，尤其是你没有一个常用的编译器，比如Visual Studio。

假如你成功构建了OpenSSL，Boost.Asio就会有一些围绕它的封装类：

* *ssl::stream*：它代替*ip:<protocol>::socket*来告诉你用什么
* *ssl::context*：这是给第一次握手用的上下文
* *ssl::rfc2818_verification*：使用这个类可以根据RFC 2818协议非常简单地通过证书认证一个主机名

首先，你创建和初始化SSL上下文，然后使用这个上下文打开一个连接到指定远程主机的socket，然后做SSL握手。握手一结束，你就可以使用Boost.Asio的*read*/write**等自由函数。

下面是一个连接到Yahoo！的HTTPS客户端例子：

```
#include <boost/asio.hpp>
#include <boost/asio/ssl.hpp>
using namespace boost::asio;
io_service service;
int main(int argc, char* argv[]) {
    typedef ssl::stream<ip::tcp::socket> ssl_socket;
    ssl::context ctx(ssl::context::sslv23);
    ctx.set_default_verify_paths();
    // 打开一个到指定主机的SSL socket
    io_service service;
    ssl_socket sock(service, ctx);
    ip::tcp::resolver resolver(service);
    std::string host = "www.yahoo.com";
    ip::tcp::resolver::query query(host, "https");
    connect(sock.lowest_layer(), resolver.resolve(query));
    // SSL 握手
    sock.set_verify_mode(ssl::verify_none);
    sock.set_verify_callback(ssl::rfc2818_verification(host));
    sock.handshake(ssl_socket::client);
    std::string req = "GET /index.html HTTP/1.0\r\nHost: " + host + "\r\nAccept: */*\r\nConnection: close\r\n\r\n";
    write(sock, buffer(req.c_str(), req.length()));
    char buff[512];
    boost::system::error_code ec;
    while ( !ec) {
        int bytes = read(sock, buffer(buff), ec);
        std::cout << std::string(buff, bytes);
    }
} 
```

第一行能很好的自释。当你连接到远程主机，你使用*sock.lowest_layer()*，也就是说，你使用底层的socket（因为*ssl::stream*仅仅是一个封装）。接下来三行进行了握手。握手一结束，你使用Booat.Asio的*write()*方法做了一个HTTP请求，然后读取（*read()*）所有接收到的字节。

当实现SSL服务端的时候，事情会变的有点复杂。Boost.Asio有一个SSL服务端的例子，你可以在*boost/libs/asio/example/ssl/server.cpp*中找到。

### Boost.Asio的Windows特性

接下来的特性只赋予Windows操作系统

#### 流处理

Boost.Asio允许你在一个Windows句柄上创建封装，这样你就可以使用大部分的自由函数，比如*read()，read_until()，write()，async_read()，async_read_until()*和*async_write()*。下面告诉你如何从一个文件读取一行：

```
HANDLE file = ::CreateFile("readme.txt", GENERIC_READ, 0, 0, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL | FILE_FLAG_OVERLAPPED, 0);
windows::stream_handle h(service, file);
streambuf buf;
int bytes = read_until(h, buf, '\n');
std::istream in(&buf);
std::string line;
std::getline(in, line);
std::cout << line << std::endl;
```

*stream_handle*类只有在I/O完成处理端口正在被使用的情况下才有效（这是默认情况）。如果情况满足，*BOOST_ASIO_HAS_WINDOWS_STREAM_HANDLE*就被定义

#### 随机访问句柄

Boost.Asio允许对一个指向普通文件的句柄进行随机读取和写入。同样，你为这个句柄创建一个封装，然后使用自由函数，比如*read_at()，write_at()，async_read_at()，async_write_at()*。要从1000的地方读取50个字节，你需要使用下面的代码片段：

```
HANDLE file = ::CreateFile("readme.txt", GENERIC_READ, 0, 0, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL | FILE_FLAG_OVERLAPPED, 0);
windows::random_access_handle h(service, file);
char buf[50];
int bytes = read_at(h, 1000, buffer( buf));
std::string msg(buf, bytes);
std::cout << msg << std::endl;
```

对于Boost.Asio，随机访问句柄只提供随机访问，你不能把它们当作流句柄使用。也就是说，自由函数，比如：*read()，read_until()，write()*以及他们的相对的异步方法都不能在一个随机访问的句柄上使用。

*random_access_handle*类只有在I/O完成处理端口在使用中才有效（这是默认情况）。如果情况满足，*BOOST_ASIO_HAS_WINDOWS_RANDOM_ACCESS_HANDLE*就被定义

#### 对象句柄

你可以通过Windows句柄等待内核对象，比如修改通知，控制台输入，事件，内存资源通知，进程，信号量，线程或者可等待的计时器。或者简单来说，所有可以调用*WaitForSingleObject*的东西。你可以在它们上面创建一个*object_handle*封装，然后在上面使用*wait()*或者*async_wait()*：

```
void on_wait_complete(boost::system::error_code err) {}
...
HANDLE evt = ::CreateEvent(0, true, true, 0);
windows::object_handle h(service, evt);
// 同步等待
h.wait();
// 异步等待
h.async_wait(on_wait_complete);
```

### Boost.Asio POSIX特性

这些特性只在Unix操作系统上可用

#### 本地socket

Boost.Asio提供了对本地socket的基本支持（也就是著名的Unix 域socket）。

本地socket是一种只能被运行在主机上的应用访问的socket。你可以使用本地socket来实现简单的进程间通讯，连接两端的方式是把一个当作客户端而另一个当作服务端。对于本地socket，端点是一个文件，比如*/tmp/whatever*。很酷的一件事情是你可以给指定的文件赋予权限，从而禁止机器上指定的用户在文件上创建socket。

你可以用客户端socket的方式连接，如下面的代码片段：

```
local::stream_protocol::endpoint ep("/tmp/my_cool_app");
local::stream_protocol::socket sock(service);
sock.connect(ep);
```

你可以创建一个服务端socket，如下面的代码片段：

```
::unlink("/tmp/my_cool_app");
local::stream_protocol::endpoint ep("/tmp/my_cool_app");
local::stream_protocol::acceptor acceptor(service, ep);
local::stream_protocol::socket sock(service);
acceptor.accept(sock);
```

只要socket被成功创建，你就可以像用普通socket一样使用它；它和其他socket类有相同的成员方法，而且你也可以在使用了socket的自由函数中使用。

注意本地socket只有在目标操作系统支持它们的时候才可用，也就是*BOOST_ASIO_HAS_LOCAL_SOCKETS*（如果被定义）

#### 连接本地socket

最终，你可以连接两个socket，或者是无连接的（数据报），或者是基于连接的（流）：

```
// 基于连接
local::stream_protocol::socket s1(service);
local::stream_protocol::socket s2(service);
local::connect_pair(s1, s2);
// 数据报
local::datagram_protocol::socket s1(service);
local::datagram_protocol::socket s2(service);
local::connect_pair(s1, s2);
```

在内部，*connect_pair*使用的是不那么著名的*POSIX socketpair()*方法。基本上它所作的事情就是在没有复杂socket创建过程的情况下连接两个socket；而且只需要一行代码就可以完成。这在过去是实现线程通信的一种简单方式。而在现代编程中，你可以避免它，然后你会发现在处理使用了socket的遗留代码时它非常有用。

#### POSIX文件描述符

Boost.Asio允许在一些POSIX文件描述符，比如管道，标准I/O和其他设备（但是不是在普通文件上）上做一些同步和异步的操作。
一旦你为这样一个POSIX文件描述符创建了一个*stream_descriptor*实例，你就可以使用一些Boost.Asio提供的自由函数。比如*read()，read_until()，write()，async_read()，async_read_until()*和*async_write()*。

下面告诉你如何从stdin读取一行然后输出到stdout：

```
size_t read_up_to_enter(error_code err, size_t bytes) { ... }
posix::stream_descriptor in(service, ::dup(STDIN_FILENO));
posix::stream_descriptor out(service, ::dup(STDOUT_FILENO));
char buff[512];
int bytes = read(in, buffer(buff), read_up_to_enter);
write(out, buffer(buff, bytes));
```

*stream_descriptor*类只在目标操作系统支持的情况下有效，也就是*BOOST_ASIO_HAS_POSIX_STREAM_DESCRIPTOR*（如果定义了）

#### Fork

Boost.Asio支持在程序中使用*fork()*系统调用。你需要告诉*io_service*实例*fork()*方法什么时候会发生以及什么时候发生了。参考下面的代码片段：

```
service.notify_fork(io_service::fork_prepare);
if (fork() == 0) {
    // 子进程
    service.notify_fork(io_service::fork_child);
    ...
} else {
    // 父进程
    service.notify_fork(io_service::fork_parent);
    ... 
} 
```

这意味着会在不同的线程使用即将被调用的*service*。尽管Boost.Asio允许这样，我还是强烈推荐你使用多线程，因为使用*boost::thread*简直就是小菜一碟。

### 总结

为简单明了的代码而奋斗。学习和使用协程会最小化你需要做的调试工作，但仅仅是在代码中有潜在bug的情况下，Boost.Asio才会伸出援手，这一点在关于调试的章节中就已经讲过。

如果你需要使用SSL，Boost.Asio是支持基本的SSL编码的

最终，如果已经知道应用是针对专门的操作系统的，你可以享用Boost.Asio为那个特定的操作系统准备的特性。
