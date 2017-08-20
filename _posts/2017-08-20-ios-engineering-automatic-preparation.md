---
layout: post
title: "iOS 工程自动化 - 思路整理"
description: ""

category: iOS 工程自动化
tags: [iOS,工程自动化]
modified: 2017-08-20

imagefeature: mmoaay_bg.jpg
comments: true
share: true
---

4 月份参加 2017@Swift 大会的时候有幸听到了 @zesming 大佬关于美团组件化的 Topic，有一张图印象特别深刻。

![来自 @zesming 大佬](http://upload-images.jianshu.io/upload_images/4481019-ce6260719848d6ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

后来跟 @zesming 大佬沟通怎么去整理组件自动构建发布思路的时候他也跟我提到了这张图。所以我准备围绕这张图来整理一下 iOS 工程自动化的思路。

# 基础知识

首先，我们需要掌握一些自动构建发布的基础知识，主要包含如下几个方面。

## GitFlow - 规范 git 操作流程

GitFlow 是由 Vincent Driessen 提出的一个 Git 操作流程标准。包含如下几个关键分支：

| 名称 | 说明 |
|---|---|
| master | 主分支 |
| develop | 主开发分支，包含确定即将发布的代码 |
| feature | 新功能分支，一般一个新功能对应一个分支，对于功能的拆分需要比较合理，以避免一些后面不必要的代码冲突 |
| release | 发布分支，发布时候用的分支，一般测试时候发现的 bug 在这个分支进行修复 |
| hotfix | 热修复分支，紧急修 bug 的时候用 |

GitFlow 的优势有如下几点：

- 并行开发：GitFlow 可以很方便的实现并行开发：每个新功能都会建立一个新的 `feature` 分支，从而和已经完成的功能隔离开来，而且只有在新功能完成开发的情况下，其对应的 `feature` 分支才会合并到主开发分支上（也就是我们经常说的 `develop` 分支）。另外，如果你正在开发某个功能，同时又有一个新的功能需要开发，你只需要提交当前 `feature` 的代码，然后创建另外一个 `feature` 分支并完成新功能开发。然后再切回之前的 `feature` 分支即可继续完成之前功能的开发。
- 协作开发：GitFlow 还支持多人协同开发，因为每个 `feature` 分支上改动的代码都只是为了让某个新的 `feature` 可以独立运行。同时我们也很容易知道每个人都在干啥。
- 发布阶段：当一个新 `feature` 开发完成的时候，它会被合并到 `develop` 分支，这个分支主要用来暂时保存那些还没有发布的内容，所以如果需要再开发新的 `feature`，我们只需要从 `develop` 分支创建新分支，即可包含所有已经完成的 `feature` 。
- 支持紧急修复：GitFlow 还包含了 `hotfix` 分支。这种类型的分支是从某个已经发布的 tag 上创建出来并做一个紧急的修复，而且这个紧急修复只影响这个已经发布的 tag，而不会影响到你正在开发的新 `feature`。

### Glow flow 是如何工作的

新功能都是在 `feature` 分支上进行开发

![](http://upload-images.jianshu.io/upload_images/4481019-c5cd904f0b2c2cd2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 `feature` 分支都是从 `develop` 分支创建，完成后再合并到 `develop` 分支上，等待发布。

![](http://upload-images.jianshu.io/upload_images/4481019-4533145a3f9ce5b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当需要发布时，我们从 `develop` 分支创建一个 `release` 分支

![](http://upload-images.jianshu.io/upload_images/4481019-bc6d36585ed8d8a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后这个 `release` 分支会发布到测试环境进行测试，如果发现问题就在这个分支直接进行修复。在所有问题修复之前，我们会不停的重复**发布->测试->修复->重新发布->重新测试**这个流程。

发布结束后，这个 `release` 分支会合并到 `develop` 和 `master` 分支，从而保证不会有代码丢失。

![](http://upload-images.jianshu.io/upload_images/4481019-42a2ee57874a1fc8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`master` 分支只跟踪已经发布的代码，合并到 `master` 上的 commit 只能来自 `release` 分支和 `hotfix` 分支。

 `hotfix` 分支的作用是紧急修复一些 Bug。

它们都是从 `master` 分支上的某个 tag 建立，修复结束后再合并到 `develop` 和 `master` 分支上。

![](http://upload-images.jianshu.io/upload_images/4481019-d25bd9456431c6bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### GitFlow 工具

如果要在项目中引入 GitFlow，推荐使用 SourceTree 来做客户端工具，它包含了所有 GitFlow 的流程，可视化操作，很方便。

![](http://upload-images.jianshu.io/upload_images/4481019-32424464dd5c2fc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## gitignore - 干掉那些干扰代码

为什么提到 gitignore，审过 PR（Pull request） 或者 MR（Merge request）的同学应该深有体会，当一个带着一大堆 Pods 库代码的 PR/MR 袭来的时候，你的内心应该是绝望的……所以我们要了解下怎么把一些非模块相关的代码 **ignore** 掉。

###  创建项目仓库中的 .gitignore

如果你在项目仓库内创建一个 `.gitignore` 文件，Git 会根据这个文件来决定哪些文件和目录是需要忽略的。

> 注意：如果你想把一个已经被跟踪的文件 ignore 掉，这是时候新增的规则并不会对这个文件产生作用，你需要先用下面的指令把这个文件设置为不跟踪：
> ```
>git rm --cached FILENAME
> ```

### 创建全局的 .gitignore

这个其实没啥意思，不过还是看一下怎么设置吧：

```
git config --global core.excludesfile ~/.gitignore_global
```

### iOSer 需要的 .gitignore

Github 官方模版给出的建议如下：

```
# Xcode
#
# gitignore contributors: remember to update Global/Xcode.gitignore, Objective-C.gitignore & Swift.gitignore

## Build generated
build/
DerivedData/

## Various settings
*.pbxuser
!default.pbxuser
*.mode1v3
!default.mode1v3
*.mode2v3
!default.mode2v3
*.perspectivev3
!default.perspectivev3
xcuserdata/

## Other
*.moved-aside
*.xccheckout
*.xcscmblueprint

## Obj-C/Swift specific
*.hmap
*.ipa
*.dSYM.zip
*.dSYM

# CocoaPods
#
# We recommend against adding the Pods directory to your .gitignore. However
# you should judge for yourself, the pros and cons are mentioned at:
# https://guides.cocoapods.org/using/using-cocoapods.html#should-i-check-the-pods-directory-into-source-control
#
# Pods/

# Carthage
#
# Add this line if you want to avoid checking in source code from Carthage dependencies.
# Carthage/Checkouts

Carthage/Build

# fastlane
#
# It is recommended to not store the screenshots in the git repo. Instead, use fastlane to re-generate the
# screenshots whenever they are needed.
# For more information about the recommended setup visit:
# https://docs.fastlane.tools/best-practices/source-control/#source-control

fastlane/report.xml
fastlane/Preview.html
fastlane/screenshots
fastlane/test_output

# Code Injection
#
# After new code Injection tools there's a generated folder /iOSInjectionProject
# https://github.com/johnno1962/injectionforxcode

iOSInjectionProject/
```

通过这个文件我们可以看到，忽略的内容主要包含下面几种：

- 构建产生的目录和文件；
- 各种临时配置文件；
- Obj-C / Swift 相关的特定文件；
- CocoaPods 第三方 Pods 库目录；
- Carthage 构建目录；
- fastlane 构建产生的文件；
- 代码注入工具产生的目录及文件。

我们可以根据自身项目的情况来对这些配置进行相应的调整和定制。

## githooks - 背后的“男人”

> 注：本段主要翻译自 githooks 官方文档。

### githooks 简介

githooks 是指 git 在执行某些特定操作时会触发的一系列程序，有点类似数据库中的触发器，由用户编写。默认情况下，这些程序都被放置在 `$GIT_DIR/hooks` 目录下，当然，我们也可以通过 git 的环境配置变量 `core.hooksPath` 来改变这个目录。

githooks 的类型有很多中，这些 hook 会在各个特定操作时帮你自动完成一些你想要做的操作，比如：在提交代码到 develop 分支的时候自动打包。

### 目前支持的 githooks

#### `git commit` 流程

##### pre-commit

由 `git commit` 触发，可以通过 `--no-verify` 选项来跳过，这个 hook 不需要参数，在得到提交消息并开始提交(commit)前被调用，如果返回非 0，则会导致 `git commit` 失败。

当默认的 pre-commit 钩子被启用时，如果它发现文件尾部有空白行，此次提交就会被终止。

如果进行 `git commit` 的命令没有指定一个编辑器来修改提交信息(commit message)，任何的 `git commit` hook 被调用时都会带上环境变量 `GIT_EDITOR=:`

##### prepare-commit-msg

执行 `git commit` 命令后，在默认提交消息准备好后但编辑器启动前，这个 hook 就会被调用。

它接受 1 到 3 个参数。第 1 个是包含了提交消息的文件的名字。第 2 个是提交消息的来源，它可以是：
 
- `message`（如果指定了 `-m` 或者 `-F` 选项） 
- `template`（如果指定了 `-t` 选项，或者在 `git config`中开启了 `commit.template` 选项） 
- `merge`（如果本次提交是一次合并，或者存在文件 `.git/MERGE_MSG`） 
- `squash`（如果存在文件 `.git/SQUASH_MSG` ）
- `commit` 并且第 3 个参数是一个提交的 SHA1 值（如果指定了 `-c`,`-C` 或者 `--amend` 选项）

如果返回值不是 0，那么 `git commit` 命令就会被中止。

这个 hook 的目的是用来在工作时编辑信息文件，并且不会被 `--no-verify` 选项跳过。一个非 0 值意味着 hook 工作失败，会终止提交。它不应该用来作为 `pre-commit` 钩子的替代。

##### commit-msg

由 `git commit` 触发，可以通过 `--no-verify` 选项来跳过，接受 1 个参数，这个参数包含了提交消息的文件的名字，如果返回非 0，则会中止 `git commit` 命令。

这个 hook 可以用来规范提交信息，比如把信息格式化成项目定制的标准格式，或者发现提交信息不符合格式时拒绝这次提交。

##### post-commit

由 `git commit` 触发，在提交后被调用，不能影响 `git commit` 的结果。

#### `git push` 流程

##### pre-push

在 `git push` 运行期间，更新了远程引用但尚未传送对象时执行。如果返回非 0，将中止推送过程。它接受远程分支的名字和位置作为参数，同时从标准输入中读取一系列待更新的引用。 你可以在推送开始之前，用它验证对引用的更新操作。

##### pre-receive

服务端处理来自客户端的推送操作时，最先被调用的 hook。它从标准输入获取一系列被推送的引用。如果它以非 0 值退出，所有的推送内容都不会被接受。你可以用这个 hook 阻止对 `non-fast-forward` 的更新，或者对该推送所修改的所有引用和文件进行访问控制。

##### update

和 `pre-receive` hook 十分类似，不同之处在于它会为每一个准备更新的分支各运行一次。假如推送者同时向多个分支推送内容，`pre-receive` 只运行一次，相比之下 `update` 则会为每一个被推送的分支各运行一次。它不会从标准输入读取内容，而是接受三个参数：引用的分支名；推送前引用所指向内容的 SHA-1 值；以及用户准备推送内容的 SHA-1 值。如果这个 hook 以非 0 值退出，只有相应的那一个引用会被拒绝；其余的依然会被更新。

##### post-receive

在整个过程完结以后运行，可以用来更新其他系统服务或者通知用户。它接受与 `pre-receive` 相同的标准输入数据。它的用途包括给某个邮件列表发信，通知持续集成服务器，或者更新缺陷追踪系统 —— 甚至可以通过分析提交信息来决定某个问题是否应该被开启、修改或者关闭。该脚本无法中止 push 进程，不过客户端在它结束运行之前将保持连接状态，所以如果你想做其他操作需谨慎使用它，因为它将耗费你很长的一段时间。

##### post-update

由远程仓库的 `git-receive-pack` 调用，也是就是本地仓库完成 `git push` 时。这个指令只能用来做通知，不能改变 `git-receive-pack` 的结果。

##### push-to-checkout

由远程仓库的 `git-receive-pack` 调用，也是就是本地仓库完成 `git push` 时。如果这个 hook 返回非 0 值，则会中断 `git push` 操作。

#### applypatch-msg & pre-applypatch & post-applypatch

由 `git am` 触发，主要用于引入第三方 patch 的时候用。因为项目中暂时没有这样的使用场景，所以不做具体研究。

#### pre-rebase

由 `git rebase` 触发，可以用来防止某个分支被 rebase。这个 hook 接受 1 到 2 个参数。第 1 个参数是上游分支，第 2 个参数是将要执行 rebase 的分支。

#### post-checkout

在 `git checkout` 成功运行后执行。你可以根据你的项目环境用它调整你的工作目录。包括放入大的二进制文件、自动生成文档或进行其他类似这样的操作。

#### post-merge

在 `git merge` 成功运行后执行。你可以用它恢复 Git 无法跟踪的工作区数据，比如权限数据。这个 hook 也可以用来验证某些在 Git 控制之外的文件是否存在，这样你就能在工作区改变时，把这些文件复制进来。

#### post-rewrite

这个 hook 被那些会替换提交记录的命令调用，比如 `git commit --amend` 和 `git rebase`（不过不包括 `git filter-branch`）。 它唯一的参数是触发重写的命令名，同时从标准输入中接受一系列重写的提交记录。 这个 hook 的用途很大程度上跟 `post-checkout` 和 `post-merge` 差不多。

#### sendemail-validate

这个 hook 由 `git send-email` 触发，它接受一个参数：包含 e-mail 接受者邮箱的文件名。如果这个 hook 返回非 0 值，`git send-email` 就会被中止。

#### pre-auto-gc

Git 的一些日常操作在运行时，偶尔会调用 `git gc --auto` 进行垃圾回收。这个 hook 会在垃圾回收开始之前被调用，可以用它来提醒你现在要回收垃圾了，或者依情形判断是否要中断回收。

## Cocoapods - 大管家

做为一个第三方库依赖管理工具，Cocoapods 在模块化开发中扮演了非常核心的角色。关于 Cocoapods 的功能和介绍这里就不一一陈述了，主要看一下和模块化开发过程中相关的一些东西。

### 自定义 Cocoapods 模板

我们一般用 `pod lib create` 这个指令来创建一个模块，其实这个指令还有一个选项：`--template-url=URL`，用来指定生成库的模板，官方模板地址是：[https://github.com/CocoaPods/pod-template](https://github.com/CocoaPods/pod-template)，根据这个模板生成的工程目录结构如下：

```
MyLib
  ├── .travis.yml
  ├── _Pods.xcproject
  ├── Example
  │   ├── MyLib
  │   ├── MyLib.xcodeproj
  │   ├── MyLib.xcworkspace
  │   ├── Podfile
  │   ├── Podfile.lock
  │   ├── Pods
  │   └── Tests
  ├── LICENSE
  ├── MyLib.podspec
  ├── Pod
  │   ├── Assets
  │   └── Classes
  │     └── RemoveMe.[swift/m]
  └── README.md
```

也就是说，我们可以 fork 这份官方模版地址，然后定制我们自己的模板（包括主工程和业务模块），增加自定义的功能。比如：

- 添加所有基础库依赖。
- 添加私有 Cocoapods 仓库。
- 添加 .gitignore 文件。
- 添加自定义脚本。

### 私有 Cocoapods 仓库

和主工程一样，模块也是需要做版本管理的，目前来看比较好的方式就是通过 Cocoapods 来发布版本。所以我们需要准备一个自己的私有 Cocoapods 仓库，其实很简单，首先在内网 git 上建立一个空仓库，然后在本地执行一下下面的指令即可：

```
pod repo add [Spec name] [Git url]
```

然后就是在发布库的时候注意一下，用如下指令即可发布到私有仓库内：

```
pod repo push [Spec name] [Lib name].podspec
```

### Cocoapods 引用第三方库的几种方式


使用过 Cocoapods 的童鞋应该都知道，Cocoapods 的引用方式有三种：

|方式|例子|说明|
|---|:---:|----|
|版本号引用|pod 'Alamofire', '~> 3.0'|这种方式引用的是已经发布的版本，包含了 `>``>=``<``<=``~>` 几种版本限制符号，其中`~>`符号代表只更新最新的小版本号，比如 `~> 1.0.0` 则只会更新到 1.0.x 的最新版本，而不会更新 1.x.0 以上的版本|
|本地路径引用|pod 'Alamofire', :path => '~/Documents/Alamofire'|这种方式直接引用本地的代码，这种方式下对引用库的修改仍然会提交到引用库的 git 上，而不会提交到主工程。|
|远程 git 路径引用|pod 'Alamofire', :git => 'https://github.com/Alamofire/Alamofire.git'|这种方式直接引用远程 git 代码，不需要引用的库进行发布，而且还支持 `:branch =>`、`:tag =>` 和 `:commit =>` 三种选项|


# 流程分析

下面就是我对 @zesming 大佬分享的流程图中各个过程的一些思考。

## 业务方需求到开发

根据 GitFlow 的规范，新的需求走的是 `feature` 的流程。这里我们应该可以开发一些辅助开发的脚本。

### 一键新建业务模块和主工程 `feature` 分支

组件化到了一个完整阶段的时候，主工程应该是没有代码的，只是一个壳。但是在发展阶段的时候，主工程还会包含一些业务代码，所以我们在开发某个 `feature` 的时候，往往是模块内有一些具体业务代码，主工程还有一些调用代码，这个时候就需要在主工程和业务模块都新建同样的 `feature` 分支，所以我们在**主工程**增加一个脚本。这个脚本的参数包含：


- 业务模块名；
- 业务模块本地路径；
- `feature` 分支名。


执行过程如下：

1. 为主工程创建 `feature` 分支；
2. 进入业务模块所在目录，为业务模块创建 `feature` 分支；
3. 设置主工程 Podfile 中业务模块引用方式为本地路径引用；
4. 打开主工程和业务模块 Example 工程。

### 一键切换主工程开发调试状态

一键切换主工程开发状态是指某些业务模块需要依赖主工程来进行调试的时候（PS：当然，比较理想的状态是业务模块可以独立运行，不过一般情况下，理想很美好，现实很残酷😂），需要将 Podfile 中这个模块的引用方式修改为本地路径的引用方式。这样主工程代码的修改和业务模块代码的修改会分别提交到各自的 git 仓库，从而实现边调试边开发边提交代码。

这个功能考虑用脚本的方式来实现，放置在**主工程**的常用脚本目录下。参数应该包含：

- 业务模块名；
- 业务模块本地路径；
- 业务模块 `feature` 分支名 [可选]。

执行的操作应该包含：

1. 如果指定了业务模块 `feature` 分支名，则需要先给业务模块新建分支，否则直接执行第 2 步；
2. 修改主工程 Podfile 中业务模块的引用方式为本地路径引用；
3. `pod install`；
4. 关闭主工程并重新打开。

### 一键切换主工程提交状态

业务模块开发调试完成之后，需要将主工程恢复到正常的状态并提交。

这个功能还是用脚本的方式实现，放置在**主工程**的常用脚本目录下。参数应该包含：

- 业务模块名；
- 业务模块远程 git 地址;
- 业务模块 `feature` 分支名。

执行的操作应该包含：

1. 根据当前业务模块本地路径，提交业务模块代码；
2. 修改主工程 Podfile 中业务模块的引用方式为远程 git 引用；
3. `pod install`；
4. 关闭主工程并重新打开。

## 提交代码到业务代码仓库

这个过程我们主要通过 githooks 来做一些一些自动检测。包括：

- OCLint
- 单元测试

## 发布组件到内网 Pods

### 准备阶段

这个阶段做的事情主要是检测当前需要发布的所有分支是否都已经提交 PR / MR 并合并到了 `develop` 分支。

> 注：这里对分支的命名会有一些规则上的要求比。如在分支名内需要带上**当前需要发布的版本号**，从而可以通过这个版本号匹配到当前需要发布的所有分支。

### 发布 -> 测试 -> bug fix -> 再次发布阶段

这里需要把 GitFlow 和 Cocoapods 的发布流程结合起来。

考虑到一般需要依赖主工程来进行测试，我们需要在**主工程**增加一个本地脚本来辅助发布，包含的参数如下：

- 当前版本号；
- 业务模块本地路径；
- 主工程 `feature` 分支名[可选]。

实现的功能如下：

1. 如果提供了主工程 `feature` 分支名，需要先切换主工程分支；否则跳过这一步；
2. 根据传入的当前版本号从 `develop` 分支建立 `release` 分支。如果已经存在，则跳过这一步；
3. 将主工程 git 仓库中 Podfile 引用该模块的方式替换为引用远程 git 仓库的 `release` 分支；
4. 然后在主工程执行 `pod update [模块名]` 更新代码；
5. 推送主工程代码到远程 git 仓库。
6. 通过 githooks 的方式打包主工程并提交测试；
7. 测试过程中如果存在问题，则通过**一键切换主工程开发调试状态**脚本直接切换开发状态并进行 bug fix；
8. bug fix 之后重新执行第 1 步。

### 测试完成发布到内网 Pods 阶段

测试完成后就可以发布业务模块到内网 Pods 了。这里我们在**业务模块工程**内准备一个脚本，参数如下：

- 当前版本号。
- 主工程本地路径。

实现的功能如下：

1. 首先我们要修改 .podspec 文件中的版本号为传入的当前版本号，并提交 push 到 `release` 分支。
2. 然后根据 GitFlow 的 `release` 流程，合并 `release` 分支到 `develop` 分支和 `master` 分支，然后在 `master` 分支建立对应版本号的 tag 并 push 到远程 git 仓库。
3. 然后就可以发布业务模块了，发布完成之后切换到主工程路径将主工程 Podfile 中对该模块的引用方式修改为版本号引用。

## 集成组件到主工程

因为之前发布组件的时候已经将主工程对各业务模块的引用方式修改为版本号引用。所以这个阶段我们只需要验证一下当前主工程引用的是否是各业务模块的最新发布版本即可。参数如下：

- 主工程依赖的所有业务模块列表

完成的功能如下：

检测主工程依赖模块的所有业务模块的最新版本是否和 Podfile 中指定的一致，如果不一致，报错。

## 发布主工程

### 准备阶段

这个阶段做的事情有两点：

1. 检测主工程当前需要发布的所有分支是否都已经提交 PR / MR 并合并到了 `develop` 分支。
2. 检测主工程当前引用的所有业务模块是否都为版本号引用。

### 发布 -> 测试 -> bug fix -> 再次发布阶段

这个阶段可以理解为集成测试阶段。同样是**主工程**中的一个脚本。参数如下：

- 当前版本号。

1. 根据传入的当前版本号从 `develop` 分支建立 `release` 分支。如果已经存在，则跳过这一步；
2. 推送主工程代码到远程 git 仓库；
3. 通过 githooks 的方式打包主工程并提交测试；
4. 测试过程中如果存在问题，则通过**一键切换主工程开发调试状态**脚本直接切换开发状态并进行 bug fix；
5. bug fix 之后重新执行第 1 步。

### 测试完成发布

1. 根据 GitFlow 的 `release` 流程，合并 `release` 分支到 `develop` 分支和 `master` 分支，然后在 `master` 分支建立对应版本号的 tag 并 push 到远程 git 仓库；
2. 通过 githooks 的方式打包主工程并发布。


# 总结

本文只是一个不成熟的思考，后续实施的过程中可能会对一些细节进行改善，也欢迎大家对我思路中不合理的地方进行指正，我会非常感激。

下一篇的内容应该是**业务方需求到开发**这个过程中的一些具体的实践记录。敬请期待！

# 参考资料

Introducing GitFlow：[https://datasift.github.io/gitflow/IntroducingGitFlow.html](https://datasift.github.io/gitflow/IntroducingGitFlow.html)

Using Git / Ignoring files：
[https://help.github.com/articles/ignoring-files/](https://help.github.com/articles/ignoring-files/)

githooks：[https://git-scm.com/docs/githooks](https://git-scm.com/docs/githooks)

自定义 Git - Git 钩子：[https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git-%E9%92%A9%E5%AD%90](https://git-scm.com/book/zh/v2/%E8%87%AA%E5%AE%9A%E4%B9%89-Git-Git-%E9%92%A9%E5%AD%90)