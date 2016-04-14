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

尽管远程程序调用在服务端是并发执行的，但是客户端发起调用的线程的行为却是同步或者阻塞的。当在 binder 线程上的远程调用执行完毕后（可能会将一个值返回给客户端），发起调用的线程才恢复执行。

让我们以一个只返回远程进程的线程名的实例来说明同步 RPC 以及它的影响。

第一步是在 .aidl 文件内定义接口，也就是通信合约。接口的描述包括了客户端进程可以在服务端进程调用的方法的定义：

{% highlight java %}
interface ISynchronous {
    String getThreadNameFast();

    String getThreadNameSlow(long sleep);

    String getThreadNameBlocking();

    String getThreadNameUnblock();
}
{% endhighlight %}

Proxy 和 Stub 内部类和 Java 接口是由 aidl 工具生成的；服务端进程重写 Stub 类以实现要支持的功能：

{% highlight java %}
private final ISynchronous.Stub mBinder = new ISynchronous.Stub() {
        CountDownLatch mLatch = new CountDownLatch(1);

        @Override
        public String getThreadNameFast() throws RemoteException {
            return Thread.currentThread().getName();
        }

        @Override
        public String getThreadNameSlow(long sleep) throws RemoteException { // Simulate a slow call
            SystemClock.sleep(sleep);
            return Thread.currentThread().getName();
        }

        @Override
        public String getThreadNameBlocking() throws RemoteException {
            mLatch.await();
            return Thread.currentThread().getName();
        }

        @Override
        public String getThreadNameUnblock() throws RemoteException {
            mLatch.countDown();
            return Thread.currentThread().getName();
        }
    };
{% endhighlight %}

在这里，所有的方法实现都返回服务端进程中执行的线程的名字，但是设置了不同程度的延迟。getThreadNameFast 方法立即返回结果，而 getThreadNameSlow 方法休眠一段客户端定义的时间间隔，getThreadNameBlocking 通过等待 CountDownLatch 被递减而发生阻塞，该递减必须等待另一个线程调用 getThreadNameUnblock 方法才能完成。

一个可以访问远程进程 binder 对象的客户端进程可以获取 Proxy 的实现并调用将被远程执行的方法：

{% highlight java %}
ISynchronous mISynchronous = ISynchronous.Stub.asInterface(binder);
String remoteThreadName = mISynchronous.getThreadNameFast();
Log.d(TAG,"result = "+remoteThreadName);
{% endhighlight %}

比如当方法的执行发生在远程进程的一个 binder 线程时，远程调用的结果会被打印为 result = Binder_1。

实现 ISynchronous 接口的 Proxy 被用来调用 binder 线程的远程方法。现在让我们看看使用 RPC 的几种方式：

* 远程调用耗时短的操作

&emsp;&emsp;对 mISynchronous.getThreadNameFast() 的调用返回的速度和运行时能处理这种通信的速度一样快，发起调用的客户端线程只是短暂的阻塞。必要时，来自一个或多个客户端的并发调用会使用多个 binder 线程；但是由于该实现很快会返回结果，binder 线程可以被高效的重复利用。

* 远程调用耗时长的操作

&emsp;&emsp;对  mISynchronous.getThreadNameSlow(long sleep) 的调用在将值返回给客户端之前会运行一段可设置的时间间隔。发起调用的客户端线程在这段时间间隔将会阻塞。

&emsp;&emsp;每个客户端调用都会占用一个 binder 线程很长的时间；其结果是，多个调用可能会耗尽 binder 线程池中的线程。在那种情况下，发起这种远程方法调用的下一个线程会将那个 transaction 放入一个 binder 队列等待，直到有一个 binder 线程可用。

* 调用阻塞方法

&emsp;&emsp;阻塞的线程，如 mISynchronous.getThreadNameBlocking() 所示，在远程方法执行完成之前也会一直阻塞客户端线程。如果多个客户端线程并发地调用服务端进程的阻塞方法，那么 binder 线程池的线程将会很快用完，从而导致其他的客户端线程无法获得远程调用的结果。如果由于阻塞，服务端没有可用的 binder 线程，那么就没有可用的 binder 线程去唤醒阻塞的线程。此时，服务端只能依赖于它自己内部的线程去进行唤醒操作，否则，服务端将不会处理任何的调用，所有等待服务端返回的客户端线程将会永远阻塞。

&emsp;&emsp;阻塞的 Java 线程一般是可以中断的，这就意味着另一个线程可以中断当前阻塞的线程使它完成执行操作。然而，处于客户端进程内的一个线程无法直接访问服务端的线程，所以无法中断远程线程。此外，正在等待同步 RPC 返回的客户端线程也不能捕捉并处理中断。

* 调用含有共享状态的方法

&emsp;&emsp;AIDL 使得客户端进程可以并发地执行服务端进程的方法。并发执行的一般规则为：接口实现负责线程安全。在以上的示例代码中，mISynchronous.getThreadNameBlocking 和 mISynchronous.getThreadNameUnblock 方法共享一个 CountDownLatch，但是没有保护它不被并发的线程访问。因此，一个客户端不能依赖 getThreadNameBlocking 来保持阻塞，直到它自己调用 getThreadNameUnblock。

***注意***一个客户端不能假定一个确定的同步 RPC 是耗时短的，因此就认为从 UI 线程发起调用是安全的；因为服务端进程的实现可能会随时间改变，从而对 UI 线程的响应造成负面影响。所以，用客户端的工作线程发起远程调用，除非你知道远程方法的执行并且它在你的控制之下。

### 异步 RPC

同步 RPC 的优势在于它很简单，易理解且易实现。但是简单是要付出代价的，因为发起调用的线程因此是阻塞的。当然，这也适用于本地进程的调用，但往往客户端的开发者对远程调用执行的代码一无所知。发起调用的线程的阻塞时间也会随着远程代码的实现的改变而改变。因此，同步 RPC 对应用的响应性会有不可预测的影响。一般通过在工作线程执行所有的远程调用来避免这种对 UI 线程的影响。然而，一旦服务端线程阻塞，客户端线程也会跟着阻塞，这就使得线程以及所有它引用的对象保持存活状态，从而导致内存泄漏。

对于异步 RPC，与同步 RPC 在客户端实现异步机制不同，异步 RPC 的远程调用方法被定义为异步方法。客户端初始化一个异步 RPC 的 transaction 并立即返回。Binder 负责将 transaction 交给服务端进程并关闭客户端和服务端的连接。

异步方法必须返回 void，其结果则由回调取回。

异步 RPC 是由 AIDL 的 oneway 关键词定义的，它既可以用于接口也可以用于单独的方法。

* 异步接口，所有的方法都会异步执行：

{% highlight java %}
oneway interface IAsynchronousInterface {
    void method1();

    void method2();
}
{% endhighlight %}

* 异步方法，该方法异步执行：

{% highlight java %}
interface IAsynchronousInterface {

    oneway void method1();

    void method2();
}
{% endhighlight %}

异步 RPC 最简单的形式是定义一个回调接口，它是一个反向的 RPC，如从服务端到客户端的调用。因此，回调接口也在 AIDL 文件内定义。

以下 AIDL 展示了异步 RPC 的一个简单示例，在这里，远程接口由一个包含回调接口的方法定义：

{% highlight java %}
interface IAsynchronous1 {
    oneway void getThreadNameSlow(IAsynchronousCallback callback);
}
{% endhighlight %}

服务端远程接口的实现如下，在方法的最后，结果在回调方法内返回：

{% highlight java %}
    IAsynchronous1.Stub mIAsynchronous1 = new IAsynchronous1.Stub() {
        @Override
        public void getThreadNameSlow(IAsynchronousCallback callback) throws RemoteException {
            // Simulate a slow call
            String threadName = Thread.currentThread().getName();
            SystemClock.sleep(10000);
            callback.handleResult(threadName);
        }
    };

{% endhighlight %}

AIDL 回调接口的声明如下：

{% highlight java %}
interface IAsynchronousCallback {
    void handleResult(String name);
}
{% endhighlight %}

客户端进程的回调接口的实现负责处理结果：

{% highlight java %}
private IAsynchronousCallback.Stub mCallback = new IAsynchronousCallback.Stub() {
        @Override
        public void handleResult(String remoteThreadName) throws RemoteException { // Handle the callback
            Log.d(TAG, "remoteThreadName = " + name);
            Log.d(TAG, "currentThreadName = " + Thread.currentThread().getName());
        }
    }
{% endhighlight %}

注意，这里远程和客户端的线程名都被打印为“Binder_1”，但是它们是分别来自客户端进程和服务端进程的不同的 binder 线程。异步的回调将会在一个 binder 线程上被接收。因此，如果回调的实现在客户端进程中与其他线程有共享数据，它应该确保线程安全。

## 用 Binder 进行消息传递






	

