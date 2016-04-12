---
layout: post
date: 2016-04-12 17:17:10 +0800
title:  "Android 进程间通信"

---

Android 应用的线程间通信大多数情况下发生在同一进程内，并共享进程内存。然而，Android 平台也通过 binder 框架支持如 IPC 的跨进程通信；binder 框架在线程之间没有共享内存时负责管理数据的通信。

Android 平台下一般的 IPC 用例，由高层级的组件比如 Intent、Service 和 ContentProvider 实现。应用在使用它们时无需知道通信是发生在进程内还是进程间。但是，有时有必要对一个应用来定义一个更明确的通信模型，并更多地参与实际的通信。本文将通过以下三个方面介绍线程的跨进程通信：

* 同步和异步 RPC
* 通过 Messager 实现消息通信
* 用 ResultReceiver 返回数据

## Android RPC

IPC 由 Linux OS 处理；Linux OS 支持信号、管道、消息队列、信号量和共享内存这五种 IPC 实现方式。在 Android 的修改过的 Linux 内核中，以 binder 框架取代 Linux IPC 的一般实现方式，来实现进程间的 RPC 机制。RPC 机制具体包括以下几步：

* 数据和方法的编组（marshalling）
* 将已编组的信息传送给远程进程
* 在远程进程解组传过来的信息（unmarshalling）
* 将返回值传回发起进程

Android 框架和核心库通过 binder 框架和 AIDL(Android Interface Definition Language) 将进程间通信抽象出来。

### Binder

Binder 使得应用可以在属于不同进程的线程间传递函数和数据：服务端进程定义一个 **android.os.Binder** 类支持的远程接口，客户端进程的线程可以通过这个远程对象访问远程接口。

既传递函数也传递数据的远程程序调用被称为 transaction；客户端进程调用 transact 方法，然后服务端进程在 onTransact 方法内接收此调用。如***图 5-1***

![图 5-1](/resources/images/figure-5-1.png)

***图 5-1***

默认情况下，客户端线程调用 transact 方法是阻塞的，直到远程线程的 onTransact 方法完成执行。传送的数据包涵 **android.os.Parcel** 对象，Android 对 Parcel 对象进行了优化以便通过 Binder 跨进程传递。参数和返回值是以 Parcel 对象的形式传递的，它们可以是字面参数，也可以是实现 **android.os.Parcelable** 接口的自定义对象。Parcelable 接口的编组和解组比 Serializable 更高效。

onTransact 方法被来自 binder 线程池的一个线程执行。该线程池只处理来自其他进程的请求，并且最大容量为 16 个线程，所以，16 个远程调用可以在每个进程中并发的执行。这就需要远程调用的实现必须保证线程安全。

IPC 可以是双向的，即服务端进程可以向客户端进程发起远程调用。因此，可以建立两个进程的双向通信机制。我们将会看到，该机制对于异步 RPC 来说很重要。

***注意***如果服务端进程在执行 onTransact 方法时，调用 transact 方法向客户端进程发起一个远程调用，那么，客户端进程不会在 binder 线程接收此请求，而是会等待第一次 transaction 的完成。

### AIDL

当一个进程想要暴露给另一个进程访问时，就必须定义这种通信的合约；根本上说，就是服务端定义一个客户端可以调用的接口。描述该接口最简单也是最普遍的方式是用在 .aidl.file 文件内定义的 AIDL。AIDL 文件编译生成支持 IPC 的 Java 代码。Android 应用与生成的 Java 代码交互，但是应用只需知道接口即可。通信合约定义的过程如***图 5-2***

![图 5-2](/resources/images/figure-5-2.png)

***图 5-2 远程通信接口的构建***

生成的 Java 接口既包含在客户端应用里也包含在服务端应用里。接口文件定义了两个内部类：Proxy 和 Stub，以此来处理数据的编组和解组以及 transaction 本身。因此，AIDL 自动生成的代码包装了 binder 框架和通信合约。

![图 5-3](/resources/images/figure-5-3.png)

***图 5-3***

如***图 5-3***所示，客户端的 proxy 和服务端的 stub 代表两个应用来处理 RPC，以此允许客户端在本地调用方法，即使该方法运行在一个服务端进程（更准确的说，是服务端线程池里的 binder 线程）。服务端必须支持来自多个进程和线程的方法的并发执行来保证线程安全。

### 同步 RPC

	

