# Lab 1: MapReduce

***Lab1*** 主要是结合**MapReduce**论文，实现一个分布式的**MapReduce**

## Link

[Lab 1: MapReduce](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)

## MapReduce论文

[英文版](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)

[中文版](http://aleda.cn/papers/DistributedSystem/mapreduce-osdi04-cn.pdf)

## Introduction

本实验中，您将构建一个 ***MapReduce*** 系统。您将实现一个调用应用程序 ***Map()*** 和 ***Reduce******()*** 函数并处理读取和写入文件的***Worker process***（工作进程），以及一个将任务分发给工作人员并处理失败工作人员的***Coordinator process***（协调员进程）。您将构建一个类似于 ***[MapReduce](http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)*** 论文中的内容。（注意：Lab中使用***Coordinator***来代替论文中的***Master***）



## Getting started

首先要配置[Go](https://studygolang.com/dl)环境

```shell
$ go env
```

您将使用 [git](https://git-scm.com/)（一个版本控制系统）获取初始的***Lab***代码。要了解有关 git 的更多信息，请查看 [Pro Git Book](https://git-scm.com/book/en/v2)或[Git User Manual](http://www.kernel.org/pub/software/scm/git/docs/user-manual.html)。拉取6.824代码：

```shell
$ git clone git://g.csail.mit.edu/6.824-golabs-2021 6.824
$ cd 6.824
$ ls
Makefile src
$
```

我们为您提供了一个简单的***sequential***(串行执行而不是并发的) ***MapReduce*** 实现在`src/main/mrsequential.go`。它在一个进程中按一次一个的执行***Map()*** 和 ***Reduce******()*** 函数。我们还为您提供了一些 ***MapReduce*** 应用程序：`mrapps/wc.go` 中的 ***word-count***，以及 `mrapps/indexer.go` 中的文本索引器。

```shell
$ cd ~/6.824
$ cd src/main
$ go build -race -buildmode=plugin ../mrapps/wc.go
$ rm mr-out*
$ go run -race mrsequential.go wc.so pg*.txt
$ more mr-out-0
A 509
ABOUT 2
ACT 8
...
```



## Your Job([moderate/hard](https://pdos.csail.mit.edu/6.824/labs/guidance.html))

你的工作是实现一个分布式的***MapReduce***，由两个程序组成：***Coordinator***和***Worker***。只有一个协调器进程，一个或多个工作进程并行执行。在一个真实的系统中，***Worker***会在一堆不同的机器上运行，但对于这个实验室，你将在一台机器上运行它们。***Worker***将通过***RPC***与***Coordinator***通信。每个工作进程都会向协调器请求一项任务，从一个或多个文件中读取任务的输入，执行任务，并将任务的输出写入一个或多个文件。***Coordinator***应注意***Worker***是否能在合理的时间内完成其任务（对于本实验，使用 10 秒），如果超时没完成应该将相同的任务分配给其他的***Worker***来完成。

***Coordinator***和***Worker***的***main***函数分别在`main/mrcoordinator.go` 和 `main/mrworker.go`中，不需要修改这两个文件。你需要把你的实现放在 `mr/coordinator.go`, `mr/worker.go`, 和 `mr/rpc.go`中。

以下是***work-count***应用程序运行代码的方法。首先`src/main`目录，进入确保 **word-count** 插件是全新构建的：

```shell
$ go build -race -buildmode=plugin ../mrapps/wc.go
```

 运行***Coordinator***进程

```sh
$ rm mr-out*
$ go run -race mrcoordinator.go pg-*.txt
```

`mrcoordinator.go` 的 `pg-*.txt` 参数是输入文件；每个文件对应一个“拆分”，并且是一个 ***Map*** 任务的输入。 `-race` 标志运行与其竞争检测器一起运行。

在一个或者多个终端中，启动一些***Worker***进程。

```
$ go run -race mrworker.go wc.so
```

