---
layout: post
title: "前言"
description: "Boost.Asio C++ Network Programming 翻译"

category: Boost.Asio
tags: [Boost.Asio,翻译]
modified: 2016-02-18

imagefeature: mmoaay_bg.jpg
comments: true
share: true
---

# 实战出精华

<!-- more -->

*在具体的C++网络编程中提升你的逼格*

*John Torjo*

---

Boost.Asio C++ 网络编程

Copyright © 2013 Packt Publishing

---

## 关于作者

做为一名权威的C++专家，**John Torjo** 的编程生涯已经超过了15年，在这15年中，除了偶尔用 `C#` 和 `Java` 写程序，他大部分时间都在研究 `C++`。

他还很喜欢在 C++ Users Journal 和其他杂志上写一些编程相关的文章。

闲暇的时候，他喜欢玩扑克、开快车。他有很多自由职业，其中一个就把他玩扑克和编程的爱好结合在了一起，如果你想联系他，可以发邮件到[john.code@torjo.com](john.code@torjo.com)。

---

我要感谢我的朋友 Alexandru Chis, Aurelian Hale, Bela Tibor Bartha, Cristian Fatu, Horia Uifaleanu, Nicolae Ghimbovschi 以及 Ovidiu Deac。感谢他们对本书提出的反馈和建议。同时我也要感谢 Packt 公司各位对我频繁错过截稿日期行为的包容。然后最需要感谢的是 Chris Kohlhoff，Boost.Asio 的作者，是他写出了如此伟大的库。

把这本书献给我最好的朋友 Darius。

---

## 关于评审员

Béla Tibor Bartha

一个使用多种技术和语言进行开发的专业软件工程师。尽管在过去的4年里，他做的是 `iOS` 和 `OSX` 应用开发，但是 `C++` 陪伴他度过了早期个人游戏项目开发的激情岁月。

---

我要感谢 John，因为他我才能做这本书的评审

---

Nicolae Ghimbovschi

一个参加各类 `C++` 项目超过5年的天才个人开发者。他主要参与一些企业通信工程的项目。作为一个狂热的 `Linux` 爱好者，他喜欢利用不同的操作系统、脚本工具和编程语言进行测试和实验。除了编程，他还喜欢骑自行车、瑜伽和冥想。

---

我要感谢 John 让我来评审这本书

---

## 关于译者

画渣程序猿mmoaay，技术很烂，喜欢平面设计、鼠绘、交友、运动和翻译，但是确作为一只程序猿混迹在IT行业。热爱开源，技术烂就只好做做设计和翻译的工作。

微博：[http://weibo.com/smmoaay](http://weibo.com/smmoaay)

---

## 关于avplayer

[http://avplayer.org](http://avplayer.org) 中国第一技术社区。

---

## 目录

---

前言

---

第一章：Boost.Asio 入门

    什么是 Boost.Asio？
        历史
        依赖
        编译 Boost.Asio
        重要的宏
    同步 VS 异步
    异常 VS 错误代码
    Boost.Asio 中的多线程
    不仅仅是网络
    计时器
    io_service 类
    总结

---

第二章：Boost.Asio 基本原理

    网络 API
    Boost.Asio 命名空间
    IP 地址
    端点
    Sockets
        同步错误代码
        Socket 成员函数
        其他注意事项
    read/write/connect自由函数
        connect 函数
        read/write 函数
    异步编程
        为什么要异步？
        异步 run(),run_one(),poll(),poll_one()
            持续运行
            run_one(),poll(),poll_one() 函数
        异步工作
        异步 post() VS dispatch() VS wrap()
    保持运行
    总结

---

第三章：回显服务端/客户端

    TCP 回显服务端/客户端
        TCP 同步客户端
        TCP 同步服务端
        TCP 异步客户端
        TCP 同步服务端
        代码
    UDP 回显服务端/客户端
        UDP 同步回显客户端
        UDP 同步回显服务端
    总结

---

第四章：客户端和服务端

    同步客户端/服务端
        同步客户端
        同步服务端
    异步客户端/服务端
        异步客户端
        异步服务端
    总结

---

第五章：同步VS异步

    同步异步混合编程
    客户端和服务端之间消息的互相传递
    客户端软件中的同步 I/O
    服务端软件中的同步 I/O
        同步服务端中的线程
    客户端软件中的异步 I/O
    服务端软件中的异步 I/O
        异步服务端中的线程
    异步操作
    代理实现
    总结

---

第六章：Boost.Asio-其他特性

    std streams 和 std buffer I/O
    Boost.Asio 和 STL流
    streambuf 类
    处理 streambuf 对象的自由函数
    协程
    总结

---

第七章：Boost.Asio-进阶

    Asio VS Boost.Asio
    调试
        处理程序跟踪信息
        例子
        处理程序跟踪文件
    SSL
    Boost.Asio 的 Windows特性
        流处理
        随机存储处理
        对象处理
    Boost.Asio 的 POSIX 特性
        本地 sockects
        连接本地 sockets
        POSIX 文件描述符
        Fork
        总结

---

索引

---

## 前言

网络编程由来已久，并且极富挑战性。Boost.Asio 对网络编程做了一个极好的抽象，从而保证只需要少量代码就可以实现一个优雅的客户端/服务端软件。在实现的过程中，它能让你体会到极大的乐趣。而且更为有益的是：Boost.Asio 还包含了一些非网络的特性，用 Boost.Asio 写出来的代码紧凑、易读，而且如果按照我在书中所说的来做，你的代码会无懈可击。

这本书涵盖了什么？

*第一章：Boost.Asio入门*将告诉你 Boost.Asio 是什么？怎么编译它？然后会有一些例子。你会发现 Boost.Asio 不仅仅是一个网络库。另外你还会接触到 Boost.Asio 中最核心的类 `io_service`。

*第二章：Boost.Asio基本原理*包含了你必须了解的内容：什么时候使用 Boost.Asio？我们将深入了解异步编程——一种比同步更需要技巧，且更有乐趣的编程方式。这一章也是在开发你自己的网络应用时可以做为参考的一章。

*第三章：回显服务端/客户端*将告诉你如何实现一个小的客户端/服务端软件；也许这会是你写过的最简单的客户端/服务端软件。回显应用所做的事情就是把客户端发过来的消息发送回去然后关闭客户端连接。我们会先实现一个同步的版本，然后再实现一个异步的版本，这样就可以非常容易地比较它们之间的不同。

*第四章：客户端和服务端*会深入讨论如何用 Boost.Asio 创建一个简单的客户端/服务端应用。我们将讨论如何避免内存泄漏和死锁等问题。为了更方便地对它们进行扩展以满足你的需求，所有的程序都只是实现一个简单的框架。

*第五章：同步 VS 异步*会带你了解在同步和异步方式之间做选择时需要考虑的事情。首要的事情就是不要混淆它们。在这一章，我们会发现实现、测试和调试每个类型应用是非常容易的。

*第六章：Boost.Asio 的其他特性*将带你了解 Boost.Asio 一些不为人知的特性。你会发现，虽然 std streams 和 streambufs 有一点点难用，但是却表现出了它们得天独厚的优势。最后，是姗姗来迟的 Boost.Asio 协程，它可以让你用一种更易读的方式来写异步代码。（就好像写同步代码一样）

*第七章：Boost.Asio 进阶*包含了一些 Boost.Asio 进阶问题的处理。虽然在日常编程中不需要深入研究它们，但是了解它们对你有益无害（Boost.Asio 高级调试，SSL，Windows 特性，POSIX 特性等）。

### 读这本书之前你需要准备什么？

如果要编译 Boost.Asio 以及运行本书中的例子，你需要一个现代编译器。例如，Visual Studio 2008 及其以上版本或者 g++ 4.4 及其以上版本

### 这本书是为谁写的？

这本书对于那些需要做网络编程却不想深入研究复杂网络 API 的开发者来说是一个福音。所有你需要的只是 Boost.Asio 提供的一套 API 。作为著名 Boost C++ 库的一部分，你只需要额外添加几个 #include 文件即可转换到 Boost.Asio。

在读这本书之前，你需要了解 Boost 核心库的一些知识，例如 Boost 智能指针、boost::noncopyable、Boost Functors、Boost Bind、shared_ from_this/enabled_shared_from_this 和 Boost 线程（线程和互斥量）。同时还需要了解 Boost 的 Date/Time。读者还需要知道阻塞的概念以及“非阻塞”操作。

### 约定

本书使用不同样式的文字来区分不同种类的信息。这里给出这些样式的例子以及它们的解释。

文本中的代码会这样显示：“通常一个 `io_service` 的例子就足够了”。

代码是下面这样的：


```
read(stream, buffer [, extra options])

async_read(stream, buffer [, extra options], handler)

write(stream, buffer [, extra options])

async_write(stream, buffer [, extra options], handler)
```

**专业词汇和重要的单词**用黑体显示

[*！警告或者重要的注释在这样的一个框里面*]

[*？技巧在这样的一个框里面*]

### 读者反馈

我们欢迎来自读者的反馈。告诉我们你对这本书的看法——喜欢哪部分，不喜欢哪部分。读者的反馈对我们非常重要，它能帮助我们写出对读者更有益的书。

你只需要发送邮件到 [feedback@packtpub.com](feedback@packtpub.com) 即可进行反馈，注意在邮件的主题中注明书名。

如果你有一个擅长的专题，想撰写一本书或者为某本书做贡献。请阅读我们在 [www.packtpub.com/authors](www.packtpub.com/authors) 上的作者指引。

### 用户支持

现在你已经是 Packt 书籍的拥有者，我们将告诉你一些事项，让你购买本书得到的收益最大化。

### 下载示例代码

你可以在 [http://www.packtpub.com](http://www.packtpub.com) 登录你的帐号，然后下载你所购买的书籍的全部示例代码。同时，你也可以通过访问 [http://www.packtpub.com/support](http://www.packtpub.com/support) 进行注册，然后这些示例代码文件将直接发送到你的邮箱。

### 纠错

尽管我们已经尽最大的努力去保证书中内容的准确性，但是错误还是不可避免的。如果你在我们的书籍中发现了错误——也许是文字，也许是代码——如果你能将它们报告给我们，我们将不胜感激。这样的话，你不仅能帮助其他读者，同时也能帮助我们改进这本书的下一个版本。如果你发现任何需要纠正的地方，访问 [http://www.packtpub.com/submit-errata](http://www.packtpub.com/submit-errata)，选择你的书籍，点击 **errata submission form** 链接，然后输入详细的纠错信息来将错误报告给我们。一经确定，你的提交就会通过，然后这个纠错就会被上传到我们的网站，或者添加到那本书的纠错信息区域的纠错列表中。所有已发现的纠错都可以访问 [http://www.packtpub.com/support](http://www.packtpub.com/support)，然后通过选择书名的方式来查看。

### 答疑

如果你有关于本书任何方面的问题，你可以通过 [questions@packtpub.com](questions@packtpub.com) 联系我们。我们将尽我们最大的努力进行解答
