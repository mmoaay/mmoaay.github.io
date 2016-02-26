---
layout: post
title: "Boost.Asio－其他特性"
description: "Boost.Asio C++ Network Programming 翻译"

category: Boost.Asio
tags: [Boost.Asio,翻译]

imagefeature: mmoaay_bg.jpg
comments: true
share: true
---

这章我们讲了解一些Boost.Asio不那么为人所知的特性。标准的stream和streambuf对象有时候会更难用一些，但正如你所见，它们也有它们的益处。最后，你会看到姗姗来迟的Boost.Asio协程的入口，它可以让你的异步代码变得非常易读。这是非常惊人的一个特性。

### 标准stream和标准I/O buffer

读这一章节之前你需要对STL stream和STL streambuf对象有所了解。

Boost.Asio在处理I/O操作时支持两种类型的buffer：

* *boost::asio::buffer()*：这种buffer关联着一个Boost.Asio的操作（我们使用的buffer被传递给一个Boost.Asio的操作）
* *boost::asio::streambuf*：这个buffer继承自*std::streambuf*，在网络编程中可以和STL stream一起使用

纵观全书，之前的例子中最常见的例子如下：

```
size_t read_complete(boost::system::error_code, size_t bytes){ ... }
char buff[1024];
read(sock, buffer(buff), read_complete);
write(sock, buffer("echo\n"));
```

通常来说使用这个就能满足你的需要，如果你想要更复杂，你可以使用*streambuf*来实现。

这个就是你可以用*streambuf*对象做的最简单也是最坏的事情：

```
streambuf buf;
read(sock, buf);
```

这个会一直读到*streambuf*对象满了，然后因为*streambuf*对象可以通过自己重新开辟空间从而获取更多的空间，它基本会读到连接被关闭。

你可以使用*read_until*一直读到一个特定的字符串：

```
streambuf buf;
read_until(sock, buf, "\n");
```

这个例子会一直读到一个“\n”为止，把它添加到*buffer*的末尾，然后退出*read*方法。

向一个*streambuf*对象写一些东西，你需要做一些类似下面的事情：

```
streambuf buf;
std::ostream out(&buf);
out << "echo" << std::endl;
write(sock, buf);
```

这是非常直观的；你在构造函数中传递你的*streambuf*对象来构建一个STL stream，将其写入到你想要发送的消息中，然后使用*write*来发送buffer的内容。

### Boost.Asio 和 STL stream

Boost.Asio在集成STL stream和网络方面做了很棒的工作。也就是说，如果你已经在使用STL扩展，你肯定就已经拥有了大量重载了操作符<<和>>的类。从socket读或者写入它们就好像在公园漫步一样简单。

假设你有下面的代码片段：

```
struct person {
    std::string first_name, last_name;
    int age;
};
std::ostream& operator<<(std::ostream & out, const person & p) {
    return out << p.first_name << " " << p.last_name << " " << p.age;
}
std::istream& operator>>(std::istream & in, person & p) {
    return in >> p.first_name >> p.last_name >> p.age;
} 
```

通过网络发送这个*person*就像下面的代码片段这么简单：

```
streambuf buf;
std::ostream out(&buf);
person p;
// … 初始化p
out << p << std::endl;
write(sock, buf);
```

另外一个部分也可以非常简单的读取：

```
read_until(sock, buf, "\n");
std::istream in(&buf);
person p;
in >> p;
```

使用*streambuf*对象（当然，也包括它用来写入的*std::ostream*和用来读取的*std::istream*）时最棒的部分就是你最终的编码会很自然：
* 当通过网络写入一些要发送的东西时，很有可能你会有多个片段的数据。所以，你需要把数据添加到一个buffer里面。如果那个数据不是一个字符串，你需要先把它转换成一个字符串。当使用<<操作符时这些操作默认都已经做了。
* 同样，在另外一个部分，当读取一个消息时，你需要解析它，也就是说，读取到一个片段的数据时，如果这个数据不是字符串，你需要将它转换为字符串。当你使用>>操作符读取一些东西时这些也是默认就做了的。

最后要给出的是一个非常著名，非常酷的诀窍，使用下面的代码片段把*streambuf*的内容输出到console中

```
streambuf buf;
...
std::cout << &buf << std::endl; //把所有内容输出到console中
```

同样的，使用下面的代码片段来把它的内容转换为一个*string*

```
std::string to_string(streambuf &buf) {
    std::ostringstream out;
    out << &buf;
    return out.str();
} 
```

### streambuf类

我之前说过，*streambuf*继承自*std::streambuf*。就像*std::streambuf*本身，它不能拷贝构造。

另外，它有一些额外的方法，如下：
* *streambuf([max_size,][allocator])*：这个方法构造了一个*streambuf*对象。你可以选择指定一个最大的buffer大小和一个分配器，分配器用来在需要的时候分配/释放内存。
* *prepare(n)*：这个方法返回一个子buffer，用来容纳连续的n个字符。它可以用来读取或者写入。方法返回的结果可以在任何Boost.Asio处理*read/write*的自由函数中使用，而不仅仅是那些用来处理*streambuf*对象的方法。
* *data()*：这个方法以连续的字符串形式返回整个buffer然后用来写入。方法返回的结果可以在任何Boost.Asio处理写入的自由函数中使用，而不仅仅是那些用来处理streambuf对象的方法。
* *comsume(n)*：在这个方法中，数据从输入队列中被移除（从read操作）
* *commit(n)*：在这个方法中，数据从输出队列中被移除(从write操作)然后加入到输入队列中（为read操作准备）。
* *size()*：这个方法以字节为单位返回整个streambuf对象的大小。
* *max_size()*：这个方法返回最多能保存的字节数。

除了最后的两个方法，其他的方法不是那么容易理解。首先，大部分时间你会把*streambuf*以参数的方式传递给*read/write*自由函数，就像下面的代码片段展示的一样：

```
read_until(sock, buf, "\n"); // 读取到buf中
write(sock, buf); // 从buf写入
```

如果你想之前的代码片段展示的一样把整个buffer都传递到一个自由函数中，方法会保证把buffer的输入输出指针指向的位置进行增加。也就是说，如果有数据需要读，你就能读到它。比如：

```
read_until(sock, buf, '\n');
std::cout << &buf << std::endl;
```

上述代码会把你刚从socket写入的东西输出。而下面的代码不会输出任何东西：

```
read(sock, buf.prepare(16), transfer_exactly(16) );
std::cout << &buf << std::endl;
```

字节被读取了，但是输入指针没有移动，你需要自己移动它，就像下面的代码片段所展示的：

```
read(sock, buf.prepare(16), transfer_exactly(16) );
buf.commit(16);
std::cout << &buf << std::endl;
```

同样的，假设你需要从*streambuf*对象中写入，如果你使用了*write*自由函数，则需要像下面一样：

```
streambuf buf;
std::ostream out(&buf);
out << "hi there" << std::endl;
write(sock, buf);
```

下面的代码会把hi there发送三次：

```
streambuf buf;
std::ostream out(&buf);
out << "hi there" << std::endl;
for ( int i = 0; i < 3; ++i)
    write(sock, buf.data());
```

发生的原因是因为buffer从来没有被消耗过，因为数据还在。如果你想消耗它，使用下面的代码片段：

```
streambuf buf;
std::ostream out(&buf);
out << "hi there" << std::endl;
write(sock, buf.data());
buf.consume(9);
```

总的来说，你最好选择一次处理整个*streambuf*实例。如果需要调整则使用上述的方法。

尽管你可以在读和写操作时使用同一个*streambuf*，你仍然建议你分开使用两个，一个读另外一个写，它会让事情变的简单，清晰，同时你也会减少很多导致bug的可能。

### 处理streambuf对象的自由函数

下面列出了Boost.Asio中处理streambuf对象的自由函数：

* *read(sock, buf[, completion_function])*：这个方法把内容从socket读取到*streambuf*对象中。*completion*方法是可选的。如果有，它会在每次*read*操作成功之后被调用，然后告诉Boost.Asio这个操作是否完成（如果没有，它继续读取）。它的格式是：*size_t completion(const boost::system::error_code & err, size_t bytes_transfered);*，如果*completion*方法返回0，我们认为*read*操作完成了，如果非0，它表示下一次调用stream的*read_some*方法需要读取的最大的字节数。
* *read_at(random_stream, offset, buf [, completion_function])*:  这个方法从一个支持随机读取的stream中读取。注意它没有被应用到socket中（因为他们没有随机读取的模型，它们是单向的，一直向前）。
* *read_until(sock, buf, char | string | regex | match_condition)*: 这个方法一直读到满足一个特性的条件为止。或者是一个char类型的数据被读到，或者是一个字符串被读到，或者是一个目前读到的字符串能匹配的正则表达式，或者*match_condition*方法告诉我们需要结束这个方法。*match_condition*方法的格式是：*pair<iterator,bool> match(iterator begin, iterator end);* ，*iterator*代表 *buffers_ iterator<streambuf::const_buffers_type>*。如果匹配到，你需要返回一个*pair*（*passed_end_of_match*被设置成true）。如果没有匹配到，你需要返回*pair*（begin被设置为false）。
* *write(sock, buf [, completion_function])*:  这个方法写入*streambuf*对象所有的内容。*completion*方法是可选的，它的表现和*read()*的*completion*方法类似：当write操作完成时返回0，或者返回一个非0数代表下一次调用stream的*write_some*方法需要写入的最大的字节数。
* *write_at(random_stream,offset, buf [, completion_function])*: 这个方法用来向一个支持随机存储的stream写入。同样，它没有被应用到socket中。
* *async_read(sock, buf [, competion_function], handler)*:  这个方法是*read()*的异步实现，handler的格式为：*void handler(const boost::system::error_code, size_t bytes)*。
* *async_read_at(radom_stream, offset, buf [, completion_function] , handler)*: 这个方法是*read_at()*的异步实现。
* *async_read_until (sock, buf, char | string | regex | match_ condition, handler)*:  这个方法是*read_until()*的异步实现。
* *async_write(sock, buf [, completion_function] , handler)*:  这个方法是*write()*的异步实现。
* *async_write_at(random_stream,offset, buf [, completion_function] , handler)*:  这个方法是*write_at()*的异步实现。

我们假设你需要一直读取直到读到一个元音字母：

```
streambuf buf;
bool is_vowel(char c) {
    return c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u';
}
size_t read_complete(boost::system::error_code, size_t bytes) {
    const char * begin = buffer_cast<const char*>( buf.data());
    if ( bytes == 0) return 1;
    while ( bytes > 0)
        if ( is_vowel(*begin++)) return 0;
        else --bytes;
    return 1;
}
...
read(sock, buf, read_complete);
```

这里需要注意的事情是对*read_complete()*中buffer的访问，也就是*buffer_cast<>*和*buf.data*。

如果你使用正则，上面的例子会更简单：

```
read_until(sock, buf, boost::regex("^[aeiou]+") ); 
```

或者我们修改例子来让*match_condition*方法工作起来：

```
streambuf buf;
bool is_vowel(char c) {
    return c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u';
}
typedef buffers_iterator<streambuf::const_buffers_type> iterator;
std::pair<iterator,bool> match_vowel(iterator b, iterator e) {
    while ( b != e)
        if ( is_vowel(*b++)) return std::make_pair(b, true);
    return std::make_pair(e, false);
}
...
size_t bytes = read_until(sock, buf, match_vowel);
```

当使用*read_until*时会有个难点：你需要记住你已经读取的字节数，因为下层的buffer可能多读取了一些字节（不像使用*read()*时）。比如：

```
std::cout << &buf << std::endl;
```

上述代码输出的字节可能比*read_until*读取到的多。

### 协程

Boost.Asio的作者在2009-2010年间实现了非常酷的一个部分，协程，它能让你更简单地设计你的异步应用。

它们可以让你同时享受同步和异步两个世界中最好的部分，也就是：异步编程但是很简单就能遵循流程控制，就好像应用是按流程实现的。

![](http://d.pcs.baidu.com/thumbnail/75bba5ebc1781380baf5c8ecf40b7f6e?fid=3238002958-250528-571276571493867&time=1420772400&sign=FDTAER-DCb740ccc5511e5e8fedcff06b081203-xqB0SvR9wei6sSPHYGH86JOKGw4%3D&rt=sh&expires=2h&r=263323555&sharesign=unknown&size=c710_u500&quality=100)

正常的流程已经在情形1种展示了，如果使用协程，你会尽可能的接近情形2。

简单来说，就是协程允许在方法中的指定位置开辟一个入口来暂停和恢复运行。

如果要使用协程，你需要在*boost/libs/asio/example/http/server4*目录下的两个头文件：*yield.hpp*和*coroutine.hpp*。在这里，Boost.Asio定义了两个虚拟的关键词（宏）和一个类：

* *coroutine*：这个类在实现协程时被你的连接类继承或者使用。
* *reenter(entry)*：这个是协程的主体。参数*entry*是一个指向*coroutine*实例的指针，它被当作一个代码块在整个方法中使用。
* *yield code*：它把一个声明当作协程的一部分来运行。当下一次进入方法时，操作会在这段代码之后执行。

为了更好的理解，我们来看一个例子。我们会重新实现 **第四章 异步客户端** 中的应用，这是一个可以登录，ping，然后能告诉你其他已登录客户端的简单客户端应用。
核心代码和下面的代码片段类似：

```
class talk_to_svr : public boost::enable_shared_from_this<talk_to_svr>, public coroutine, boost::noncopyable {
    ...
    void step(const error_code & err = error_code(), size_t bytes = 0) {
        reenter(this) 
        { 
            for (;;) {
                yield async_write(sock_, write_buffer_, MEM_FN2(step,_1,_2) );
                yield async_read_until( sock_, read_buffer_,"\n", MEM_FN2(step,_1,_2));
                yield service.post( MEM_FN(on_answer_from_server));
            }
        } 
    }
}; 
```

首先改变的事就是：我们只有一个叫做*step()*的方法，而没有大量类似*connect()，on_connect()，on_read()，do_read()，on_write()，do_write()*等等的成员方法。

方法的主体在*reenter(this) { for (;;) { }}* 内。你可以把*reenter(this)*当作我们上次运行的代码，所以这次我们执行的是下一次的代码。

在*reenter*代码块中，你会发现几个*yield*声明。你第一次进入方法时，*async_write*方法被执行，第二次*async_read_until*方法被执行，第三次*service.post*方法被执行，然后第四次*async_write*方法被执行，然后一直循环下去。

你需要一直记住*for(;;){}*实例。参考下面的代码片段：

```
void step(const error_code & err = error_code(), size_t bytes = 0) {
    reenter(this) {
        yield async_write(sock_, write_buffer_, MEM_FN2(step,_1,_2) );
        yield async_read_until( sock_, read_buffer_, "\n",MEM_FN2(step,_1,_2));
        yield service.post(MEM_FN(on_answer_from_server));
    }
} 
```

如果我们第三次使用上述的代码片段，我们会进入方法然后执行*service.post*。当我们第四次进入方法时，我们跳过*service.post*，不执行任何东西。当执行第五次时仍然不执行任何东西，然后一直这样下去：

```
class talk_to_svr : public boost::enable_shared_from_this<talk_to_svr>, public coroutine, boost::noncopyable {
    talk_to_svr(const std::string & username) : ... {}
    void start(ip::tcp::endpoint ep) {
        sock_.async_connect(ep, MEM_FN2(step,_1,0) );
    }
    static ptr start(ip::tcp::endpoint ep, const std::string &username) {
        ptr new_(new talk_to_svr(username));
        new_->start(ep); 
        return new_;
    }
    void step(const error_code & err = error_code(), size_t bytes = 0)
    {
        reenter(this) { 
            for (;;) {
                if ( !started_) {
                    started_ = true;
                    std::ostream out(&write_buf_);
                    out << "login " << username_ << "\n";
                }
                yield async_write(sock_, write_buf_,MEM_FN2(step,_1,_2));
                yield async_read_until( sock_,read_buf_,"\n",MEM_FN2(step,_1,_2));
                yield service.post(MEM_FN(on_answer_from_server));
            }
        }
    }
    void on_answer_from_server() {
        std::istream in(&read_buf_);
        std::string word;
        in >> word;
        if ( word == "login") on_login();
        else if ( word == "ping") on_ping();
        else if ( word == "clients") on_clients();
        read_buf_.consume( read_buf_.size());
        if (write_buf_.size() > 0) service.post(MEM_FN2(step,error_code(),0));
    }
    ... 
private:
    ip::tcp::socket sock_;
    streambuf read_buf_, write_buf_;
    bool started_;
    std::string username_;
    deadline_timer timer_;
};
```

当我们启动连接时，*start()*被调用，然后它会异步地连接到服务端。当连接完成时，我们第一次进入*step()*。也就是我们发送我们登录信息的时候。

在那之后，我们调用*async_write*，然后调用*async_read_until*，再处理消息（*on_answer_from_server*）。

我们在*on_answer_from_server*处理接收到的消息；我们读取第一个字符，然后把它分发到相应的方法。剩下的消息（如果还有一些消息没读完）我们都忽略掉：

```
class talk_to_svr : ... {
    ...
    void on_login() { do_ask_clients(); }
    void on_ping() {
        std::istream in(&read_buf_);
        std::string answer; in >> answer;
        if ( answer == "client_list_changed")
            do_ask_clients();
        else postpone_ping();
    }
    void on_clients() {
        std::ostringstream clients; clients << &read_buf_;
        std::cout << username_ << ", new client list:" << clients.str();
        postpone_ping();
    }
    void do_ping() {
        std::ostream out(&write_buf_); out << "ping\n";
        service.post( MEM_FN2(step,error_code(),0));
    } 
    void postpone_ping() {
        timer_.expires_from_now(boost::posix_time::millisec(rand() % 7000));
        timer_.async_wait( MEM_FN(do_ping));
    }
    void do_ask_clients() {
        std::ostream out(&write_buf_);
        out << "ask_clients\n";
    }
}; 
```

完整的例子还会更复杂一点，因为我们需要随机地ping服务端。实现这个功能我们需要在第一次请求客户端列表完成之后做一个ping操作。然后，在每个从服务端返回的ping操作的结果中，我们做另外一个ping操作。

使用下面的代码片段来执行整个过程：

```
int main(int argc, char* argv[]) {
    ip::tcp::endpoint ep(ip::address::from_string("127.0.0.1"),8001);
    talk_to_svr::start(ep, "John");
    service.run();
} 
```

使用协程，我们节约了15行代码，而且代码也变的更加易读。

在这里我们仅仅接触了协程的一点皮毛。如果你想要了解更多，请登录作者的个人主页：[http://blog.think-async.com/2010_03_01_archive.html](http://blog.think-async.com/2010_03_01_archive.html)。

### 总结

我们已经了解了如何使用Boost.Asio玩转STL stream和streambuf对象。我们也了解了如何使用协程来让我们的代码更加简洁和易读。

下面就是重头戏了，比如Asio VS Boost.Asio，高级调试，SSL和平台相关特性。
