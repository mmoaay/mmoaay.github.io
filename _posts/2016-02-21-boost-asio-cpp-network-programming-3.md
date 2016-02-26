---
layout: post
title: "客户端和服务端"
description: "Boost.Asio C++ Network Programming 翻译"

category: Boost.Asio
tags: [Boost.Asio,翻译]

imagefeature: mmoaay_bg.jpg
comments: true
share: true
---

在这一章节，我们会深入学习怎样使用Boost.Asio建立非凡的客户端和服务端应用。你可以运行并测试它们，而且在理解之后，你可以把它们做为框架来构造自己的应用。

在接下来的例子中：

* 客户端使用一个用户名（无密码）登录到服务端
* 所有的连接由客户端建立，当客户端请求时服务端回应
* 所有的请求和回复都以换行符结尾（’\n’）
* 对于5秒钟没有ping操作的客户端，服务端会自动断开其连接

客户端可以发送如下请求：

* 获得所有已连接客户端的列表
* 客户端可以ping，当它ping时，服务端返回*ping ok*或者*ping client_list_chaned*（在接下来的例子中，客户端重新请求已连接的客户端列表）

为了更有趣一点，我们增加了一些难度：

* 每个客户端登录6个用户连接，比如Johon,James,Lucy,Tracy,Frank和Abby
* 每个客户端连接随机地ping服务端（随机7秒；这样的话，服务端会时不时关闭一个连接）

### 同步客户端/服务端

首先，我们会实现同步应用。你会发现它的代码很直接而且易读的。而且因为所有的网络调用都是阻塞的，所以它不需要独立的线程。

#### 同步客户端

同步客户端会以你所期望的串行方式运行；连接到服务端，登录服务器，然后执行连接循环，比如休眠一下，发起一个请求，读取服务端返回，然后再休眠一会，然后一直循环下去……

![](http://d.pcs.baidu.com/thumbnail/102a243f8953a60d8531f3c68699e517?fid=3238002958-250528-439846994747753&time=1420768800&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-bHTbbqvG4PXZKcDNFdOWL6iVPVY%3D&rt=sh&expires=2h&r=382985944&sharesign=unknown&size=c710_u500&quality=100)

因为我们是同步的，所以我们让事情变得简单一点。首先，连接到服务器，然后再循环，如下：

```
ip::tcp::endpoint ep( ip::address::from_string("127.0.0.1"), 8001);
void run_client(const std::string & client_name) {
    talk_to_svr client(client_name);
    try {
        client.connect(ep);
        client.loop();
    } catch(boost::system::system_error & err) {
        std::cout << "client terminated " << std::endl;
    }
}
```

下面的代码片段展示了talk_to_svr类：

```
struct talk_to_svr {
    talk_to_svr(const std::string & username) : sock_(service), started_(true), username_(username) {}
    void connect(ip::tcp::endpoint ep) {
        sock_.connect(ep);
    }
    void loop() {
        write("login " + username_ + "\n");
        read_answer();
        while ( started_) {
            write_request();
            read_answer();
            boost::this_thread::sleep(millisec(rand() % 7000));
        }
    }

    std::string username() const { return username_; }
    ... 
private:
    ip::tcp::socket sock_;
    enum { max_msg = 1024 };
    int already_read_;
    char buff_[max_msg];
    bool started_;
    std::string username_;
}; 
```

在这个循环中，我们仅仅填充1个比特，做一个ping操作之后就进入睡眠状态，之后再读取服务端的返回。我们的睡眠是随机的（有时候超过5秒），这样服务端就有可能在某个时间点断开我们的连接：

```
void write_request() {
    write("ping\n");
}
void read_answer() {
    already_read_ = 0;
    read(sock_, buffer(buff_), boost::bind(&talk_to_svr::read_complete, this, _1, _2));
    process_msg();
}
void process_msg() {
    std::string msg(buff_, already_read_);
    if ( msg.find("login ") == 0) on_login();
    else if ( msg.find("ping") == 0) on_ping(msg);
    else if ( msg.find("clients ") == 0) on_clients(msg);
    else std::cerr << "invalid msg " << msg << std::endl;
} 
```

对于读取结果，我们使用在之前章节就有说到的*read_complete*来保证我们能读到换行符（’\n’）。这段逻辑在*process_msg()*中，在这里我们读取服务端的返回，然后分发到正确的方法去处理：

```
void on_login() { do_ask_clients(); }
void on_ping(const std::string & msg) {
    std::istringstream in(msg);
    std::string answer;
    in >> answer >> answer;
    if ( answer == "client_list_changed")
        do_ask_clients();
}
void on_clients(const std::string & msg) {
    std::string clients = msg.substr(8);
    std::cout << username_ << ", new client list:" << clients;
}
void do_ask_clients() {
    write("ask_clients\n");
    read_answer();
}
void write(const std::string & msg) { sock_.write_some(buffer(msg)); }
size_t read_complete(const boost::system::error_code & err, size_t bytes) {
    // ... 和之前一样
}
```

在读取服务端对我们ping操作的返回时，如果得到的消息是*client_list_changed*，我们就需要重新请求客户端列表。

#### 同步服务端

同步服务端也是相当简单的。它只需要两个线程，一个负责接收新的客户端连接，另外一个负责处理已经存在的客户端请求。它不能使用单线程，因为等待新的客户端连接是一个阻塞操作，所以我们需要另外一个线程来处理已经存在的客户端请求。

![](http://d.pcs.baidu.com/thumbnail/ceff46cf09767285fd5fa58be5d5beae?fid=3238002958-250528-625961824567867&time=1420768800&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-jZ%2BsgWwaNZblSuxpNAYBN2BuXP8%3D&rt=sh&expires=2h&r=387068104&sharesign=unknown&size=c710_u500&quality=100)

正常来说服务端都比客户端要难实现。一方面，它要管理所有已经连接的客户端。因为我们是同步的，所以我们需要至少两个线程，一个负责接受新的客户端连接（因为accept()是阻塞的）而另一个负责回复已经存在的客户端。

```
void accept_thread() {
    ip::tcp::acceptor acceptor(service,ip::tcp::endpoint(ip::tcp::v4(), 8001));
    while ( true) {
        client_ptr new_( new talk_to_client);
        acceptor.accept(new_->sock());
        boost::recursive_mutex::scoped_lock lk(cs);
        clients.push_back(new_);
    }
}

void handle_clients_thread() {
    while ( true) {
        boost::this_thread::sleep( millisec(1));
        boost::recursive_mutex::scoped_lock lk(cs);
        for(array::iterator b = clients.begin(), e = clients.end(); b!= e; ++b)
            (*b)->answer_to_client();
        // 删除已经超时的客户端
        clients.erase(std::remove_if(clients.begin(), clients.end(), boost::bind(&talk_to_client::timed_out,_1)), clients.end());
    }
}
int main(int argc, char* argv[]) {
    boost::thread_group threads;
    threads.create_thread(accept_thread);
    threads.create_thread(handle_clients_thread);
    threads.join_all();
} 
```

为了分辨客户端发送过来的请求我们需要保存一个客户端的列表。

每个*talk_to_client*实例都拥有一个socket，socket类是不支持拷贝构造的，所以如果你想要把它们保存在一个*std::vector*对象中，你需要一个指向它的智能指针。这里有两种实现的方式：在*talk_to_client*内部保存一个指向socket的智能指针然后创建一个*talk_to_client*实例的数组，或者让*talk_to_client*实例用变量的方式保存socket，然后创建一个指向*talk_to_client*智能指针的数组。我选择后者，但是你也可以选前面的方式：

```
typedef boost::shared_ptr<talk_to_client> client_ptr;
typedef std::vector<client_ptr> array;
array clients;
boost::recursive_mutex cs; // 用线程安全的方式访问客户端数组
```

*talk_to_client*的主要代码如下：

```
struct talk_to_client : boost::enable_shared_from_this<talk_to_client>
{
    talk_to_client() { ... }
    std::string username() const { return username_; }
    void answer_to_client() {
        try {
            read_request();
            process_request();
        } catch ( boost::system::system_error&) { stop(); }
        if ( timed_out())
            stop();
    }
    void set_clients_changed() { clients_changed_ = true; }
    ip::tcp::socket & sock() { return sock_; }
    bool timed_out() const {
        ptime now = microsec_clock::local_time();
        long long ms = (now - last_ping).total_milliseconds();
        return ms > 5000 ;
    }
    void stop() {
        boost::system::error_code err; sock_.close(err);
    }
    void read_request() {
        if ( sock_.available())
            already_read_ += sock_.read_some(buffer(buff_ + already_read_, max_msg - already_read_));
    }
... 
private:
    // ...  和同步客户端中的一样
    bool clients_changed_;
    ptime last_ping;
}; 
```


上述代码拥有非常好的自释能力。其中最重要的方法是*read_request()*。它只在存在有效数据的情况才读取，这样的话，服务端永远都不会阻塞：

```
void process_request() {
    bool found_enter = std::find(buff_, buff_ + already_read_, '\n') < buff_ + already_read_;
    if ( !found_enter)
        return; // 消息不完整
        // 处理消息
    last_ping = microsec_clock::local_time();
    size_t pos = std::find(buff_, buff_ + already_read_, '\n') - buff_;
    std::string msg(buff_, pos);
    std::copy(buff_ + already_read_, buff_ + max_msg, buff_);
    already_read_ -= pos + 1;
    if ( msg.find("login ") == 0) on_login(msg);
    else if ( msg.find("ping") == 0) on_ping();
    else if ( msg.find("ask_clients") == 0) on_clients();
    else std::cerr << "invalid msg " << msg << std::endl;
}
void on_login(const std::string & msg) {
    std::istringstream in(msg);
    in >> username_ >> username_;
    write("login ok\n");
    update_clients_changed();
} 
void on_ping() {
    write(clients_changed_ ? "ping client_list_changed\n" : "ping ok\n");
    clients_changed_ = false;
}
void on_clients() {
    std::string msg;
    { boost::recursive_mutex::scoped_lock lk(cs);
        for( array::const_iterator b = clients.begin(), e = clients.end() ; b != e; ++b)
            msg += (*b)->username() + " ";
    }
    write("clients " + msg + "\n");
}
void write(const std::string & msg){sock_.write_some(buffer(msg)); }
```

观察*process_request()*。当我们读取到足够多有效的数据时，我们需要知道我们是否已经读取到整个消息(如果*found_enter*为真)。这样做的话，我们可以使我们避免一次读多个消息的可能（’\n’之后的消息也被保存到缓冲区中），然后我们解析读取到的整个消息。剩下的代码都是很容易读懂的。

### 异步客户端/服务端

现在，是比较有趣（也比较难）的异步实现！

当查看示意图时，你需要知道Boost.Asio代表由Boost.Asio执行的一个异步调用。例如*do_read()*，Boost.Asio和*on_read()*代表了从*do_read()*到*on_read()*的逻辑流程，但是你永远不知道什么时候轮到*on_read()*被调用，你只是知道你最终会调用它。

#### 异步客户端

到这里事情会变得有点复杂，但是仍然是可控的。当然你也会拥有一个不会阻塞的应用。

![](http://d.pcs.baidu.com/thumbnail/953e9b90f743389e6ea7a425aaeda307?fid=3238002958-250528-223058845569586&time=1420768800&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-56mkJ9mIQ8E81BqrXlmWpBYEqJU%3D&rt=sh&expires=2h&r=552891765&sharesign=unknown&size=c710_u500&quality=100)

下面的代码你应该已经很熟悉：

```
#define MEM_FN(x)       boost::bind(&self_type::x, shared_from_this())
#define MEM_FN1(x,y)    boost::bind(&self_type::x, shared_from_
this(),y)
#define MEM_FN2(x,y,z)  boost::bind(&self_type::x, shared_from_
this(),y,z)
class talk_to_svr : public boost::enable_shared_from_this<talk_to_svr>, boost::noncopyable {
    typedef talk_to_svr self_type;
    talk_to_svr(const std::string & username) : sock_(service), started_(true), username_(username), timer_
(service) {}
    void start(ip::tcp::endpoint ep) {
        sock_.async_connect(ep, MEM_FN1(on_connect,_1));
} 
public:
    typedef boost::system::error_code error_code;
    typedef boost::shared_ptr<talk_to_svr> ptr;
    static ptr start(ip::tcp::endpoint ep, const std::string & username) {
        ptr new_(new talk_to_svr(username));
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
    size_t read_complete(const boost::system::error_code &err, size_t bytes) {
        if ( err) return 0;
        bool found = std::find(read_buffer_, read_buffer_ + bytes, '\n') < read_buffer_ + bytes;
        return found ? 0 : 1;
    }
private:
    ip::tcp::socket sock_;
    enum { max_msg = 1024 };
    char read_buffer_[max_msg];
    char write_buffer_[max_msg];
    bool started_;
    std::string username_;
    deadline_timer timer_;
};
```

你会看到额外还有一个叫*deadline_timer timer_*的方法用来ping服务端；而且ping操作同样是随机的。

下面是类的逻辑：

```
void on_connect(const error_code & err) {
       if ( !err)      do_write("login " + username_ + "\n");
       else            stop();
   }
void on_read(const error_code & err, size_t bytes) {
    if ( err) stop();
    if ( !started() ) return;
    // 处理消息
    std::string msg(read_buffer_, bytes);
    if ( msg.find("login ") == 0) on_login();
    else if ( msg.find("ping") == 0) on_ping(msg);
    else if ( msg.find("clients ") == 0) on_clients(msg);
}
void on_login() {
    do_ask_clients();
}
void on_ping(const std::string & msg) {
    std::istringstream in(msg);
    std::string answer;
    in >> answer >> answer;
    if ( answer == "client_list_changed") do_ask_clients();
    else postpone_ping();
}
void on_clients(const std::string & msg) {
    std::string clients = msg.substr(8);
    std::cout << username_ << ", new client list:" << clients ;
    postpone_ping();
} 
```

在*on_read()*中，首先的两行代码是亮点。在第一行，如果出现错误，我们就停止。而第二行，如果我们已经停止了（之前就停止了或者刚好停止），我们就返回。反之如果所有都是OK，我们就对收到的消息进行处理。

最后是*do_**方法，实现如下：

```
void do_ping() { do_write("ping\n"); }
void postpone_ping() {
    timer_.expires_from_now(boost::posix_time::millisec(rand() % 7000));
    timer_.async_wait( MEM_FN(do_ping));
}
void do_ask_clients() { do_write("ask_clients\n"); }
void on_write(const error_code & err, size_t bytes) { do_read(); }
void do_read() {
    async_read(sock_, buffer(read_buffer_), MEM_FN2(read_complete,_1,_2), MEM_FN2(on_read,_1,_2));
}
void do_write(const std::string & msg) {
    if ( !started() ) return;
    std::copy(msg.begin(), msg.end(), write_buffer_);
    sock_.async_write_some( buffer(write_buffer_, msg.size()), MEM_FN2(on_write,_1,_2));
```

注意每一个*read*操作都会触发一个ping操作

* 当*read*操作结束时，*on_read()*被调用
* *on_read()*调用*on_login()，on_ping()*或者*on_clients()*
* 每一个方法要么发出一个ping，要么请求客户端列表
* 如果我们请求客户端列表，当*read*操作接收到它们时，它会发出一个ping操作。

#### 异步服务端

这个示意图是相当复杂的；从Boost.Asio出来你可以看到4个箭头指向*on_accept，on_read，on_write*和*on_check_ping*。这也就意味着你永远不知道哪个异步调用是下一个完成的调用，但是你可以确定的是它是这4个操作中的一个。

![](http://d.pcs.baidu.com/thumbnail/eb7c5e88701b3738d5f57cb774af20f9?fid=3238002958-250528-454635957192459&time=1420768800&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-BELSnAVGnjDaCwLdOtTjcybqk%2BY%3D&rt=sh&expires=2h&r=476530834&sharesign=unknown&size=c710_u500&quality=100)

现在，我们是异步的了；我们可以继续保持单线程。接受客户端连接是最简单的部分，如下所示：

```
ip::tcp::acceptor acceptor(service, ip::tcp::endpoint(ip::tcp::v4(), 8001));
void handle_accept(talk_to_client::ptr client, const error_code & err)
{
    client->start();
    talk_to_client::ptr new_client = talk_to_client::new_();
    acceptor.async_accept(new_client->sock(), boost::bind(handle_accept,new_client,_1));
}
int main(int argc, char* argv[]) {
    talk_to_client::ptr client = talk_to_client::new_();
    acceptor.async_accept(client->sock(),boost::bind(handle_accept,client,_1));
    service.run();
}
```

上述代码会一直异步地等待一个新的客户端连接（每个新的客户端连接会触发另外一个异步等待操作）。

我们需要监控*client list changed*事件（一个新客户端连接或者一个客户端断开连接），然后当事件发生时通知所有的客户端。因此，我们需要保存一个客户端连接的数组，否则除非你不需要在某一时刻知道所有连接的客户端，你才不需要这样一个数组。

```
class talk_to_client; 
typedef boost::shared_ptr<talk_to_client>client_ptr;
typedef std::vector<client_ptr> array;
array clients;
```

connection类的框架如下：

```
class talk_to_client : public boost::enable_shared_from_this<talk_to_client> , boost::noncopyable {
    talk_to_client() { ... }
public:
    typedef boost::system::error_code error_code;
    typedef boost::shared_ptr<talk_to_client> ptr;
    void start() {
        started_ = true;
        clients.push_back( shared_from_this());
        last_ping = boost::posix_time::microsec_clock::local_time();
        do_read(); //首先，我们等待客户端连接
    }
    static ptr new_() { ptr new_(new talk_to_client); return new_; }
    void stop() {
        if ( !started_) return;
        started_ = false;
        sock_.close();
        ptr self = shared_from_this();
        array::iterator it = std::find(clients.begin(), clients.end(), self);
        clients.erase(it);
        update_clients_changed();
    }
    bool started() const { return started_; }
    ip::tcp::socket & sock() { return sock_;}
    std::string username() const { return username_; }
    void set_clients_changed() { clients_changed_ = true; }
    … 
private:
    ip::tcp::socket sock_;
    enum { max_msg = 1024 };
    char read_buffer_[max_msg];
    char write_buffer_[max_msg];
    bool started_;
    std::string username_;
    deadline_timer timer_;
    boost::posix_time::ptime last_ping;
    bool clients_changed_;
};
```

我会用*talk_to_client*或者*talk_to_server*来调用*connection*类，从而让你更明白我所说的内容。

现在你需要用到之前的代码了；它和我们在客户端应用中所用到的是一样的。我们还有另外一个*stop()*方法，这个方法用来从客户端数组中移除一个客户端连接。

服务端持续不断地等待异步的*read*操作：

```
void on_read(const error_code & err, size_t bytes) {
    if ( err) stop();
    if ( !started() ) return;
    std::string msg(read_buffer_, bytes);
    if ( msg.find("login ") == 0) on_login(msg);
    else if ( msg.find("ping") == 0) on_ping();
    else if ( msg.find("ask_clients") == 0) on_clients();
}
void on_login(const std::string & msg) {
    std::istringstream in(msg);
    in >> username_ >> username_;
    do_write("login ok\n");
    update_clients_changed();
}
void on_ping() {
    do_write(clients_changed_ ? "ping client_list_changed\n" : "ping ok\n");
    clients_changed_ = false;
}
void on_clients() {
    std::string msg;
    for(array::const_iterator b =clients.begin(),e =clients.end(); b != e; ++b)
        msg += (*b)->username() + " ";
    do_write("clients " + msg + "\n");
} 
```

这段代码是简单易懂的；需要注意的一点是：当一个新客户端登录，我们调用*update_clients_changed()*，这个方法为所有客户端将*clients_changed_*标志为*true*。

服务端每收到一个请求就用相应的方式进行回复，如下所示：

```
void do_ping() { do_write("ping\n"); }
void do_ask_clients() { do_write("ask_clients\n"); }
void on_write(const error_code & err, size_t bytes) { do_read(); }
void do_read() {
    async_read(sock_, buffer(read_buffer_), MEM_FN2(read_complete,_1,_2), MEM_FN2(on_read,_1,_2));
    post_check_ping();
}
void do_write(const std::string & msg) {
    if ( !started() ) return;
    std::copy(msg.begin(), msg.end(), write_buffer_);
    sock_.async_write_some( buffer(write_buffer_, msg.size()), MEM_FN2(on_write,_1,_2));
}
size_t read_complete(const boost::system::error_code & err, size_t bytes) {
    // ... 就像之前
}
```

在每个*write*操作的末尾，*on_write()*方法被调用，这个方法会触发另外一个异步读操作，这样的话“等待请求－回复请求”这个循环就会一直执行，直到客户端断开连接或者超时。

在每次读操作开始之前，我们异步等待5秒钟来观察客户端是否超时。如果超时，我们关闭它的连接：

```
void on_check_ping() {
    ptime now = microsec_clock::local_time();
    if ( (now - last_ping).total_milliseconds() > 5000)
        stop();
    last_ping = boost::posix_time::microsec_clock::local_time();
}
void post_check_ping() {
    timer_.expires_from_now(boost::posix_time::millisec(5000));
    timer_.async_wait( MEM_FN(on_check_ping));
}
```

这就是整个服务端的实现。你可以运行并让它工作起来！

在代码中，我向你们展示了这一章我们学到的东西，为了更容易理解，我把代码稍微精简了下；比如，大部分的控制台输出我都没有展示，尽管在这本书附赠的代码中它们是存在的。我建议你自己运行这些例子，因为从头到尾读一次代码能加强你对本章展示应用的理解。

### 总结

我们已经学到了怎么写一些基础的客户端/服务端应用。我们已经避免了一些诸如内存泄漏和死锁的低级错误。所有的编码都是框架式的，这样你就可以根据你自己的需求对它们进行扩展。

在接下来的章节中，我们会更加深入地了解使用Boost.Asio进行同步编程和异步编程的不同点，同时你也会学会如何嵌入你自己的异步操作。

