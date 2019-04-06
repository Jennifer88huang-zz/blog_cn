# Go in Pulsar
本章主要从 CGO Client，Go Client 和 Pulsar Go Function 几个方面来介绍 Go 在 Pulsar 中的实现。

## 1. CGO Client

关于 CGO Client，我们主要讨论如下几个话题，build error 的成因、CGO 与 Go 的区别、CGO 的性能损耗和 debug 问题以及 CGO 使用感悟和 CGO 的版本迭代。


### 1.1 又是 build error？

![image.png](https://upload-images.jianshu.io/upload_images/6967649-3b015cc869a0abc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上简单罗列了大家使用 CGO 版本 pulsar-client-go 时在 build 中遇到的问题。由于 CGO 自身原因，社区人员对快速构建并使用 pulsar-client-go 存在很多困扰。目前 pulsar-client-go 是基于 CGO 封装的，它的运行依赖于 pulsar-client-cpp，而很多 gopher 因为对 cpp 缺乏经验而无法快速上手 pulsar-client-go。

### 1.2 CGO 不是 Go

Go 语言最初的定位是“下一代 C 语言”，所以在 Go 诞生时继承了部分 C 的遗产。除了语法设计上与 C 有一些相似之处外，为使用 C 语言的代码，Go 语言还打通了与 C 语言的通道，进行了混合编程。

但需要注意的是，首先， CGO 本身和 Go 语言本身没有太大关系，CGO 只是在编译层面提供了一种连接 Go 和 C 的工具。Go 语言本身没有定义和 C 进行交互的方式。因此，CGO 更像一种工具，而非语言的扩展。

另一方面，Go 语言支持交叉编译，可编译出不同平台的可执行文件，而 CGO 不支持这一功能。

### 1.3 性能损耗

性能问题一直是 CGO 的短板。调用普通 Go 函数不同于调用 CGO 函数，需通过类似系统调用（runtime.CGOcall）的形式进行交互，而这种调用方式会造成一定的性能损耗。Go 的内存模型和 C 的内存模型是不一样的，它需要把 gorutine 的调用栈翻译成 C 的线程再去执行，而转换过程无可避免地会造成一定损耗。在正常编程时，我们会尽量避免系统调用。

Go 代码是有调度的，有可能执行一段代码后调度到别的任务去执行；但是 C 不识别 Go 的调度器。假设调用 C 时，C 代码需要执行较长的时间，这个时候调度器会认为 M 处于闲置状态，可以把 P 让出来，系统这时候会新建一个 M 和这个 P 配对执行别的任务。这时 M 处于休眠状态，但是 C 的代码迟迟不能返回，便可能导致 M 的数量在某个时间段内陡增。

另一个问题是 C 与 Go 的内存管理方式也存在差异，C 是手动管理，而 Go 是带 gc 的语言，可以进行自动管理。如果要使用 CGO，需注意尽量减少它与 C 之间的调用。

### 1.4 随缘 debug

CGO 代码的 debug 是个大问题。Go 无法直接调用 C 代码，所以在遇到问题时只能先将代码回退到某一个可以正常运行的阶段，再一行行复原代码，看是否按照预期的结果进行输出。不然，“print大法”也是不错的选择。

### 1.5 使用 CGO 的感悟

- CGO 有自己的一套类型系统，所以在使用时需要了解这两种语言类型之间的转换规则。在 Go 语言中访问 C 语言的符号时，一般通过虚拟的“C”包访问，比如 C.int 对应 C 语言的 int 类型。
- 如果 C 代码比较简单，可以放在 Go 文件里直接使用 go run 来执行。但是当调用的 C 文件较多或 C 为一个独立文件时，需使用 go build 来编译执行。使用 go build 编译后，代码会与 C 生成的链接库进行链接执行。在机器层面，使用哪种语言并不重要，只要是可识别的机器码就行。
- unsafe.Pointer 是 Go 指针和 C 指针转换的中介。
- 上面提及 C 和 Go 的内存管理方式不一样，所以在进行内存分配时，一定要明确这段代码的内存分配是在 C 上还是 Go 上，也就是要明白这段内存究竟谁是管理者。归 C 管则需要手动 free，不然会造成内存泄漏；归 Go 管则无需担心，垃圾回收会完成处理。
- 最后总结一句，一定要处理好 Go 与 C 之间的边界问题。

### 1.6 CGO 版本迭代

#### 1.5


```
$ CGO {SRCDIR} 展开为源目录
```

如果 C struct 以零长度字段结束，但自身长度不为零，则 Go 不能改字段。

#### 1.6

C 代码共享 Go 指针，确保与 gc 共存。

新增 C.complexfloat(complex64)、C.complexdouble(complex128)

#### 1.7

新函数 C.Cbytes

#### 1.8

环境变量 `PKG_CONFIG` 用于设置 `#cgo pkg-config` 指令。

CGO 命令支持 `-srcddir` 参数。

CGO 代码调用 C.malloc，返回 NULL 将导致进程奔溃。

> pkg-config 用于查找和输出库路径

#### 1.10

支持类型别名

支持直接从 C 访问 Go 字符串

#### 1.11

macOS/ios 内核调用，通过 libSystem.so


上述是从1.5到1.11几个版本中 CGO 的更新，可以看到变动非常小。

## 2. Go Client

Go 语言自身的优势以及 Go 社区的不断壮大，带动了社区对 Go Client 的大量需求。目前 Pulsar 社区向大家提供了基于 CGO 封装的 pulsar-client-go，上述提到的这些问题导致 pulsar-client-go 对 Go 社区还不够友好，这也促使我们要基于原生的 Go 语言对 pulsar-client-go 进行封装。

在此特别感谢 ComCast 向社区贡献 Go 版本的 pulsar-client-go，我们目前正是基于这个版本在进行功能的 catch up 工作。Comcast 没有完全实现该客户端，因此还不能支持 Pulsar 的所有功能。目前它支持到 Pulsar 2.0。

下面是一些还没有实现的主要功能：

* Batch frame support
* Payload compression support
* Partitioned topics support
* Athenz authentication support
* Encryption support

在 fork 到 wolfstudy 的仓库下之后，我们对 ComCast 的架构进行了调整，方便项目的管理。ComCast 最初并没有对项目进行 package 结构的管理，这种方式不会遇到 Go 中相对恶心的一个问题—-包之间的循环引用，但是后期如果开发功能过多，整个项目结构会非常混乱，对社区不够友好。

![image.png](https://upload-images.jianshu.io/upload_images/6967649-6f27f261d6fad275.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

目前我们将需要实现的一些功能以 issue 的形式罗列了出来，并且秉持最小化开发的原则，将每个任务尽量细化，方便大家参与。大家在参与的时候，可以在自己感兴趣的功能对应的 issue 下添加相应的 comments，避免重复工作。欢迎大家 star 参与。

![image.png](https://upload-images.jianshu.io/upload_images/6967649-5f26cf7ef5114edc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3. Pulsar Go Function
在介绍 Pulsar Go Function 之前，我们可以首先了解一下 Pulsar Function 的背景知识。

### 3.1 What is Pulsar Function

首先我们来看一下什么是 function。我们把 function 定义为一个轻量级的计算框架，让计算这件事情变得更简单。用户只需关心计算逻辑，剩下的事情由 Pulsar Function 处理。

![image.png](https://upload-images.jianshu.io/upload_images/6967649-afe6dbab164add85.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以看到，Pulsar Function 从 input topics 中消费数据，将消费到的数据传递给 Pulsar Function 做处理。处理完后，将处理的结果写到 output topic，而 log topic 则负责处理用户使用 Pulsar 时生成的日志。每条消息到来时，Pulsar Function 会被调用一次。

### 3.2 Why Pulsar Function

Pulsar Function 有两大优点：
1. 部署简单
    - Localrun（本地运行一个函数，适用于开发者）
    - managed-worker service来运行和管理functions
    - k8s：每个 function 等价于一个 k8s 的 statefulset，利用 k8s 进行弹性扩展

2. 运维简单、接口简单，用户可以快速上手，解放运维，还运维人员一片晴朗的蓝天。


### 3.3 使用场景

只需要单个事件的处理、单个操作能完成的事情，我可以使用 Pulsar Function 来做，例如：

- ETL
- Data Enrichment
- Data Filtering
- Routing

### 3.4 Function Server


![image.png](https://upload-images.jianshu.io/upload_images/6967649-99d0e2da2507c9a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1. 用户给 REST server 发送一个请求
1. REST server 响应用户的请求
1. Function Metadata Manage 将更新写到FMT
1. Function Metadata Manager 从 FMT 中读取更新
1. Scheduler Manager 将更新写到 assignment topic
1. Function Runtime Manager 从 assignment topic 中读取更新
1. Membership Manager 配合 Coordination Topic 来做 leader 的选举
1. Membership Manager 配合 Coordination Topic 保证 active membership 的资格

我们拿一个例子来说明：

1. 用户提交一个请求到 REST sever 来执行一个 function 的实例。因为 function 的分配不依赖于任何 worker，所以这个请求可能被提交到任意的一个 worker 来处理。
2. REST server 将请求传递给 Function Metadata Manager，Function Metadata Manager 将该请求写入 Function Metadata Topic (FMT)。
3. Function Metadata Manager 会监听所有新进入 FMT 的 message。当 Function Metadata Manager 从 FMT 收到一个新的 message 时，首先会去检查消息是否过期，如过期则直接丢弃。如没过期，Function Metadata Manager 使用该消息更新其内部状态，其中包含正在运行的所有 function 的全局视图。由于每个工作程序都运行一个 Function Metadata Manager，因此每个 worker 都有一个最终一致的全局视图，其中包含所有正在运行的函数的状态。因此，可以将读取请求（例如获取正在运行的函数的状态的请求）提交给任何工作者。
4. 当 Function Metadata Manager 更新其内部状态时，会去触发 Scheduler Manager。因为这个时候系统有新的更新进来，必须对这个新的更新进行计算。这时候，处于 leader 状态的 worker 执行调度策略，看这一次的计算分配给谁执行比较合适，然后将新的分配写入到 Assignment Topic。 Membership Manager 用于维护集群中的 leader 以及所有处于 active 状态成员的列表。
5. Function Runtime Manager 会去监听 Assignment Topic 看是否有新的更新。当有更新进来时，Function Runtime Manager 将更新其内部状态，其中包含所有 worker 的全局视图。如果有更新，Function Runtime Worker 会根据这个更新，判断是否需要 start 或者 stop function 的实例。 

## 4. Pulsar Go Function实现

Pulsar Function 的 server 端和 client 端使用 protobuf 进行了解耦，原则上 protobuf 所能支持的语言，Pulsar Function 都可以支持。这大大简化了我们开发 Go Function 的工作，我们无需关心 runtime 和 worker 发生了什么事情。我们把要实现的 Go 语言的 instance 当作 client 端，两者中间通过 protobuf 协议进行交互。我们只需要根据预先定义好的 proto 文件，生成 Go 语言的 pb 文件，在此基础上封装一个 Go Function client 端，暴露给外部用户使用即可。之前 Pulsar Function 已支持 Java 和 Python，但是随着 Go 社区的壮大，对于 Go Function client 的需求也越来越多，所以我们急需 Go Function Client。这样，一方面可以丰富 Pulsar Function 的 client 端；同时也可方便熟悉 Go 语言的人快速了解、使用 Pulsar Function。


### 4.1 SDK实现思路

当我们以 SDK 的形式提供给用户时，用户只需要传入一个函数的名字，我们根据接收到的 function name 利用 Go 的反射来验证用户实现的参数和返回值列表是否正确。之后，我们将验证的结果以 handler 形式返回。拿到具体需要处理的 handler 之后，我们启动 Pulsar Client，创建消费者从 input topics 中去消费相应的 record，go func 处理完成之后，通过 producer 将结果输出到 output topic。

### 4.2 与 Java, Python Instance 的区别

Pulsar 首先向用户暴露一个 Function 的接口。

```
type Function interface {
   Process(input []byte, ctx Context) ([]byte, error)
}
```

用户拿到这个接口后，去实现相应的代码逻辑，如最简单的 wordcount。之后，内部的一个 instance 的主代码逻辑需要动态的将用户的这段 process 逻辑反射进来，作为主体代码的逻辑来执行。

比如：在 Python Function 中可以通过`getattr(mod, class_name)`拿到 Function 的引用。 在Java Function 中可以通过动态类加载来实现。那么，这个“反射”机制对应到 Go 中应该如何实现？

Go 是一门静态类型的语言，本身不支持动态反射，所以我们没办法像 Java 和 Python 一样把一段代码内嵌到 instance 中执行。虽然 Go 1.8 已经通过添加 plugin 来支持动态加载，但它仍不成熟且只支持 linux 平台。所以，我们不妨换个思路来实现。

因此，在最终版本的 Go Function 实现中，我们把 Pulsar 实现的 Go Function Instance 也当作一个单独的 SDK 提供给用户，用户通过调用 Pulsar Go Instance SDK 将自己编写的 Function 注册进来。

### 4.3 SDK example

```
import (
	"context"
	"fmt"

	"github.com/apache/pulsar/pulsar-function-go/pf"
)

func contextFunc(ctx context.Context) {
	if fc, ok := pf.FromContext(ctx); ok {
		fmt.Printf("function ID is:%s, ", fc.GetFuncID())
		fmt.Printf("function version is:%s\n", fc.GetFuncVersion())
	}
}

func main() {
	pf.Start(contextFunc)
}
```


