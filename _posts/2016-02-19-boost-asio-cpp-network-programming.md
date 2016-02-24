---
layout: post
title: "Boost.Asio 基本原理"
description: "Boost.Asio C++ Network Programming 翻译"

category: Boost.Asio
tags: [Boost.Asio,翻译]

imagefeature: mmoaay_bg.jpg
comments: true
share: true
---

## Boost.Asio基本原理

这一章涵盖了使用Boost.Asio时必须知道的一些事情。我们也将深入研究比同步编程更复杂、更有乐趣的异步编程。

### 网络API

这一部分包含了当使用Boost.Asio编写网络应用程序时必须知道的事情。

### Boost.Asio命名空间

Boost.Asio的所有内容都包含在boost::asio命名空间或者其子命名空间内。

* *boost::asio*：这是核心类和函数所在的地方。重要的类有io_service和streambuf。类似*read, read_at, read_until*方法，它们的异步方法，它们的写方法和异步写方法等自由函数也在这里。
* *boost::asio::ip*：这是网络通信部分所在的地方。重要的类有*address, endpoint, tcp,
 udp和icmp*，重要的自由函数有*connect*和*async_connect*。要注意的是在*boost::asio::ip::tcp::socket*中间，*socket*只是*boost::asio::ip::tcp*类中间的一个*typedef*关键字。
* *boost::asio::error*：这个命名空间包含了调用I/O例程时返回的错误码
* *boost::asio::ssl*：包含了SSL处理类的命名空间
* *boost::asio::local*：这个命名空间包含了POSIX特性的类
* *boost::asio::windows*：这个命名空间包含了Windows特性的类

### IP地址

对于IP地址的处理，Boost.Asio提供了*ip::address , ip::address_v4*和*ip::address_v6*类。
它们提供了相当多的函数。下面列出了最重要的几个：

* *ip::address(v4_or_v6_address)*:这个函数把一个v4或者v6的地址转换成*ip::address*
* *ip::address:from_string(str)*：这个函数根据一个IPv4地址（用.隔开的）或者一个IPv6地址（十六进制表示）创建一个地址。
* *ip::address::to_string()* ：这个函数返回这个地址的字符串。
* *ip::address_v4::broadcast([addr, mask])*:这个函数创建了一个广播地址
*ip::address_v4::any()*：这个函数返回一个能表示任意地址的地址。
* *ip::address_v4::loopback(), ip_address_v6::loopback()*：这个函数返回环路地址（为v4/v6协议）
* *ip::host_name()*：这个函数用string数据类型返回当前的主机名。

大多数情况你会选择用*ip::address::from_string*：

```
ip::address addr = ip::address::from_string("127.0.0.1");
```

如果你想通过一个主机名进行连接，下面的代码片段是无用的：

```
// 抛出异常
ip::address addr = ip::address::from_string("www.yahoo.com");
```


### 端点

端点是使用某个端口连接到的一个地址。不同类型的socket有它自己的*endpoint*类，比如*ip::tcp::endpoint、ip::udp::endpoint*和*ip::icmp::endpoint*

如果想连接到本机的80端口，你可以这样做：

```
ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 80);
```

有三种方式来让你建立一个端点：

* *endpoint()*：这是默认构造函数，某些时候可以用来创建UDP/ICMP socket。
* *endpoint(protocol, port)*：这个方法通常用来创建可以接受新连接的服务器端socket。
* *endpoint(addr, port)*:这个方法创建了一个连接到某个地址和端口的端点。

例子如下：

```
ip::tcp::endpoint ep1;
ip::tcp::endpoint ep2(ip::tcp::v4(), 80);
ip::tcp::endpoint ep3( ip::address::from_string("127.0.0.1), 80);
```

如果你想连接到一个主机（不是IP地址），你需要这样做：

```
// 输出 "87.248.122.122"
io_service service;
ip::tcp::resolver resolver(service);
ip::tcp::resolver::query query("www.yahoo.com", "80");
ip::tcp::resolver::iterator iter = resolver.resolve( query);
ip::tcp::endpoint ep = *iter;
std::cout << ep.address().to_string() << std::endl;
```

你可以用你需要的socket类型来替换tcp。首先，为你想要查询的名字创建一个查询器，然后用resolve()函数解析它。如果成功，它至少会返回一个入口。你可以利用返回的迭代器，使用第一个入口或者遍历整个列表来拿到全部的入口。

给定一个端点，可以获得他的地址，端口和IP协议（v4或者v6）：

```
std::cout << ep.address().to_string() << ":" << ep.port()
<< "/" << ep.protocol() << std::endl;
```

### 套接字

Boost.Asio有三种类型的套接字类：*ip::tcp, ip::udp*和*ip::icmp*。当然它也是可扩展的，你可以创建自己的socket类，尽管这相当复杂。如果你选择这样做，参照一下*boost/asio/ip/tcp.hpp, boost/asio/ip/udp.hpp*和*boost/asio/ip/icmp.hpp*。它们都是含有内部typedef关键字的超小类。

你可以把*ip::tcp, ip::udp, ip::icmp*类当作占位符；它们可以让你便捷地访问其他类/函数，如下所示：

* *ip::tcp::socket, ip::tcp::acceptor, ip::tcp::endpoint,ip::tcp::resolver, ip::tcp::iostream*
* *ip::udp::socket, ip::udp::endpoint, ip::udp::resolver*
* *ip::icmp::socket, ip::icmp::endpoint, ip::icmp::resolver*

*socket*类创建一个相应的*socket*。而且总是在构造的时候传入io_service实例：

```
io_service service;
ip::udp::socket sock(service)
sock.set_option(ip::udp::socket::reuse_address(true));
```

每一个socket的名字都是一个typedef关键字

* *ip::tcp::socket = basic_stream_socket<tcp>*
* *ip::udp::socket = basic_datagram_socket<udp>*
* *ip::icmp::socket = basic_raw_socket<icmp>*

### 同步错误码

所有的同步函数都有抛出异常或者返回错误码的重载，比如下面的代码片段：

```
sync_func( arg1, arg2 ... argN); // 抛出异常
boost::system::error_code ec;
sync_func( arg1 arg2, ..., argN, ec); // 返回错误码
```

在这一章剩下的部分，你会见到大量的同步函数。简单起见，我省略了有返回错误码的重载，但是不可否认它们确实是存在的。

### socket成员方法

这些方法被分成了几组。并不是所有的方法都可以在各个类型的套接字里使用。这个部分的结尾将有一个列表来展示各个方法分别属于哪个socket类。

注意所有的异步方法都立刻返回，而它们相对的同步实现需要操作完成之后才能返回。

### 连接相关的函数

这些方法是用来连接或绑定socket、断开socket字连接以及查询连接是活动还是非活动的：

* *assign(protocol,socket)*：这个函数分配了一个原生的socket给这个socket实例。当处理老（旧）程序时会使用它（也就是说，原生socket已经被建立了）
* *open(protocol)*：这个函数用给定的IP协议（v4或者v6）打开一个socket。你主要在UDP/ICMP socket，或者服务端socket上使用。
* *bind(endpoint)*：这个函数绑定到一个地址
* *connect(endpoint)*：这个函数用同步的方式连接到一个地址
* *async_connect(endpoint)*：这个函数用异步的方式连接到一个地址
* *is_open()*：如果套接字已经打开，这个函数返回true
* *close()*：这个函数用来关闭套接字。调用时这个套接字上任何的异步操作都会被立即关闭，同时返回*error::operation_aborted*错误码。
* *shutdown(type_of_shutdown)*：这个函数立即使send或者receive操作失效，或者两者都失效。
* *cancel()*：这个函数取消套接字上所有的异步操作。这个套接字上任何的异步操作都会立即结束，然后返回*error::operation_aborted*错误码。

例子如下：

```
ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 80);
ip::tcp::socket sock(service);
sock.open(ip::tcp::v4()); n
sock.connect(ep);
sock.write_some(buffer("GET /index.html\r\n"));
char buff[1024]; sock.read_some(buffer(buff,1024));
sock.shutdown(ip::tcp::socket::shutdown_receive);
sock.close();
```


### 读写函数

这些是在套接字上执行I/O操作的函数。

对于异步函数来说，处理程序的格式*void handler(const boost::system::error_code& e, size_t bytes)*都是一样的

* *async_receive(buffer, [flags,] handler)*：这个函数启动从套接字异步接收数据的操作。
* *async_read_some(buffer,handler)*：这个函数和*async_receive(buffer, handler)*功能一样。
* *async_receive_from(buffer, endpoint[, flags], handler)*：这个函数启动从一个指定端点异步接收数据的操作。
* *async_send(buffer [, flags], handler)*：这个函数启动了一个异步发送缓冲区数据的操作。
* *async_write_some(buffer, handler)*：这个函数和a*sync_send(buffer, handler)*功能一致。
* *async_send_to(buffer, endpoint, handler)*：这个函数启动了一个异步send缓冲区数据到指定端点的操作。
* *receive(buffer [, flags])*：这个函数异步地从所给的缓冲区读取数据。在读完所有数据或者错误出现之前，这个函数都是阻塞的。
* *read_some(buffer)*：这个函数的功能和*receive(buffer)*是一致的。
* * receive_from(buffer, endpoint [, flags])*：这个函数异步地从一个指定的端点获取数据并写入到给定的缓冲区。在读完所有数据或者错误出现之前，这个函数都是阻塞的。
* *send(buffer [, flags])*：这个函数同步地发送缓冲区的数据。在所有数据发送成功或者出现错误之前，这个函数都是阻塞的。
* *write_some(buffer)*：这个函数和*send(buffer)*的功能一致。
* *send_to(buffer, endpoint [, flags])*：这个函数同步地把缓冲区数据发送到一个指定的端点。在所有数据发送成功或者出现错误之前，这个函数都是阻塞的。
* *available()*：这个函数返回有多少字节的数据可以无阻塞地进行同步读取。

稍后我们将讨论缓冲区。让我们先来了解一下标记。标记的默认值是0，但是也可以是以下几种：

* *ip::socket_type::socket::message_peek*：这个标记只监测并返回某个消息，但是下一次读消息的调用会重新读取这个消息。
* *ip::socket_type::socket::message_out_of_band*：这个标记处理带外（OOB）数据，OOB数据是被标记为比正常数据更重要的数据。关于OOB的讨论在这本书的内容之外。
* *ip::socket_type::socket::message_do_not_route*：这个标记指定数据不使用路由表来发送。
* *ip::socket_type::socket::message_end_of_record*：这个标记指定的数据标识了记录的结束。在Windows下不支持。

你最常用的可能是*message_peek*，使用方法请参照下面的代码片段：

```
char buff[1024];
sock.receive(buffer(buff), ip::tcp::socket::message_peek );
memset(buff,1024, 0);
// 重新读取之前已经读取过的内容
sock.receive(buffer(buff) );
```

下面的是一些教你如何同步或异步地从不同类型的套接字上读取数据的例子：

* 例1是在一个TCP套接字上进行同步读写：

```
ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 80);
ip::tcp::socket sock(service);
sock.connect(ep);
sock.write_some(buffer("GET /index.html\r\n"));
std::cout << "bytes available " << sock.available() << std::endl;
char buff[512];
size_t read = sock.read_some(buffer(buff));
```


* 例2是在一个UDP套接字上进行同步读写：

```
ip::udp::socket sock(service);
sock.open(ip::udp::v4());
ip::udp::endpoint receiver_ep("87.248.112.181", 80);
sock.send_to(buffer("testing\n"), receiver_ep);
char buff[512];
ip::udp::endpoint sender_ep;
sock.receive_from(buffer(buff), sender_ep);
```


*[？注意：就像上述代码片段所展示的那样，使用receive_from从一个UDP套接字读取数据时，你需要构造一个默认的端点]*

* 例3是从一个UDP服务套接字中异步读取数据：

```
using namespace boost::asio;
io_service service;
ip::udp::socket sock(service);
boost::asio::ip::udp::endpoint sender_ep;
char buff[512];
void on_read(const boost::system::error_code & err, std::size_t read_bytes) {
    std::cout << "read " << read_bytes << std::endl;
    sock.async_receive_from(buffer(buff), sender_ep, on_read);
}
int main(int argc, char* argv[]) {
    ip::udp::endpoint ep(ip::address::from_string("127.0.0.1"),
8001);
    sock.open(ep.protocol());
    sock.set_option(boost::asio::ip::udp::socket::reuse_address(true));
    sock.bind(ep);
    sock.async_receive_from(buffer(buff,512), sender_ep, on_read);
    service.run();
}
```

### 套接字控制：

这些函数用来处理套接字的高级选项：

* *get_io_service()*：这个函数返回构造函数中传入的io_service实例
* *get_option(option)*：这个函数返回一个套接字的属性
* *set_option(option)*：这个函数设置一个套接字的属性
* *io_control(cmd)*：这个函数在套接字上执行一个I/O指令

这些是你可以获取/设置的套接字选项：

| 名字 | 描述 | 类型 |
| -- | -- |
| broadcast | 如果为true，允许广播消息 | bool |
| debug | 如果为true，启用套接字级别的调试 | bool | 
|do_not_route | 如果为true，则阻止路由选择只使用本地接口 | bool | 
| enable_connection_aborted | 如果为true，记录在accept()时中断的连接 | bool | 
| keep_alive | 如果为true，会发送心跳 | bool | 
| linger | 如果为true，套接字会在有未发送数据的情况下挂起close() | bool | 
| receive_buffer_size | 套接字接收缓冲区大小 | int | 
| receive_low_watemark  | 规定套接字输入处理的最小字节数 | int | 
| reuse_address | 如果为true，套接字能绑定到一个已用的地址 | bool | 
| send_buffer_size  | 套接字发送缓冲区大小 | int | 
| send_low_watermark | 规定套接字数据发送的最小字节数 | int | 
| ip::v6_only | 如果为true，则只允许IPv6的连接 | bool | 

每个名字代表了一个内部套接字typedef或者类。下面是对它们的使用：

```
ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 80);
ip::tcp::socket sock(service);
sock.connect(ep);
// TCP套接字可以重用地址
ip::tcp::socket::reuse_address ra(true);
sock.set_option(ra);
// 获取套接字读取的数据
ip::tcp::socket::receive_buffer_size rbs;
sock.get_option(rbs);
std::cout << rbs.value() << std::endl;
// 把套接字的缓冲区大小设置为8192
ip::tcp::socket::send_buffer_size sbs(8192);
sock.set_option(sbs);
```

*[?在上述特性工作之前，套接字要被打开。否则，会抛出异常]*

### TCP VS UDP VS ICMP

就像我之前所说，不是所有的成员方法在所有的套接字类中都可用。我做了一个包含成员函数不同点的列表。如果一个成员函数没有出现在这，说明它在所有的套接字类都是可用的。

| 名字 | TCP | UDP | ICMP |
| -- | -- | -- | -- |
|async_read_some | 是 | - | - |
|async_receive_from | - | 是 | 是 |
|async_write_some | 是 | - | - |
|async_send_to | - | 是 | 是 |
|read_some | 是 | - | - |
|receive_from | - | 是 | 是 |
|write_some | 是 | - | - |
|send_to | - | 是 | 是 |

### 其他方法

其他与连接和I/O无关的函数如下：

* *local_endpoint()*：这个方法返回套接字本地连接的地址。
* *remote_endpoint()*：这个方法返回套接字连接到的远程地址。
* *native_handle()*：这个方法返回原始套接字的处理程序。你只有在调用一个Boost.Asio不支持的原始方法时才需要用到它。
* *non_blocking()*：如果套接字是非阻塞的，这个方法返回true，否则false。
* *native_non_blocking()*：如果套接字是非阻塞的，这个方法返回true，否则返回false。但是，它是基于原生的套接字来调用本地的api。所以通常来说，你不需要调用这个方法（non_blocking()已经缓存了这个结果）；你只有在直接调用native_handle()这个方法的时候才需要用到这个方法。
* *at_mark()*：如果套接字要读的是一段OOB数据，这个方法返回true。这个方法你很少会用到。

### 其他需要考虑的事情

最后要注意的一点，套接字实例不能被拷贝，因为拷贝构造方法和＝操作符是不可访问的。

```
ip::tcp::socket s1(service), s2(service);
s1 = s2; // 编译时报错
ip::tcp::socket s3(s1); // 编译时报错
```

这是非常有意义的，因为每一个实例都拥有并管理着一个资源（原生套接字本身）。如果我们允许拷贝构造，结果是我们会有两个实例拥有同样的原生套接字；这样我们就需要去处理所有者的问题（让一个实例拥有所有权？或者使用引用计数？还是其他的方法）Boost.Asio选择不允许拷贝（如果你想要创建一个备份，请使用共享指针）

```
typedef boost::shared_ptr<ip::tcp::socket> socket_ptr;
socket_ptr sock1(new ip::tcp::socket(service));
socket_ptr sock2(sock1); // ok
socket_ptr sock3;			
sock3 = sock1; // ok
```

### 套接字缓冲区

当从一个套接字读写内容时，你需要一个缓冲区，用来保存读取和写入的数据。缓冲区内存的有效时间必须比I/O操作的时间要长；你需要保证它们在I/O操作结束之前不被释放。
对于同步操作来说，这很容易；当然，这个缓冲区在receive和send时都存在。

```
char buff[512];
...
sock.receive(buffer(buff));
strcpy(buff, "ok\n");
sock.send(buffer(buff));
```

但是在异步操作时就没这么简单了，看下面的代码片段：
	
```
// 非常差劲的代码 ...
void on_read(const boost::system::error_code & err, std::size_t read_bytes)
{ ... }
void func() {
    char buff[512];
    sock.async_receive(buffer(buff), on_read);
}
```

在我们调用*async_receive()*之后，buff就已经超出有效范围，它的内存当然会被释放。当我们开始从套接字接收一些数据时，我们会把它们拷贝到一片已经不属于我们的内存中；它可能会被释放，或者被其他代码重新开辟来存入其他的数据，结果就是：内存冲突。

对于上面的问题有几个解决方案：

* 使用全局缓冲区
* 创建一个缓冲区，然后在操作结束时释放它
* 使用一个集合对象管理这些套接字和其他的数据，比如缓冲区数组

第一个方法显然不是很好，因为我们都知道全局变量非常不好。此外，如果两个实例使用同一个缓冲区怎么办？

下面是第二种方式的实现：					
	
```
void on_read(char * ptr, const boost::system::error_code & err, std::size_t read_bytes) {						
    delete[] ptr;
}
....
char * buff = new char[512];
sock.async_receive(buffer(buff, 512), boost::bind(on_read,buff,_1,_2))
```

或者，如果你想要缓冲区在操作结束后自动超出范围，使用共享指针

```
struct shared_buffer {
    boost::shared_array<char> buff;
    int size;
    shared_buffer(size_t size) : buff(new char[size]), size(size) {
    }
    mutable_buffers_1 asio_buff() const {
        return buffer(buff.get(), size);
    }
};


// 当on_read超出范围时, boost::bind对象被释放了,
// 同时也会释放共享指针
void on_read(shared_buffer, const boost::system::error_code & err, std::size_t read_bytes) {}
sock.async_receive(buff.asio_buff(), boost::bind(on_read,buff,_1,_2));
```

shared_buffer类拥有实质的*shared_array<>*，*shared_array<>*存在的目的是用来保存*shared_buffer*实例的拷贝－当最后一个*share_array<>*元素超出范围时，*shared_array<>*就被自动销毁了，而这就是我们想要的结果。

因为Boost.Asio会给完成处理句柄保留一个拷贝，当操作完成时就会调用这个完成处理句柄，所以你的目的达到了。那个拷贝是一个boost::bind的仿函数，它拥有着实际的*shared_buffer*实例。这是非常优雅的！

第三个选择是使用一个连接对象来管理套接字和其他数据，比如缓冲区，通常来说这是正确的解决方案但是非常复杂。在这一章的末尾我们会对这种方法进行讨论。

### 缓冲区封装函数

纵观所有代码，你会发现：无论什么时候，当我们需要对一个buffer进行读写操作时，代码会把实际的缓冲区对象封装在一个buffer()方法中，然后再把它传递给方法调用：

```
char buff[512];
sock.async_receive(buffer(buff), on_read);
```

基本上我们都会把缓冲区包含在一个类中以便Boost.Asio的方法能遍历这个缓冲区，比方说，使用下面的代码：				

```
sock.async_receive(some_buffer, on_read);
```

实例*some_buffer*需要满足一些需求，叫做*ConstBufferSequence*或者*MutableBufferSequence*（你可以在Boost.Asio的文档中查看它们）。创建你自己的类去处理这些需求的细节是非常复杂的，但是Boost.Asio已经提供了一些类用来处理这些需求。所以你不用直接访问这些缓冲区，而可以使用*buffer()*方法。

自信地讲，你可以把下面列出来的类型都包装到一个buffer()方法中：

* 一个char[] const 数组
* 一个字节大小的void *指针
* 一个std::string类型的字符串
* 一个POD const数组（POD代表纯数据，这意味着构造器和释放器不做任何操作）
* 一个pod数据的std::vector
* 一个包含pod数据的boost::array
* 一个包含pod数据的std::array

下面的代码都是有效的：

```
struct pod_sample { int i; long l; char c; };
...
char b1[512];
void * b2 = new char[512];
std::string b3; b3.resize(128);
pod_sample b4[16];
std::vector<pod_sample> b5; b5.resize(16);
boost::array<pod_sample,16> b6;
std::array<pod_sample,16> b7;
sock.async_send(buffer(b1), on_read);
sock.async_send(buffer(b2,512), on_read);
sock.async_send(buffer(b3), on_read);
sock.async_send(buffer(b4), on_read);
sock.async_send(buffer(b5), on_read);
sock.async_send(buffer(b6), on_read);
sock.async_send(buffer(b7), on_read);
```

总的来说就是：与其创建你自己的类来处理*ConstBufferSequence*或者*MutableBufferSequence*的需求，不如创建一个能在你需要的时候保留缓冲区，然后返回一个mutable_buffers_1实例的类，而我们早在shared_buffer类中就这样做了。

### read/write/connect自由函数

Boost.Asio提供了处理I/O的自由函数，我们分四组来分析它们。

#### connect方法

这些方法把套接字连接到一个端点。

* *connect(socket, begin [, end] [, condition])*：这个方法遍历队列中从start到end的端点来尝试同步连接。begin迭代器是调用*socket_type::resolver::query*的返回结果（你可能需要回顾一下端点这个章节）。特别提示end迭代器是可选的；你可以忽略它。你还可以提供一个condition的方法给每次连接尝试之后调用。用法是*Iterator connect_condition(const boost::system::error_code & err,Iterator next);*。你可以选择返回一个不是*next*的迭代器，这样你就可以跳过一些端点。
* *async_connect(socket, begin [, end] [, condition], handler)*：这个方法异步地调用连接方法，在结束时，它会调用完成处理方法。用法是*void handler(constboost::system::error_code & err, Iterator iterator);*。传递给处理方法的第二个参数是连接成功端点的迭代器（或者end迭代器）。

它的例子如下：

```
using namespace boost::asio::ip;
tcp::resolver resolver(service);
tcp::resolver::iterator iter = resolver.resolve(tcp::resolver::query("www.yahoo.com","80"));
tcp::socket sock(service);
connect(sock, iter);
```

一个主机名可以被解析成多个地址，而*connect*和*async_connect*能很好地把你从尝试每个地址然后找到一个可用地址的繁重工作中解放出来，因为它们已经帮你做了这些。

#### read/write方法

这些方法对一个流进行读写操作（可以是套接字，或者其他表现得像流的类）：

* *async_read(stream, buffer [, completion] ,handler)*：这个方法异步地从一个流读取。结束时其处理方法被调用。处理方法的格式是：*void handler(const boost::system::error_ code & err, size_t bytes)*;。你可以选择指定一个完成处理方法。完成处理方法会在每个*read*操作调用成功之后调用，然后告诉Boost.Asio *async_read*操作是否完成（如果没有完成，它会继续读取）。它的格式是：*size_t completion(const boost::system::error_code& err, size_t bytes_transfered) *。当这个完成处理方法返回0时，我们认*为read*操作完成；如果它返回一个非0值，它表示了下一个*async_read_some*操作需要从流中读取的字节数。接下来会有一个例子来详细展示这些。
* *async_write(stream, buffer [, completion], handler)*：这个方法异步地向一个流写入数据。参数的意义和*async_read*是一样的。
* *read(stream, buffer [, completion])*：这个方法同步地从一个流中读取数据。参数的意义和*async_read*是一样的。
* *write(stream, buffer [, completion])*:  这个方法同步地向一个流写入数据。参数的意义和*async_read*是一样的。

```
async_read(stream, stream_buffer [, completion], handler)
async_write(strean, stream_buffer [, completion], handler)
write(stream, stream_buffer [, completion])
read(stream, stream_buffer [, completion]) 
```

首先，要注意第一个参数变成了流，而不单是socket。这个参数包含了socket但不仅仅是socket。比如，你可以用一个Windows的文件句柄来替代socket。
当下面情况出现时，所有read和write操作都会结束：

* 可用的缓冲区满了（当读取时）或者所有的缓冲区已经被写入（当写入时）
* 完成处理方法返回0（如果你提供了这么一个方法）
* 错误发生时

下面的代码会异步地从一个socket中间读取数据直到读取到’\n’：

```
io_service service;
ip::tcp::socket sock(service);
char buff[512];
int offset = 0;
size_t up_to_enter(const boost::system::error_code &, size_t bytes) {
    for ( size_t i = 0; i < bytes; ++i)
        if ( buff[i + offset] == '\n') 
            return 0;
    return 1; 
 }
void on_read(const boost::system::error_code &, size_t) {}
...
async_read(sock, buffer(buff), up_to_enter, on_read); 
```

Boost.Asio也提供了一些简单的完成处理仿函数： 

* transfer_at_least(n)
* transfer_exactly(n)
* transfer_all()

例子如下： 

```
char buff[512]; 
void on_read(const boost::system::error_code &, size_t) {} 
// 读取32个字节 
async_read(sock, buffer(buff), transfer_exactly(32), on_read);
```

上述的4个方法，不使用普通的缓冲区，而使用由Boost.Asio的*std::streambuf*类继承来的*stream_buffer*方法。stl流和流缓冲区非常复杂；下面是例子： 

```
io_service service;  
void on_read(streambuf& buf, const boost::system::error_code &, size_t) { 
    std::istream in(&buf);
    std::string line;
    std::getline(in, line);
    std::cout << "first line: " << line << std::endl; 
}
int main(int argc, char* argv[]) { 
    HANDLE file = ::CreateFile("readme.txt", GENERIC_READ, 0, 0, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL | FILE_FLAG_OVERLAPPED, 0);
    windows::stream_handle h(service, file);
    streambuf buf;
    async_read(h, buf, transfer_exactly(256), boost::bind(on_read,boost::ref(buf),_1,_2));
    service.run(); 
}
```

在这里，我向你们展示了如何在一个Windows文件句柄上调用*async_read*。读取前256个字符，然后把它们保存到缓冲区中，当操作结束时。*on_read*被调用，再创建*std::istream*用来传递缓冲区，读取第一行（*std::getline*），最后把它输出到命令行中。 

#### read_until/async_read_until方法 

这些方法在条件满足之前会一直读取： 

* *async_read_until(stream, stream_buffer, delim, handler)*:这个方法启动一个异步*read*操作。*read*操作会在读取到某个分隔符时结束。分隔符可以是字符,*std::string*或者*boost::regex*。处理方法的格式为：*void handler(const boost::system::error_code & err, size_t bytes);*。
* *async_read_until(strem, stream_buffer, completion, handler)*：这个方法和之前的方法是一样的，但是没有分隔符，而是一个完成处理方法。完成处理方法的格式为：*pair< iterator,bool > completion(iterator begin, iterator end);*，其中迭代器的类型为*buffers_iterator< streambuf::const_buffers_type >*。你需要记住的是这个迭代器是支持随机访问的。你扫描整个区间（begin，end），然后决定read操作是否应该结束。返回的结果是一个结果对，第一个成员是一个迭代器，它指向最后被这个方法访问的字符；第二个成员指定read操作是否需要结束，需要时返回true，否则返回false。
* *read_until(stream, stream_buffer, delim)*：这个方法执行一个同步的*read*操作，参数的意义和*async_read_until*一样。
* *read_until(stream, stream_buffer, completion)*：这个方法执行一个同步的read操作，参数的意义和*async_read_until*一样。

下面这个例子在读到一个指定的标点符号之前会一直读取：

```
typedef buffers_iterator<streambuf::const_buffers_type> iterator;
std::pair<iterator, bool> match_punct(iterator begin, iterator end) {
    while ( begin != end)
        if ( std::ispunct(*begin))
            return std::make_pair(begin,true);
    return std::make_pair(end,false);
}
void on_read(const boost::system::error_code &, size_t) {}
...
streambuf buf;
async_read_until(sock, buf, match_punct, on_read);
```

如果我们想读到一个空格时就结束，我们需要把最后一行修改为：

```
async_read_until(sock, buff, ' ', on_read);
```

#### *_at方法

这些方法用来在一个流上面做随机存取操作。由你来指定*read*和*write*操作从什么地方开始（*offset*）：

* *async_read_at(stream, offset, buffer [, completion], handler)*：这个方法在指定的流的offset处开始执行一个异步的read操作，当操作结束时，它会调用handler。handler的格式为：*void handler(const boost::system::error_code&  err, size_t bytes);*。*buffer*可以是普通的*wrapper()*封装或者*streambuf*方法。如果你指定一个completion方法，它会在每次read操作成功之后调用，然后告诉Boost.Asio *async_read_at*操作已经完成（如果没有，则继续读取）。它的格式为：*size_t  completion(const boost::system::error_code& err, size_t bytes);*。当completion方法返回0时，我们认为*read*操作完成了；如果返回一个非零值，它代表了下一次调用流的*async_read_some_at*方法的最大读取字节数。
* *async_write_at(stream, offset, buffer [, completion], handler)*：这个方法执行一个异步的write操作。参数的意义和*async_read_at*是一样的
* *read_at(stream, offset, buffer [, completion])*：这个方法在一个执行的流上，指定的*offset*处开始read。参数的意义和*async_read_at*是一样的
* *write_at(stream, offset, buffer [, completion])*：这个方法在一个执行的流上，指定的*offset*处开始write。参数的意义和*async_read_at*是一样的

这些方法不支持套接字。它们用来处理流的随机访问；也就是说，流是可以随机访问的。套接字显然不是这样（套接字是不可回溯的）。

下面这个例子告诉你怎么从一个文件偏移为256的位置读取128个字节：

```
io_service service;
int main(int argc, char* argv[]) {
    HANDLE file = ::CreateFile("readme.txt", GENERIC_READ, 0, 0, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL | FILE_FLAG_OVERLAPPED, 0);
    windows::random_access_handle h(service, file);
    streambuf buf;
    read_at(h, 256, buf, transfer_exactly(128));
    std::istream in(&buf);
    std::string line;
    std::getline(in, line);
    std::cout << "first line: " << line << std::endl;
}
```

### 异步编程

这部分对异步编程时可能碰到的一些问题进行了深入的探究。我建议你先读一遍，然后在接下来读这本书的过程中，再经常回过头来看看，从而增强你对这部分的理解。

#### 异步的需求

就像我之前所说的，同步编程比异步编程简单很多。这是因为，线性的思考是很简单的（调用A，调用A结束，调用B，调用B结束，然后继续，这是以事件处理的方式来思考）。后面你会碰到这种情况，比如：五件事情，你不知道它们执行的顺序，也不知道他们是否会执行！

尽管异步编程更难，但是你会更倾向于选择使用它，比如：写一个需要处理很多并发访问的服务端。并发访问越多，异步编程就比同步编程越简单。

假设：你有一个需要处理1000个并发访问的应用，从客户端发给服务端的每个信息都会再返回给客户端，以‘\n’结尾。

同步方式的代码，1个线程：

```
using namespace boost::asio;
struct client {
    ip::tcp::socket sock;
    char buff[1024]; // 每个信息最多这么大
    int already_read; // 你已经读了多少
};
std::vector<client> clients;
void handle_clients() {
    while ( true)
        for ( int i = 0; i < clients.size(); ++i)
            if ( clients[i].sock.available() ) on_read(clients[i]);
}
void on_read(client & c) {
    int to_read = std::min( 1024 - c.already_read, c.sock.available());
    c.sock.read_some( buffer(c.buff + c.already_read, to_read));
    c.already_read += to_read;
    if ( std::find(c.buff, c.buff + c.already_read, '\n') < c.buff + c.already_read) {
        int pos = std::find(c.buff, c.buff + c.already_read, '\n') - c.buff;
        std::string msg(c.buff, c.buff + pos);
        std::copy(c.buff + pos, c.buff + 1024, c.buff);
        c.already_read -= pos;
        on_read_msg(c, msg);
    }
}
void on_read_msg(client & c, const std::string & msg) {
    // 分析消息，然后返回
    if ( msg == "request_login")
        c.sock.write( "request_ok\n");
    else if ...
}
```

有一种情况是在任何服务端（和任何基于网络的应用）都需要避免的，就是代码无响应的情况。在我们的例子里，我们需要*handle_clients()*方法尽可能少的阻塞。如果方法在某个点上阻塞，任何进来的信息都需要等待方法解除阻塞才能被处理。

为了保持响应，只在一个套接字有数据的时候我们才读，也就是说，*if ( clients[i].sock.available() ) on_read(clients[i])*。在*on_read*时，我们只读当前可用的；调用*read_until(c.sock, buffer(...),  '\n')*会是一个非常糟糕的选择，因为直到我们从一个指定的客户端读取了完整的消息之前，它都是阻塞的（我们永远不知道它什么时候会读取到完整的消息）

这里的瓶颈就是*on_read_msg()*方法；当它执行时，所有进来的消息都在等待。一个良好的*on_read_msg()*方法实现会保证这种情况基本不会发生，但是它还是会发生（有时候向一个套接字写入数据，缓冲区满了时，它会被阻塞）
同步方式的代码，10个线程

```
using namespace boost::asio;
struct client {
　  // ... 和之前一样
    bool set_reading() {
        boost::mutex::scoped_lock lk(cs_);
        if ( is_reading_) return false; // 已经在读取
        else { is_reading_ = true; return true; }
    }
    void unset_reading() {
        boost::mutex::scoped_lock lk(cs_);
        is_reading_ = false;
    }
private:
    boost::mutex cs_;
    bool is_reading_;
};
std::vector<client> clients;
void handle_clients() {
    for ( int i = 0; i < 10; ++i)
        boost::thread( handle_clients_thread);
}
void handle_clients_thread() {
    while ( true)
        for ( int i = 0; i < clients.size(); ++i)
            if ( clients[i].sock.available() )
                if ( clients[i].set_reading()) {
                    on_read(clients[i]);
                    clients[i].unset_reading();
                }
}
void on_read(client & c) {
    // 和之前一样
}
void on_read_msg(client & c, const std::string & msg) {
    // 和之前一样
}
```

为了使用多线程，我们需要对线程进行同步，这就是*set_reading()*和*set_unreading()*所做的。*set_reading()*方法非常重要，比如你想要一步实现“判断是否在读取然后标记为读取中”。但这是有两步的（“判断是否在读取”和“标记为读取中”），你可能会有两个线程同时为一个客户端判断是否在读取，然后你会有两个线程同时为一个客户端调用*on_read*，结果就是数据冲突甚至导致应用崩溃。

你会发现代码变得极其复杂。

同步编程有第三个选择，就是为每个连接开辟一个线程。但是当并发的线程增加时，这就成了一种灾难性的情况。

然后，让我们来看异步编程。我们不断地异步读取。当一个客户端请求某些东西时，*on_read*被调用，然后回应，然后等待下一个请求（然后开始另外一个异步的read操作）。

异步方式的代码，10个线程

```
using namespace boost::asio;
io_service service;
struct client {
    ip::tcp::socket sock;
    streambuf buff; // 从客户端取回结果
}
std::vector<client> clients;
void handle_clients() {
    for ( int i = 0; i < clients.size(); ++i)
        async_read_until(clients[i].sock, clients[i].buff, '\n', boost::bind(on_read, clients[i], _1, _2));
    for ( int i = 0; i < 10; ++i)
        boost::thread(handle_clients_thread);
}
void handle_clients_thread() {
    service.run();
}
void on_read(client & c, const error_code & err, size_t read_bytes) {
    std::istream in(&c.buff);
    std::string msg;
    std::getline(in, msg);
    if ( msg == "request_login")
        c.sock.async_write( "request_ok\n", on_write);
    else if ...
    ...
    // 等待同一个客户端下一个读取操作
    async_read_until(c.sock, c.buff, '\n', boost::bind(on_read, c, _1, _2));
}
```

发现代码变得有多简单了吧？client结构里面只有两个成员，*handle_clients()*仅仅调用了*async_read_until*，然后它创建了10个线程，每个线程都调用*service.run()*。这些线程会处理所有来自客户端的异步read操作，然后分发所有向客户端的异步write操作。另外需要注意的一件事情是：*on_read()*一直在为下一次异步read操作做准备（看最后一行代码）。

#### 异步run(), run_one(), poll(), poll_ one()

为了实现监听循环，*io_service*类提供了4个方法，比如：*run(), run_one(), poll()*和*poll_one()*。虽然大多数时候使用*service.run()*就可以，但是你还是需要在这里学习其他方法实现的功能。

##### 持续运行

再一次说明，如果有等待执行的操作，*run()*会一直执行，直到你手动调用*io_service::stop()*。为了保证*io_service*一直执行，通常你添加一个或者多个异步操作，然后在它们被执行时，你继续一直不停地添加异步操作，比如下面代码：

```
using namespace boost::asio;
io_service service;
ip::tcp::socket sock(service);
char buff_read[1024], buff_write[1024] = "ok";
void on_read(const boost::system::error_code &err, std::size_t bytes);
void on_write(const boost::system::error_code &err, std::size_t bytes)
{
    sock.async_read_some(buffer(buff_read), on_read);
}
void on_read(const boost::system::error_code &err, std::size_t bytes)
{
    // ... 处理读取操作 ...
    sock.async_write_some(buffer(buff_write,3), on_write);
}
void on_connect(const boost::system::error_code &err) {
    sock.async_read_some(buffer(buff_read), on_read);
}
int main(int argc, char* argv[]) {
    ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 2001);
    sock.async_connect(ep, on_connect);
    service.run();
}
```

1. 当*service.run()*被调用时，有一个异步操作在等待。
2. 当socket连接到服务端时，*on_connect*被调用了，它会添加一个异步操作。
3. 当*on_connect*结束时，我们会留下一个等待的操作（*read*）。
4. 当*on_read*被调用时，我们写入一个回应，这又添加了另外一个等待的操作。
5. 当*on_read*结束时，我们会留下一个等待的操作*（write*）。
6. 当*on_write*操作被调用时，我们从服务端读取另外一个消息，这也添加了另外一个等待的操作。
7. 当*on_write*结束时，我们有一个等待的操作（read）。
8. 然后一直继续循环下去，直到我们关闭这个应用。

#####run_one(), poll(), poll_one() 方法

我在之前说过异步方法的handler是在调用了*io_service::run*的线程里被调用的。因为在至少90%～95%的时候，这是你唯一要用到的方法，所以我就把它说得简单了。对于调用了*run_one(), poll()*或者*poll_one()*的线程这一点也是适用的。

*run_one()*方法最多执行和分发一个异步操作：
* 如果没有等待的操作，方法立即返回0
* 如果有等待操作，方法在第一个操作执行之前处于阻塞状态，然后返回1

你可以认为下面两段代码是等效的：

```
io_service service;
service.run(); // 或者
while ( !service.stopped()) service.run_once();
```

你可以使用*run_once()*启动一个异步操作，然后等待它执行完成。

```
io_service service;
bool write_complete = false;
void on_write(const boost::system::error_code & err, size_t bytes)
{ write_complete = true; }
 …
std::string data = "login ok”;
write_complete = false;
async_write(sock, buffer(data), on_write);
do service.run_once() while (!write_complete);
```

还有一些使用*run_one()*方法的例子，包含在Boost.Asio诸如*blocking_tcp_client.cpp*和*blocking_udp_client.cpp*的文件中。

*poll_one*方法以非阻塞的方式最多运行一个准备好的等待操作：

* 如果至少有一个等待的操作，而且准备好以非阻塞的方式运行，poll_one方法会运行它并且返回1
* 否则，方法立即返回0

操作正在等待并准备以非阻塞方式运行，通常意味着如下的情况：

* 一个计时器过期了，然后它的*async_wait*处理方法需要被调用
* 一个I/O操作完成了（比如*async_read*），然后它的hanlder需要被调用
* 之前被加入*io_services*实例队列中的自定义handler（这会在之后的章节中详解）

你可以使用*poll_one*去保证所有I/O操作的handler完成运行，同时做一些其他的工作

```
io_service service;
while ( true) {
    // 运行所有完成了IO操作的handler
    while ( service.poll_one()) ;
    // ... 在这里做其他的事情 …
} 
```

*poll()*方法会以非阻塞的方式运行所有等待的操作。下面两段代码是等效的：

```
io_service service;
service.poll(); // 或者
while ( service.poll_one()) ;
```

所有上述方法都会在失败的时候抛出*boost::system::system_error*异常。这是我们所不希望发生的事情；这里抛出的异常通常都是致命的，也许是资源耗尽，或者是你handler的其中一个抛出了异常。另外，每个方法都有一个不抛出异常，而是返回一个*boost::system::error_code*的重载：

```
io_service service;
boost::system::error_code err = 0;
service.run(err);
if ( err) std::cout << "Error " << err << std::endl;
```

#### 异步工作

异步工作不仅仅指用异步地方式接受客户端到服务端的连接、异步地从一个socket读取或者写入到socket。它包含了所有可以异步执行的操作。

默认情况下，你是不知道每个异步handler的调用顺序的。除了通常的异步调用（来自异步socket的读取/写入/接收）。你可以使用*service.post()*来使你的自定义方法被异步地调用。例如：

```
#include <boost/thread.hpp>
#include <boost/bind.hpp>
#include <boost/asio.hpp>
#include <iostream>
using namespace boost::asio;
io_service service;
void func(int i) {
    std::cout << "func called, i= " << i << std::endl;
}

void worker_thread() {
    service.run();
}

int main(int argc, char* argv[]) {
    for ( int i = 0; i < 10; ++i)
        service.post(boost::bind(func, i));
    boost::thread_group threads;
    for ( int i = 0; i < 3; ++i)
        threads.create_thread(worker_thread);
    // 等待所有线程被创建完
    boost::this_thread::sleep( boost::posix_time::millisec(500));
    threads.join_all();
}
```

在上面的例子中，*service.post(some_function)*添加了一个异步方法调用。

这个方法在某一个调用了*service.run()*的线程中请求*io_service*实例，然后调用给定的*some_funtion*之后立即返回。在我们的例子中，这个线程是我们之前创建的三个线程中的一个。你不能确定异步方法调用的顺序。你不要期待它们会以我们调用*post()*方法的顺序来调用。下面是运行之前代码可能得到的结果：

```
func called, i= 0
func called, i= 2
func called, i= 1
func called, i= 4
func called, i= 3
func called, i= 6
func called, i= 7
func called, i= 8
func called, i= 5
func called, i= 9
```

有时候你会想让一些异步处理方法顺序执行。比如，你去一个餐馆（*go_to_restaurant*），下单（*order*），然后吃（*eat*）。你需要先去餐馆，然后下单，最后吃。这样的话，你需要用到*io_service::strand*，这个方法会让你的异步方法被顺序调用。看下面的例子：

```
using namespace boost::asio;
io_service service;
void func(int i) {
    std::cout << "func called, i= " << i << "/" << boost::this_thread::get_id() << std::endl;
}
void worker_thread() {
    service.run();
}
int main(int argc, char* argv[])
{
    io_service::strand strand_one(service), strand_two(service);
    for ( int i = 0; i < 5; ++i)
        service.post( strand_one.wrap( boost::bind(func, i)));
    for ( int i = 5; i < 10; ++i)
        service.post( strand_two.wrap( boost::bind(func, i)));
    boost::thread_group threads;
    for ( int i = 0; i < 3; ++i)
        threads.create_thread(worker_thread);
    // 等待所有线程被创建完
    boost::this_thread::sleep( boost::posix_time::millisec(500));
    threads.join_all();
}
```

在上述代码中，我们保证前面的5个线程和后面的5个线程是顺序执行的。*func called, i = 0*在*func called, i = 1*之前被调用，然后调用*func called, i = 2*……同样*func  called, i = 5*在*func called, i = 6*之前，*func called, i = 6*在*func called, i = 7*被调用……你需要注意的是尽管方法是顺序调用的，但是不意味着它们都在同一个线程执行。运行这个程序可能得到的一个结果如下：

```
func called, i= 0/002A60C8
func called, i= 5/002A6138
func called, i= 6/002A6530
func called, i= 1/002A6138
func called, i= 7/002A6530
func called, i= 2/002A6138
func called, i= 8/002A6530
func called, i= 3/002A6138
func called, i= 9/002A6530
func called, i= 4/002A6138
```

#### 异步post() VS dispatch() VS wrap()

Boost.Asio提供了三种让你把处理方法添加为异步调用的方式：

* *service.post(handler)*：这个方法能确保其在请求*io_service*实例，然后调用指定的处理方法之后立即返回。handler稍后会在某个调用了*service.run()*的线程中被调用。
* *service.dispatch(handler)*：这个方法请求*io_service*实例去调用给定的处理方法，但是另外一点，如果当前的线程调用了*service.run()*，它可以在方法中直接调用handler。
* *service.wrap(handler)*：这个方法创建了一个封装方法，当被调用时它会调用*service.dispatch(handler)*，这个会让人有点困惑，接下来我会简单地解释它是什么意思。

在之前的章节中你会看到关于*service.post()*的一个例子，以及运行这个例子可能得到的一种结果。我们对它做一些修改，然后看看*service.dispatch()*是怎么影响输出的结果的：

```
using namespace boost::asio;
io_service service;
void func(int i) {
    std::cout << "func called, i= " << i << std::endl;
}
void run_dispatch_and_post() {
    for ( int i = 0; i < 10; i += 2) {
    service.dispatch(boost::bind(func, i));
    service.post(boost::bind(func, i + 1));
    }
}
int main(int argc, char* argv[]) {
    service.post(run_dispatch_and_post);
    service.run();
}
```

在解释发生了什么之前，我们先运行程序，观察结果：

```
func called, i= 0
func called, i= 2
func called, i= 4
func called, i= 6
func called, i= 8
func called, i= 1
func called, i= 3
func called, i= 5
func called, i= 7
func called, i= 9
```

偶数先输出，然后是奇数。这是因为我用*dispatch()*输出偶数，然后用*post()*输出奇数。*dispatch()*会在返回之前调用hanlder，因为当前的线程调用了*service.run()*，而*post()*每次都立即返回了。

现在，让我们讲讲*service.wrap(handler)*。*wrap()*返回了一个仿函数，它可以用来做另外一个方法的参数：

```
using namespace boost::asio;
io_service service;
void dispatched_func_1() {
    std::cout << "dispatched 1" << std::endl;
}
void dispatched_func_2() {
    std::cout << "dispatched 2" << std::endl;
}
void test(boost::function<void()> func) {
    std::cout << "test" << std::endl;
    service.dispatch(dispatched_func_1);
    func();
}
void service_run() {
    service.run();
}
int main(int argc, char* argv[]) {
    test( service.wrap(dispatched_func_2));
    boost::thread th(service_run);
    boost::this_thread::sleep( boost::posix_time::millisec(500));
    th.join();
}
```

*test(service.wrap(dispatched_func_2));*会把*dispatched_ func_2*包装起来创建一个仿函数，然后传递给*test*当作一个参数。当*test()*被调用时，它会分发调用方法1，然后调用*func()*。这时，你会发现调用*func()*和*service.dispatch(dispatched_func_2)*是等价的，因为它们是连续调用的。程序的输出证明了这一点：
```
test
dispatched 1
dispatched 2
```
*io_service::strand *类（用来序列化异步调用）也包含了*poll(), dispatch()*和 *wrap()*等成员函数。它们的作用和*io_service*的*poll(), dispatch()*和*wrap()*是一样的。然而，大多数情况下你只需要把*io_service::strand::wrap()*方法做为*io_service::poll()*或者*io_service::dispatch()*方法的参数即可。
### 保持活动
假设你需要做下面的操作：
```
io_service service;
ip::tcp::socket sock(service);
char buff[512];
...
read(sock, buffer(buff));
```

在这个例子中，*sock*和*buff*的存在时间都必须比*read()*调用的时间要长。也就是说，在调用*read()*返回之前，它们都必须有效。这就是你所期望的；你传给一个方法的所有参数在方法内部都必须有效。当我们采用异步方式时，事情会变得比较复杂。
```
io_service service;
ip::tcp::socket sock(service);
char buff[512];
void on_read(const boost::system::error_code &, size_t) {}
...
async_read(sock, buffer(buff), on_read);
```

在这个例子中，*sock*和*buff*的存在时间都必须比*read()*操作本身时间要长，但是read操作持续的时间我们是不知道的，因为它是异步的。

当使用socket缓冲区的时候，你会有一个*buffer*实例在异步调用时一直存在（使用*boost::shared_array<>*）。在这里，我们可以使用同样的方式，通过创建一个类并在其内部管理socket和它的读写缓冲区。然后，对于所有的异步操作，传递一个包含智能指针的*boost::bind*仿函数给它：
```
using namespace boost::asio;
io_service service;
struct connection : boost::enable_shared_from_this<connection> {
    typedef boost::system::error_code error_code;
    typedef boost::shared_ptr<connection> ptr;
    connection() : sock_(service), started_(true) {}
    void start(ip::tcp::endpoint ep) {
        sock_.async_connect(ep, boost::bind(&connection::on_connect, shared_from_this(), _1));
    }
    void stop() {
        if ( !started_) return;
        started_ = false;
        sock_.close();
    }
    bool started() { return started_; }
private:
    void on_connect(const error_code & err) {
        // 这里你决定用这个连接做什么: 读取或者写入
        if ( !err) do_read();
        else stop();
    }
    void on_read(const error_code & err, size_t bytes) {
        if ( !started() ) return;
        std::string msg(read_buffer_, bytes);
        if ( msg == "can_login") do_write("access_data");
        else if ( msg.find("data ") == 0) process_data(msg);
        else if ( msg == "login_fail") stop();
    }
    void on_write(const error_code & err, size_t bytes) {
        do_read(); 
    }
    void do_read() {
        sock_.async_read_some(buffer(read_buffer_), boost::bind(&connection::on_read, shared_from_this(),   _1, _2)); 
    }
    void do_write(const std::string & msg) {
        if ( !started() ) return;
        // 注意: 因为在做另外一个async_read操作之前你想要发送多个消息, 
        // 所以你需要多个写入buffer
        std::copy(msg.begin(), msg.end(), write_buffer_);
        sock_.async_write_some(buffer(write_buffer_, msg.size()), boost::bind(&connection::on_write, shared_from_this(), _1, _2)); 
    }

    void process_data(const std::string & msg) {
        // 处理服务端来的内容，然后启动另外一个写入操作
    }
private:
    ip::tcp::socket sock_;
    enum { max_msg = 1024 };
    char read_buffer_[max_msg];
    char write_buffer_[max_msg];
    bool started_;
};

int main(int argc, char* argv[]) {
    ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 8001);
    connection::ptr(new connection)->start(ep);
} 
```
在所有异步调用中，我们传递一个*boost::bind*仿函数当作参数。这个仿函数内部包含了一个智能指针，指向*connection*实例。只要有一个异步操作等待时，Boost.Asio就会保存*boost::bind*仿函数的拷贝，这个拷贝保存了指向连接实例的一个智能指针，从而保证*connection*实例保持活动。问题解决！

当然，*connection*类仅仅是一个框架类；你需要根据你的需求对它进行调整（它看起来会和当前服务端例子的情况相当不同）。

你需要注意的是创建一个新的连接是相当简单的：*connection::ptr(new connection)- >start(ep)*。这个方法启动了到服务端的（异步）连接。当你需要关闭这个连接时，调用*stop()*。

当实例被启动时（*start()*），它会等待客户端的连接。当连接发生时。*on_connect()*被调用。如果没有错误发生，它启动一个read操作（*do_read()*）。当read操作结束时，你就可以解析这个消息；当然你应用的*on_read()*看起来会各种各样。而当你写回一个消息时，你需要把它拷贝到缓冲区，然后像我在*do_write()*方法中所做的一样将其发送出去，因为这个缓冲区同样需要在这个异步写操作中一直存活。最后需要注意的一点——当写回时，你需要指定写入的数量，否则，整个缓冲区都会被发送出去。

### 总结

网络api实际上要繁杂得多，这个章节只是做为一个参考，当你在实现自己的网络应用时可以回过头来看看。

Boost.Asio实现了端点的概念，你可以认为是IP和端口。如果你不知道准确的IP，你可以使用*resolver*对象将主机名，例如*www.yahoo.com*转换为一个或多个IP地址。

我们也可以看到API的核心——socket类。Boost.Asio提供了*TCP、UDP*和 *ICMP*的实现。而且你还可以用你自己的协议来对它进行扩展；当然，这个工作不适合缺乏勇气的人。

异步编程是刚需。你应该已经明白为什么有时候需要用到它，尤其在写服务端的时候。调用*service.run()*来实现异步循环就已经可以让你很满足，但是有时候你需要更进一步，尝试使用*run_one()、poll()*或者*poll_one()*。

当实现异步时，你可以异步执行你自己的方法；使用*service.post()*或者*service.dispatch()*。

最后，为了使socket和缓冲区（read或者write）在整个异步操作的生命周期中一直活动，我们需要采取特殊的防护措施。你的连接类需要继承自*enabled_shared_from_this*，然后在内部保存它需要的缓冲区，而且每次异步调用都要传递一个智能指针给*this*操作。

下一章会进行实战操作；在实现回显客户端/服务端应用时会有大量的编程实践。
