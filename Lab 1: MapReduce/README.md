# Lab 1: MapReduce

***Lab1*** 主要是结合**MapReduce**论文，实现一个分布式的**MapReduce**

## Link

[Lab 1: MapReduce](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)

## MapReduce论文

[英文版](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)

[中文版](http://aleda.cn/papers/DistributedSystem/mapreduce-osdi04-cn.pdf)

## Introduction

本实验中，你将构建一个 ***MapReduce*** 系统。你将实现一个用于调用应用程序 ***Map()*** 和 ***Reduce()*** 函数并处理读取和写入文件的***Worker process***（工作进程），以及一个将任务分发给***Worker***并处理***Worker***失败的***Coordinator process***（协调员进程）。你将构建一个类似于 ***[MapReduce](http://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)*** 论文中的内容。（注意：**Lab**中使用***Coordinator***来代替论文中的***Master***）



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

我们为您提供了一个简单的***sequential***(串行执行而不是并发的) ***MapReduce*** 实现在`src/main/mrsequential.go`中。它只在一个进程中按照流水线的顺序执行***Map()*** 和 ***Reduce******()*** 函数。我们还为您提供了一些 ***MapReduce*** 应用程序：`mrapps/wc.go` 中的 ***word-count***，以及 `mrapps/indexer.go` 中的**indexer**。您可以按照以下方式来运行***word-count***

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

（注意⚠️⚠️：如果编译.so时没用使用`-race`参数，将不能使用`-race`参数运行）

`mrsequential.go `将其输出写入文件 `mr-out-0` 中。输入来自名为`pg-xxx.txt`的文本文件。

可以随意借鉴`mrsequential.go`中的代码。您还应该查看`wrapps/wc.go`以了解一个***MapReduce***应用程序的大致样子。

## Your Job([moderate/hard](https://pdos.csail.mit.edu/6.824/labs/guidance.html))

你的工作是实现一个分布式的***MapReduce***，由两个程序组成：***Coordinator***和***Worker***。只有一个**Coordinator Process**，一个或多个**Worker Process**并行执行。在真实的系统中，**Worker**会在一堆不同的机器上运行，但对于这个实验，你将在一台机器上运行它们。**Worker**将通过***RPC***与***Coordinator***通信。每个**Worker**都会向Coordinator请求一项任务，从一个或多个文件中读取任务的输入，执行任务，并将任务的输出写入一个或多个文件。***Coordinator***应注意***Worker***是否能在合理的时间内完成其任务（对于本实验，使用 10 秒），如果超时没完成应该将相同的任务分配给其他的***Worker***来完成。

提供了一些代码来让你开始，***Coordinator***和***Worker***的***main***函数分别在`main/mrcoordinator.go` 和 `main/mrworker.go`中，不需要修改这两个文件。你只需要把你的实现放在 `mr/coordinator.go`, `mr/worker.go`, 和 `mr/rpc.go`中。

以下是***work-count***应用程序运行代码的方法。首先入`src/main`目录，进入确保 **word-count** 插件是全新构建的：

```shell
$ go build -race -buildmode=plugin ../mrapps/wc.go
```

 运行***Coordinator***进程

```sh
$ rm mr-out*
$ go run -race mrcoordinator.go pg-*.txt
```

`mrcoordinator.go` 的输入参数 `pg-*.txt` 代表文件；每个文件对应一个“split”，并且是一个 ***Map*** 任务的输入。 `-race` 标志运行与其竞争检测器一起运行。

在一个或者多个终端中，启动一些***Worker***进程。

```shell
$ go run -race mrworker.go wc.so
```

当***Coordinatoe***和***Worker***进程运行完成后，输出文件在`mr-out-*`。当你完成实验后，输出文件的排序并集应与`mrsequential.go`的输出相匹配，如下所示：

```shell
$ cat mr-out-* | sort | more
A 509	
ABOUT 2
ACT 8
...
```

我们在 `main/test-mr.sh` 中为你提供了一个测试脚本。当使用`pg-xxx.txt`文件作为输入时，检查**word-count**和**indexer**两个**MapReduce**应用程序是否产生正确的输出。这些测试还会检查您的实现是否并行运行 **Map** 和 **Reduce** 任务，以及你的实现是否从在运行任务时崩溃的 **worker** 中恢复。

如果您现在运行测试脚本，它将挂起，因为**Coordinator**永远不会完成：

```shell
$ cd ~/6.824/src/main
$ bash test-mr.sh
*** Starting wc test.
```

您可以在 `mr/coordinator.go` 中的 `Done()` 函数中将 `ret := false`更改为 **true**，以便协调器立即退出。然后：

```shell
$ bash test-mr.sh
*** Starting wc test.
sort: No such file or directory
cmp: EOF on mr-wc-all
--- wc output is not the same as mr-correct-wc.txt
--- wc test: FAIL
$
```

测试脚本期望在名为 **mr-out-X** 的文件中看到输出，每个 **reduce** 任务一个。 `mr/coordinator.go` 和 `mr/worker.go` 的空实现不生成这些文件（或做很多其他事情），所以测试失败。

完成后，测试脚本输出应如下所示：

```shell
$ bash test-mr.sh
*** Starting wc test.
--- wc test: PASS
*** Starting indexer test.
--- indexer test: PASS
*** Starting map parallelism test.
--- map parallelism test: PASS
*** Starting reduce parallelism test.
--- reduce parallelism test: PASS
*** Starting crash test.
--- crash test: PASS
*** PASSED ALL TESTS
$
```

您还会看到 Go RPC 包中的一些错误，如下所示

```
2019/12/16 13:27:09 rpc.Register: method "Done" has 1 input parameters; needs exactly three

```

忽略这些消息，将协调器注册为 RPC 服务器检查其所有方法是否适用于 RPC（有 3 个输入）；我们知道 Done 不是通过 RPC 调用的。

## A few rules:

+ ***Map()***阶段应该将生成的中间**Key**划分为**nReduce**份，放入**nReduce**个桶中。其中 **nReduce** 是 `main/mrcoordinator.go` 传递给 `MakeCoordinator()` 的参数。
+ ***Worker*** 应该将第 ***X*** 个 ***reduce*** 任务的输出放在文件 **mr-out-X** 中。
+ **mr-out-X** 文件中每个Reduce函数输出应包含一行。该行应使用 Go `"%v %v" `格式生成，使用键和值调用。查看 `main/mrsequential.go `中注释`“this is the correct format”`的行。如果您的实现与此格式偏差太大，则测试脚本将失败。
+ 您可以修改`mr/worker.go`、`mr/coordinator.go`、`mr/rpc.go`。也可以临时修改其他文件进行测试，但请确保您的代码适用于原始版本；我们将使用原始版本进行测试。
+ ***Worker*** 应该将中间 **Map** 输出放在当前目录中的文件中，您的***Worker*** 稍后可以在其中读取它们作为 **Reduce** 任务的输入。
+ `main/mrcoordinator.go`期望 `mr/coordinator.go` 实现一个 `Done() `方法，该方法在 `MapReduce` 完全完成时返回 **true**，在那时，`mrcoordinator`将退出。
+ 当工作完全完成时，**Worker**应该退出。一个简单事实现方法是使用call的返回值：如果 **worker** 无法联系协调器，则可以假设协调器已退出，因为工作已完成，因此 **worker** 也可以终止。根据您的设计，您可能还会发现有一个“请退出”的伪任务，协调器可以提供给工作人员。

## Hints

+ 一种上手方法是修改`mr/worker.go`的Worker()，向***Coordinator***发送一个RPC来请求任务

+ 应用程序 **Map** 和 **Reduce** 函数在运行时使用 Go 插件包从名称以 **.so** 结尾的文件中加载。

+ 如果您更改了 `mr/ `目录中的任何内容，您可能必须重新构建您使用的任何 **MapReduce** 插件，例如 `go build -race -buildmode=plugin ../mrapps/wc.go`

+ 本实验依赖于共享文件系统的工作人员。当所有工作人员都在同一台机器上运行时，这很简单，但如果工作人员在不同的机器上运行，则需要像 GFS 这样的全局文件系统。

+ 中间文件的合理命名约定是**mr-X-Y**，其中**X** 是**Map** 任务编号，**Y** 是**reduce** 任务编号。

+ **worker** 的 **map** 任务代码需要一种方法来将中间键/值对存储在文件中，以便在 **reduce** 任务期间可以正确读回。一种可能性是使用 Go 的 encoding/json 包。将键/值对写入 JSON 文件：

  ```go
  enc := json.NewEncoder(file)
    for _, kv := ... {
      err := enc.Encode(&kv)
  ```

  读取时：

  ```go
  dec := json.NewDecoder(file)
    for {
      var kv KeyValue
      if err := dec.Decode(&kv); err != nil {
        break
      }
      kva = append(kva, kv)
    }
  ```

  

+ **worker** 的 **map** 部分可以使用 `ihash(key)` 函数（在 `worker.go` 中）为给定的键选择 reduce 任务。

+ 您可以从 `mrsequential.go` 窃取一些代码，用于读取 **Map** 输入文件、对 **Map** 和 **Reduce** 之间的中间键/值对进行排序，以及将 **Reduce** 输出存储在文件中。

+ 协调器作为一个 RPC 服务器，将是并发的；不要忘记锁定共享数据。

+ 使用 Go 的比赛检测器，使用 `go build -race` 和 `go run -race`。` test-mr.sh `默认使用竞争检测器运行测试。

+ 有时需要等待，例如在最后一张地图完成之前，**reduce** 无法启动。一种可能性是工作人员定期要求协调员工作，在每个请求之间与 `time.Sleep()` 一起睡觉。另一种可能性是协调器中的相关 RPC 处理程序具有等待的循环，使用 `time.Sleep() `或` sync.Cond`。Go 在其自己的线程中为每个 RPC 运行处理程序，因此一个处理程序正在等待这一事实不会阻止协调器处理其他 RPC。

+ 协调器无法可靠地区分崩溃的工作线程、活着但由于某种原因停止的工作线程以及正在执行但速度太慢而无用的工作线程。您能做的最好的事情是让协调员等待一段时间，然后放弃并将任务重新发布给不同的工作人员。对于这个实验室，让协调员等待十秒钟；之后协调员应该假设工人已经死亡（当然，它可能没有）。

+ 如果您选择实施备份任务（Section 3.6），请注意，我们测试了当工作人员执行任务而不会崩溃时，您的代码不会安排无关的任务。备份任务只能在相对较长的时间段（例如 10 秒）后安排。

+ 要测试崩溃恢复，您可以使用 `mrapps/crash.go` 应用程序插件。它在 Map 和 Reduce 函数中随机退出。

+ 为了确保没有人在出现崩溃的情况下观察到部分写入的文件，MapReduce 论文提到了使用临时文件并在完全写入后自动重命名的技巧。您可以使用 `ioutil.TempFile` 创建一个临时文件，并使用 `os.Rename` 对其进行原子重命名。

+ `test-mr.sh` 运行子目录 mr-tmp 中的所有进程，因此如果出现问题并且您想查看中间文件或输出文件，请查看那里。您可以修改 `test-mr.sh` 以在测试失败后退出，因此脚本不会继续测试（并覆盖输出文件）。

+ `test-mr-many.sh` 提供了一个简单的脚本来运行 `test-mr.sh` 超时（这是我们将如何测试您的代码。它将运行测试的次数作为参数。您不应并行运行多个 `test-mr.sh` 实例，因为协调器将重用同一个套接字，从而导致冲突。

