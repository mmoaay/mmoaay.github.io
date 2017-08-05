---
layout: post
title: "iOS 工程自动化 - OCLint"
description: ""

category: iOS 工程自动化
tags: [iOS,工程自动化,OCLint]
modified: 2017-08-06

imagefeature: mmoaay_bg.jpg
comments: true
share: true
---

# 前言

最近一直在做 iOS 工程自动化方向的事情，所以把自己研究和实践的内容进行记录并分享，希望能给大家一些帮助。

# 为什么要使用 OCLint

做为一个静态代码分析工具，我们引入 OCLint 的目的主要是为了提高我们的代码质量。通常我们提高代码质量的方式是通过 CodeReview，但是这个过程耗费的人工和时间往往较大，所以我们想通过 OCLint 的一些规则，让机器帮我们完成一部分代码质量的检测，从而提高我们的工作效率。

# 安装 OCLint

OCLint 的安装方式有很多中，这里我们选择最简单的方式：通过 Homebrew 安装。

## 安装 Homebrew

```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## 安装 OCLint

```
brew tap oclint/formulae
brew install oclint
```

# 命令行指令

OCLint 包含三个命令行指令：

- oclint：基础指令。通过这个指令可以指定加载验证规则、编译代码、分析代码和生成报告。
- oclint-json-compilation-database：高级指令。通过这个指令可以从 `compile_commands.json` 文件中读取配置信息并执行 oclint。
- oclint-xcodebuild：通过这个指令可以从 Xcode 的 `xcodebuild.log` 文件导出编译选项并保存成 JSON Compilation Database 格式。然后把保存到 `compile_commands.json` 文件中。

然后我们来看一下每个指令的选项和功能：

## oclint 

### 规则加载选项

- -R <目录>：指定规则加载的目录。可以是多个目录，用空格隔开，默认情况下会搜索 `$(oclint 可执行文件目录)/../lib/oclint/rules`。
- -disable-rule <规则名> 通过规则名使某些验证规则失效。
- -rc <参数>=<值> 修改某些阈值。

下面是一个简单的示例：

```
oclint -R /path/to/rules -disable-rule GotoStatement
```

表示从 `/path/to/rules` 加载规则，然后使 `GotoStatement` 这个验证规则失效。

然后来看一下阈值，下面是 OCLint 里面一些常见的阈值：

|名称|描述|默认值|
| ------------- |:-------------| -----|
| CYCLOMATIC_COMPLEXITY |循环嵌套数限制|10|
| LONG_CLASS |类行数限制|1000|
| LONG_LINE |每行的字符限制|100|
| LONG_METHOD |方法行数限制|50|
| LONG_VARIABLE_NAME |参数名字符限制|20|
| MAXIMUM_IF_LENGTH |if 的行数限制|15|
| MINIMUM_CASES_IN_SWITCH |switch case 的最小数目|3|
| NPATH_COMPLEXITY |通过该方法的非循环执行路径数量限制|200|
| NCSS_METHOD |连续未注释行数限制|30|
| NESTED_BLOCK_DEPTH |block 嵌套层数限制|5|
| SHORT_VARIABLE_NAME |变量名的最小字符数限制|3|
| TOO_MANY_FIELDS |类成员限制|20|
| TOO_MANY_METHODS |类方法数限制|30|
| TOO_MANY_PARAMETERS |参数个数限制|10|

我们也可以把这些阈值配置到项目的 `.oclint` 文件下，比如：

```
rule-configurations:
  - key: CYCLOMATIC_COMPLEXITY
    value: 15
  - key: LONG_LINE
    value: 50
```

### 编译选项

可以通过 `--` 的方式在指令的最后添加编译选项。

比如我们通过下面的 `clang` 指令编译一个文件：

```
clang -x objective-c -arch armv7 -std=gnu99 -fobjc-arc -O0 -isysroot /Developer/SDKs/iPhoneOS6.0.sdk -g -I./Pods/Headers -c RPActivityIndicatorManager.m
```

在 OCLint 里面就可以这么写：

```
oclint [oclint options] RPActivityIndicatorManager.m -- -x objective-c -arch armv7 -std=gnu99 -fobjc-arc -O0 -isysroot /Developer/SDKs/iPhoneOS6.0.sdk -g -I./Pods/Headers -c
```

### 编译数据库选项

-p <构建目录>：选择一个包含 `compile_commands.json` 文件的目录作为构建目录。如果没有指定构建目录，oclint 指令会查找第一个输入文件的所有父目录来找到 `compile_commands.json` 文件。

### 生成报告选项

- -o <目录>：指定报告的输出目标。
- -report-type <类型名>：指定报告输出的类型。默认是普通文本。

报告输出的类型有如下几种：

- Plain Text Report(text)
- HTML Report(html)
- XML Report(xml)
- JSON Reporter (json)
- PMD Reporter (pmd)：这种类型主要提供给 CI 系统使用，在 CI 系统的展示会更友好。
- Xcode Reporter (xcode)：主要提供给 Xcode 内查看。

同样，我们也可以把这个配置加入到 `.oclint` 文件中：

```
report-type: html
output: oclint.html
```

### 退出状态选项

- -max-priority-1 <阈值>
- -max-priority-2 <阈值>
- -max-priority-3 <阈值>

首先我们来看一下 OCLint 会返回的 5 种退出 Code：

- 0 - SUCCESS：成功。
- 1 - RULE_NOT_FOUND：没有找到验证规则。
- 2 - REPORTER_NOT_FOUND：没有指定报告输出地址。
- 3 - ERROR_WHILE_PROCESSING：验证过程中出错。
- 4 - ERROR_WHILE_REPORTING：生成报告时出错。
- 5 - VIOLATIONS_EXCEED_THRESHOLD：违反规则的次数超出阈值。

然后我们来看一下各个优先级默认的阈值：优先级 3 的阈值为 20；优先级 2 的阈值为 10；优先级 1 的阈值为 0。如果超过了其中任意一个阈值，就表示你的代码质量是不达标的，然后 OCLint 会返回 VIOLATIONS_EXCEED_THRESHOLD。

### 全局分析选项

-enable-global-analysis

开启这个选项可以得到更准确的分析结果，但是相对耗时比较长，一般不采用。

### Clang 静态分析选项

-enable-clang-static-analyzer

如果开启这个选项，OCLint 会 hook Clang 的编译过程，然后收集编译信息然后添加到报告中。需要注意的是：这也会增加分析的时间。

### Debug 选项

-debug

开启这个选项会给出更详细的信息。但是这个选项只有在 OCLint 的 debug 标示开启的时候才有效。

## oclint-json-compilation-database

### 过滤选项

- -i INCLUDES, -include INCLUDES, –include INCLUDES：
- -e EXCLUDES, -exclude EXCLUDES, –exclude EXCLUDES：

这两个选项是指在 `compile_commands.json` 文件中配置的基础上添加一些文件/目录来执行 oclint，或者让一些文件/目录不执行 oclint。

这两个选项支持正则匹配，因为这个命令是用 Python 写的，所以正则的格式以 [Python 正则表达式语法](http://docs.python.org/2/library/re.html#re-syntax) 为准。

比如，我们一般不对第三方库执行 oclint ，这个时候可以用下面的指令来把 Pods 目录排除：

```
oclint-json-compilation-database -e Pods
```

### oclint 的选项

可以通过 `--` 的方式在指令的最后 oclint 选项。比如：

```
oclint-json-compilation-database -e Pods -- -o=report.html
```

在最后添加了一个 oclint 的 -o 选项，表示将报告输出到当前目录的 report.html 文件中。

### 调试选项

- -v：通过这个选项可以输出最终执行的 oclint 指令。
- -debug：开启这个选项会给出更详细的信息。但是同样这个选项只有在 OCLint 的 debug 标示开启的时候才有效。


## oclint-xcodebuild

这个命令是给使用 Xcode 的用户提供的。主要用于分析 `xcodebuild.log` 文件，然后快速生成 `compile_commands.json` 文件。

> 这货已经被 oclint 抛弃了，改用 xcpretty。

### 安装 xcpretty

其实只需要执行下面指令即可:

```
gem install xcpretty
```
### 使用 xcpretty 生成 `compile_commands.json` 文件

通过下面的指令即可生成 `compile_commands.json` 文件：

```
xcodebuild [flags] | xcpretty -r json-compilation-database -o compile_commands.json
```

如果想保存 `xcodebuild.log`，可以换成下面的指令：

```
xcodebuild [flags] | tee xcodebuild.log | xcpretty -r json-compilation-database -o compile_commands.json
```

然后就可以执行 `oclint-json-compilation-database` 来进行验证了。

# 与其他工具配合

## xcodebuild

因为 oclint-xcodebuild 已经被 oclint 抛弃了，所以跟 xcodebuild 的配合可以直接忽略。

## xctool

> 了解到 xctool 在 Xcode 8 以后已经不再支持 build，而是变成了 xcbuild 指令，所以我们换成 xcbuild 来生成 `compile_commands.json` 文件。

### 安装 xcbuild

首先从 github 下载源码：

```
git clone https://github.com/facebook/xcbuild
cd xcbuild
git submodule update --init
```

然后执行如下指令：

```
make
```

> 这里需要用到 cmake 和 ninja，所以我们通过下面的指令安装：

> ```
> brew install cmake ninja
> ```

### 使用 xcbuild 生成 `compile_commands.json` 文件

使用如下指令即可生成 `compile_commands.json` 文件

```
xcbuild [flags] | xcpretty -r json-compilation-database -o compile_commands.json
```

事实上 xcbuild 和 xcodebuild 的指令是完全兼容的…

## Xcode IDE

oclint 可以和 Xcode IDE 结合，把错误直接在 IDE 中显示出来。

首先，我们在项目中创建一个新的 target，然后选择 `Aggregate` 作为模板。

![](http://oclint-docs.readthedocs.io/en/stable/_images/xcode_screenshot_2.png)

然后给这个 target 命名，注意我们可以建立多个 target，然后分别关注代码分析的多个方面。

然后在 Build Phases 选项卡中选择 Add Run Script。

![](http://oclint-docs.readthedocs.io/en/stable/_images/xcode_screenshot_3.png)

关于脚本的编写我们仍然选择最简单的方式，即 xcodebuild + xcpretty + oclint-json-compilation-database 的方式来做 oclint。脚本如下：

```
source ~/.bash_profile
cd ${PROJECT_DIR}
xcodebuild clean
xcodebuild -workspace LPDLogger.xcworkspace -configuration Debug -scheme LPDLogger-Example build | xcpretty -r json-compilation-database -o compile_commands.json
oclint-json-compilation-database -- -report-type xcode
```

然后我们就可以开始执行分析了，因为这里我们选择的 report-type 是 xcode，这时 oclint 发现的错误会直接在 IDE 中标示出来，方便我们对代码进行改进。

![](http://oclint-docs.readthedocs.io/en/stable/_images/xcode_screenshot_8.png)


## Travis CI

Travis CI 大家应该都很熟悉，它的配置也很简单，在工程目录下添加 `.travis.yml` 文件即可。

所以我们在 `.travis.yml` 中配置如下的内容即可：

```
language: objective-c
osx_image: xcode8.0
before_install:
- brew cask uninstall oclint
- brew tap oclint/formulae
- brew install oclint
script:
  - xcodebuild -workspace LPDLogger.xcworkspace -configuration Debug -scheme LPDLogger-Example build | xcpretty -r json-compilation-database -o compile_commands.json
  - oclint-json-compilation-database
```

我们对这个文件做一下分析：

- 首先，`language` 设置为 `objective-c`，`osx_image` 设置为 `xcode8.0`
- 然后 `before_install` 所做的事情就是安装 oclint。
- 最后 `script` 所做的事情就是对代码做 oclint。


## Jenkins CI

相比 Travis CI，Jenkins CI 提供了更友好的界面来进行 oclint 的配置和报告展示。

> 注：这个部分笔者还没有亲自实践，所以主要还是把官方的英文文档用中文的方式表述一遍。

### 建立持续集成的工程

首先，创建一个 free-style 的项目：

![](http://oclint-docs.readthedocs.io/en/stable/_images/jenkins_1.png)

设置持续集成项目所需要的一些步骤：

![](http://oclint-docs.readthedocs.io/en/stable/_images/jenkins_2.png)


### 配置 OCLint 和 PMD 插件

新建一个类型为 Execute shell 的 build step：

![](http://oclint-docs.readthedocs.io/en/stable/_images/jenkins_3.png)

然后配置 oclint 的指令，这里有几个注意点：

- 先添加生成 `compile_commands.json` 指令。
- 在某些情况下，`oclint` 指令比 `oclint-json-compilation-database` 指令好用。
- 设置 report-type 为 pmd。
- 设置输出文件名，接下来我们还需要用到这个名字。

![](http://oclint-docs.readthedocs.io/en/stable/_images/jenkins_4.png)

新建一个类型为 `Publish PMD analysis results` 的 post-build action。

![](http://oclint-docs.readthedocs.io/en/stable/_images/jenkins_5.png)

然后输入刚才输出的文件名。

![](http://oclint-docs.readthedocs.io/en/stable/_images/jenkins_6.png)

### 执行分析

然后我们就可以开始进行分析了，分析结束之后我们就可以通过 `PMD Warnings` 来查看各个规则相应的报错数了。

![](http://oclint-docs.readthedocs.io/en/stable/_images/jenkins_8.png)


# 结语

至此 OCLint 的介绍以及集成都已经完成了，大家有兴趣的话也可以在自己的开源项目中实践一下，总的来说还是比较简单的。下一篇文章应该是 Cocoapods 库发布相关的一些内容，敬请期待……

# 参考资料

官方文档：[http://oclint-docs.readthedocs.io/en/stable/](http://oclint-docs.readthedocs.io/en/stable/)

