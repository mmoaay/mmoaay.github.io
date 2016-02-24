---
layout: post
title: "客户端和服务端"
description: "Boost.Asio C++ Network Programming 翻译"

category: Boost.Asio
tags: [Boost.Asio,翻译]
modified: 2016-02-24

imagefeature: mmoaay_bg.jpg
comments: true
share: true
---

## 回显服务端/客户端

在这一章，我们将会实现一个小的客户端/服务端应用，这可能会是你写过的最简单的客户端/服务端应用。回显应用就是一个把客户端发过来的任何内容回显给其本身，然后关闭连接的的服务端。这个服务端可以处理任何数量的客户端。每个客户端连接之后发送一个消息，服务端接收到完成消息后把它发送回去。在那之后，服务端关闭连接。

<!-- more -->

因此，每个回显客户端连接到服务端，发送一个消息，然后读取服务端返回的结果，确保这是它发送给服务端的消息就结束和服务端的会话。

我们首先实现一个同步应用，然后实现一个异步应用，以便你可以很容易对比他们：

![](http://d.pcs.baidu.com/thumbnail/3ed527792035c0abc0d8e70405180310?fid=3238002958-250528-552015406596888&time=1420768800&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-jmDoVsCZ6Qw1kecSlrmm13%2BuoY0%3D&rt=sh&expires=2h&r=776548562&sharesign=unknown&size=c710_u500&quality=100)

为了节省空间，下面的代码有一些被裁剪掉了。你可以在附加在这本书的代码中看到全部的代码。

### TCP回显服务端/客户端

对于TCP而言，我们需要一个额外的保证；每一个消息以换行符结束(‘\n’)。编写一个同步回显服务端/客户端非常简单。

我们会展示编码内容，比如同步客户端，同步服务端，异步客户端和异步服务端。

#### TCP同步客户端

在大多数有价值的例子中，客户端通常比服务端编码要简单（因为服务端需要处理多个客户端请求）。

下面的代码展示了不符合这条规则的一个例外：

```
size_t read_complete(char * buf, const error_code & err, size_t bytes)
{
    if ( err) return 0;
    bool found = std::find(buf, buf + bytes, '\n') < buf + bytes;
    // 我们一个一个读取直到读到回车，不缓存
    return found ? 0 : 1;
}
void sync_echo(std::string msg) {
    msg += "\n”;
    ip::tcp::socket sock(service);
    sock.connect(ep);
    sock.write_some(buffer(msg));
    char buf[1024];
    int bytes = read(sock, buffer(buf), boost::bind(read_complete,buf,_1,_2));
    std::string copy(buf, bytes - 1);
    msg = msg.substr(0, msg.size() - 1);
    std::cout << "server echoed our " << msg << ": "<< (copy == msg ? "OK" : "FAIL") << std::endl;
    sock.close();
}
int main(int argc, char* argv[]) {
    char* messages[] = { "John says hi", "so does James", "Lucy just got home", "Boost.Asio is Fun!", 0 };
    boost::thread_group threads;
    for ( char ** message = messages; *message; ++message) {
        threads.create_thread( boost::bind(sync_echo, *message));
        boost::this_thread::sleep( boost::posix_time::millisec(100));
    }
    threads.join_all();
}
```

核心功能*sync_echo*。它包含了连接到服务端，发送信息然后等待回显的所有逻辑。

你会发现，在读取时，我使用了自由函数*read()*，因为我想要读’\n’之前的所有内容。*sock.read_some()*方法满足不了这个要求，因为它只会读可用的，而不是全部的消息。

*read()*方法的第三个参数是完成处理句柄。当读取到完整消息时，它返回0。否则，它会返回我下一步（直到读取结束）能都到的最大的缓冲区大小。在我们的例子中，返回结果始终是1，因为我永远不想读的消息比我们需要的更多。

在*main()*中，我们创建了几个线程；每个线程负责把消息发送到客户端，然后等待操作结束。如果你运行这个程序，你会看到下面的输出：

```
server echoed our John says hi: OK
server echoed our so does James: OK
server echoed our Lucy just got home: OK
server echoed our Boost.Asio is Fun!: OK
```

注意：因为我们是同步的，所以不需要调用*service.run()*。

#### TCP同步服务端

回显同步服务端的编写非常容易，参考如下的代码片段：

```
io_service service;
size_t read_complete(char * buff, const error_code & err, size_t bytes) {
    if ( err) return 0;
    bool found = std::find(buff, buff + bytes, '\n') < buff + bytes;
    // 我们一个一个读取直到读到回车，不缓存
    return found ? 0 : 1;
}
void handle_connections() {
    ip::tcp::acceptor acceptor(service, ip::tcp::endpoint(ip::tcp::v4(),8001));
    char buff[1024];
    while ( true) {
        ip::tcp::socket sock(service);
        acceptor.accept(sock);
        int bytes = read(sock, buffer(buff), boost::bind(read_complete,buff,_1,_2));
        std::string msg(buff, bytes);
        sock.write_some(buffer(msg));
        sock.close();
    }
}
int main(int argc, char* argv[]) {
    handle_connections();
}
```

服务端的逻辑主要在*handle_connections()*。因为是单线程，我们接受一个客户端请求，读取它发送给我们的消息，然后回显，然后等待下一个连接。可以确定，当两个客户端同时连接时，第二个客户端需要等待服务端处理完第一个客户端的请求。

还是要注意因为我们是同步的，所以不需要调用*service.run()*。

#### TCP异步客户端

当我们开始异步时，编码会变得稍微有点复杂。我们会构建在**第二章 保持活动**中展示的*connection*类。

观察这个章节中接下来的代码，你会发现每个异步操作启动了新的异步操作，以保持*service.run()*一直工作。

首先，核心功能如下：

```
#define MEM_FN(x)       boost::bind(&self_type::x, shared_from_this())
#define MEM_FN1(x,y)    boost::bind(&self_type::x, shared_from_this(),y)
#define MEM_FN2(x,y,z)  boost::bind(&self_type::x, shared_from_this(),y,z)
class talk_to_svr : public boost::enable_shared_from_this<talk_to_svr> , boost::noncopyable {
    typedef talk_to_svr self_type;
    talk_to_svr(const std::string & message) : sock_(service), started_(true), message_(message) {}
    void start(ip::tcp::endpoint ep) {
        sock_.async_connect(ep, MEM_FN1(on_connect,_1));
    }
public:
    typedef boost::system::error_code error_code;
    typedef boost::shared_ptr<talk_to_svr> ptr;
    static ptr start(ip::tcp::endpoint ep, const std::string &message) {
        ptr new_(new talk_to_svr(message));
        new_->start(ep);
        return new_;
    }
    void stop() {
        if ( !started_) return;
        started_ = false;
        sock_.close();
    }
    bool started() { return started_; }
    ...
private:
    ip::tcp::socket sock_;
    enum { max_msg = 1024 };
    char read_buffer_[max_msg];
    char write_buffer_[max_msg];
    bool started_;
    std::string message_; 
}; 
```

我们需要一直使用指向*talk_to_svr*的智能指针，这样的话当在*tack_to_svr*的实例上有异步操作时，那个实例是一直活动的。为了避免错误，比如在栈上构建一个*talk_to_svr*对象的实例时，我把构造方法设置成了私有而且不允许拷贝构造（继承自*boost::noncopyable*）。

我们有了核心方法，比如*start(),stop()*和*started()*，它们所做的事情也正如它们名字表达的一样。如果需要建立连接，调用*talk_to_svr::start(endpoint, message)*即可。我们同时还有一个read缓冲区和一个write缓冲区。（*read_buufer_*和*write_buffer_*）。

*MEM_FN* *是一个方便使用的宏，它们通过*shared_ptr_from_this()*方法强制使用一个指向* *this *的智能指针。

下面的几行代码和之前的解释非常不同：

```
//等同于 "sock_.async_connect(ep, MEM_FN1(on_connect,_1));"
sock_.async_connect(ep,boost::bind(&talk_to_svr::on_connect,shared_ptr_from_this(),_1));
sock_.async_connect(ep, boost::bind(&talk_to_svr::on_connect,this,_1));
```

在上述例子中，我们正确的创建了*async_connect*的完成处理句柄；在调用完成处理句柄之前它会保留一个指向*talk_to_server*实例的智能指针，从而保证当其发生时*talk_to_server*实例还是保持活动的。

在接下来的例子中，我们错误地创建了完成处理句柄，当它被调用时，*talk_to_server*实例很可能已经被释放了。

从socket读取或写入时，你使用如下的代码片段：

```
void do_read() {
    async_read(sock_, buffer(read_buffer_), MEM_FN2(read_complete,_1,_2), MEM_FN2(on_read,_1,_2));
}
void do_write(const std::string & msg) {
    if ( !started() ) return;
    std::copy(msg.begin(), msg.end(), write_buffer_);
    sock_.async_write_some( buffer(write_buffer_, msg.size()), MEM_FN2(on_write,_1,_2));
}
size_t read_complete(const boost::system::error_code & err, size_t bytes) {
    // 和TCP客户端中的类似
}
```

*do_read()*方法会保证当*on_read()*被调用的时候，我们从服务端读取一行。*do_write()*方法会先把信息拷贝到缓冲区（考虑到当*async_write*发生时msg可能已经超出范围被释放），然后保证实际的写入操作发生时*on_write()*被调用。

然后是最重要的方法，这个方法包含了类的主要逻辑：

```
void on_connect(const error_code & err) {
    if ( !err)      do_write(message_ + "\n");
    else            stop();
}
void on_read(const error_code & err, size_t bytes) {
    if ( !err) {
        std::string copy(read_buffer_, bytes - 1);
        std::cout << "server echoed our " << message_ << ": " << (copy == message_ ? "OK" : "FAIL") << std::endl; 
    }
    stop(); 
}
void on_write(const error_code & err, size_t bytes) {
    do_read();
} 
```

当连接成功之后，我们发送消息到服务端,*do_write()*。当write操作结束时，*on_write()*被调用，它初始化了一个*do_read()*方法，当*do_read()*完成时。*on_read()*被调用；这里，我们简单的检查一下返回的信息是否是服务端的回显，然后退出服务。

我们会发送三个消息到服务端让它变得更有趣一点：

```
int main(int argc, char* argv[]) {
    ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 8001);
    char* messages[] = { "John says hi", "so does James", "Lucy got home", 0 };
    for ( char ** message = messages; *message; ++message) {
        talk_to_svr::start( ep, *message);
        boost::this_thread::sleep( boost::posix_time::millisec(100));
    }
    service.run();
}
```

上述的代码会生成如下的输出：

```
server echoed our John says hi: OK
server echoed our so does James: OK
server echoed our Lucy just got home: OK
```

#### TCP异步服务端

核心功能和同步服务端的功能类似，如下：

```
class talk_to_client : public boost::enable_shared_from_this<talk_to_
   client>, boost::noncopyable {
    typedef talk_to_client self_type;
    talk_to_client() : sock_(service), started_(false) {}
public:
    typedef boost::system::error_code error_code;
    typedef boost::shared_ptr<talk_to_client> ptr;
    void start() {
        started_ = true;
        do_read(); 
    }

    static ptr new_() {
        ptr new_(new talk_to_client);
        return new_;
    }
    void stop() {
        if ( !started_) return;
        started_ = false;
        sock_.close();
    }
    ip::tcp::socket & sock() { return sock_;}
    ...
private:
    ip::tcp::socket sock_;
    enum { max_msg = 1024 };
    char read_buffer_[max_msg];
    char write_buffer_[max_msg];
    bool started_;
};
```

因为我们是非常简单的回显服务，这里不需要*is_started()*方法。对每个客户端，仅仅读取它的消息，回显，然后关闭它。

*do_read()，do_write()*和*read_complete()*方法和TCP同步服务端的完全一致。

主要的逻辑同样是在*on_read()*和*on_write()*方法中：

```
void on_read(const error_code & err, size_t bytes) {
    if ( !err) {
        std::string msg(read_buffer_, bytes);
        do_write(msg + "\n");
    }
    stop(); 
}
void on_write(const error_code & err, size_t bytes) {
    do_read();
}
```

对客户端的处理如下：

```
ip::tcp::acceptor acceptor(service, ip::tcp::endpoint(ip::tcp::v4(),8001));
void handle_accept(talk_to_client::ptr client, const error_code & err)
{
    client->start();
    talk_to_client::ptr new_client = talk_to_client::new_();
    acceptor.async_accept(new_client->sock(), boost::bind(handle_accept,new_client,_1));
}
int main(int argc, char* argv[]) {
    talk_to_client::ptr client = talk_to_client::new_();
    acceptor.async_accept(client->sock(), boost::bind(handle_accept,client,_1));
    service.run();
} 
```

每一次客户端连接到服务时，*handle_accept*被调用，它会异步地从客户端读取，然后同样异步地等待一个新的客户端。

#### 代码

你会在这本书相应的代码中得到所有4个应用（TCP回显同步客户端，TCP回显同步服务端，TCP回显异步客户端，TCP回显异步服务端）。当测试时，你可以使用任意客户端/服务端组合（比如，一个异步客户端和一个同步服务端）。

### UDP回显服务端/客户端

因为UDP不能保证所有信息都抵达接收者，我们不能保证“信息以回车结尾”。
没收到消息，我们只是回显，但是没有socket去关闭（在服务端），因为我们是UDP。

#### UDP同步回显客户端

UDP回显客户端比TCP回显客户端要简单：

```
ip::udp::endpoint ep( ip::address::from_string("127.0.0.1"), 8001);
void sync_echo(std::string msg) {
    ip::udp::socket sock(service, ip::udp::endpoint(ip::udp::v4(), 0));
    sock.send_to(buffer(msg), ep);
    char buff[1024];
    ip::udp::endpoint sender_ep;
    int bytes = sock.receive_from(buffer(buff), sender_ep);
    std::string copy(buff, bytes);
    std::cout << "server echoed our " << msg << ": " << (copy == msg ? "OK" : "FAIL") << std::endl;
    sock.close();
}
int main(int argc, char* argv[]) {
    char* messages[] = { "John says hi", "so does James", "Lucy got home", 0 };
    boost::thread_group threads;
    for ( char ** message = messages; *message; ++message) {
        threads.create_thread( boost::bind(sync_echo, *message));
        boost::this_thread::sleep( boost::posix_time::millisec(100));
    }
    threads.join_all();
}
```

所有的逻辑都在*synch_echo()*中；连接到服务端，发送消息，接收服务端的回显，然后关闭连接。

#### UDP同步回显服务端

UDP回显服务端会是你写过的最简单的服务端：

```
io_service service;
void handle_connections() {
    char buff[1024];
    ip::udp::socket sock(service, ip::udp::endpoint(ip::udp::v4(), 8001));
    while ( true) {
        ip::udp::endpoint sender_ep;
        int bytes = sock.receive_from(buffer(buff), sender_ep);
        std::string msg(buff, bytes);
        sock.send_to(buffer(msg), sender_ep);
    } 
}
int main(int argc, char* argv[]) {
    handle_connections();
} 
```

它非常简单，而且能很好的自释。

我把异步UDP客户端和服务端留给读者当作一个练习。

### 总结

我们已经写了完整的应用，最终让Boost.Asio得以工作。回显应用是开始学习一个库时非常好的工具。你可以经常学习和运行这个章节所展示的代码，这样你就可以非常容易地记住这个库的基础。
在下一章，我们会建立更复杂的客户端/服务端应用，我们要确保避免低级错误，比如内存泄漏，死锁等等。

