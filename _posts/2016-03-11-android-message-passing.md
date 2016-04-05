---
layout: post
date: 2016-03-11 14:13:10 +0800
title:  "Android 的消息传递"

---

到目前为止，关于线程间通信的讨论都是在常规的 Java 语言中，在任何一个 Java 应用中都可以见到。由于 Android 应用中 UI 主线程的阻塞倾向，在将管道，共享内存以及阻塞队列这些机制应用于 Android 应用时，会引起很多问题。当在使用具有阻塞行为的机制时，UI 主线程的响应性会受到威胁，并且很有可能导致其被挂起。

在 Android 应用中，最常见的线程间通信发生在 UI 主线程和工作线程之间。因此，Android 平台定义了自己的消息传递机制来达到线程间通信的目的。UI 主线程可以通过将要处理的数据信息发送给后台线程的方式将耗时的任务移交给后台线程处理。该消息传递机制是一种非阻塞的生产者-消费者模式，因此在消息传递期间，生产者线程和消费者线程都不会发生阻塞。

这种机制在 Android 平台中是一种基本的消息处理机制，相关 API 和实现的一系列类位于 **android.os** 包中，见**图4-4**：

![图4-4](/resources/images/figure-4-4.png)

***图4-4***

**android.os.Looper**

&emsp;&emsp;消息的分发者，与有且只有一个消费者线程相联系。

**android.os.Handler**

&emsp;&emsp;消费者线程的消息处理者，并且作为生产者线程向消息队列中插入消息的接口。一个 Looper 可以与多个 Handler 相联系，但是它们插入消息的消息队列是同一个。

**android.os.MessageQueue**

&emsp;&emsp;在消费者线程上处理的消息的无界链表。每一个 Looper 和线程至多拥有一个 MessageQueue.

**android.os.Message**

&emsp;&emsp;消费者线程上执行的的消息类。

生产者线程插入消息以及消费者线程处理消息的过程如**图4-5**所示：

![图4-5](/resources/images/figure-4-5.png)

***图4-5 展示了多个生产者线程与一个消费者线程之间的消息传递***

1. 插入消息：生产者线程通过与消费者线程连接的 Handler 将消息插入消息队列。
2. 取回消息：运行在消费者线程的 Looper 将消息从消息队列中顺序的取出。
3. 分发消息：Handler 负责处理运行在消费者线程的消息。一个线程可以拥有多个用来处理消息的 Handler 实例；Looper 则保证消息能够分发给正确的 Handler。

## 示例：简单的消息传递

在我们详细的剖析各个组件之前，让我们先通过一个简单的示例来熟悉代码。

接下来的代码实现可能是最常见的消息传递用例之一。用户按下屏幕上的一个按钮，触发一个耗时操作，比如网络请求。为了避免阻塞 UI 的渲染，该耗时操作（此例中用 doLongRunningOperation() 方法代表）必须在工作线程上执行。因此，我们这里只有一个生产者线程（ UI 线程 ）和一个消费者线程（LooperThread）。

这里我们设置了一个消息的队列，它像平常一样在 onClick() 回调中处理按钮点击事件，该操作发生在 UI 线程。在我们的实现中，该回调向我们的消息队列中插入了一条伪消息。简单起见，此处省略了布局和控件相关的代码。

{% highlight java %}
public class LooperActivity extends Activity {

    LooperThread mLooperThread;
    
    private static class LooperThread extends Thread { //1.
        public Handler mHandler;
        public void run() {
            Looper.prepare(); //2.
            mHandler = new Handler() { //3.
            
            	@Override
                public void handleMessage(Message msg) { //4.
                    if (msg.what == 0) {
                        doLongRunningOperation();
                    }
                }
                
            };
            Looper.loop(); //5.
        }
    }
    
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mLooperThread = new LooperThread(); //6.
        mLooperThread.start();
    }

    public void onClick(View v) {
        if (mLooperThread.mHandler != null) { //7.
            Message msg = mLooperThread.mHandler.obtainMessage(0); //8.
            mLooperThread.mHandler.sendMessage(msg); //9.
        }
    }

    private void doLongRunningOperation() {
    	 // Add long running operation here.
    }

    protected void onDestroy() {
        mLooperThread.mHandler.getLooper().quit(); //10.
    }
}
{% endhighlight %}

1. 定义工作线程，作为消息队列的消费者。
2. 把一个 Looper 和一个隐式的 MessageQueue 与线程联系起来。
3. 创建一个 Handler 用来向消息队列插入消息。这儿我们用的是默认构造函数，因此它对当前线程的 Looper 一无所知。所以，这个 Handler 只能在 Looper.prepare() 之后创建。
4. 当消息分发给工作线程时，执行此回调。它检查 what 参数，然后执行耗时操作。
5. 开始将消息队列里的消息分发给消费者线程。这个调用是阻塞的，所以工作线程不会结束。
6. 启动工作线程，准备处理消息。
7. 因为在工作线程中创建 mHandler 和在 UI 线程使用它之间存在竞争条件，所以需要判断 mHandler 是否有效。
8. 用一个设为0的 what 参数初始化一个消息对象。
9. 向消息队列中插入消息。
10. 终结工作线程。Looper.quit() 能够终止消息的分发并且使 Looper.loop() 从阻塞中释放出来，因此，run() 方法可以结束执行，从而导致线程终止。

##  消息传递中使用到的类

现在让我们更细致的观察消息传递中特定的组件以及它们的使用。

### MessageQueue

消息队列对应于 **android.os.MessageQueue** 类。它由一个无界的单向链表实现。生产者线程插入的消息不会立即分发给消费者线程。这些消息是以时间戳排序的。在等待队列中，拥有最小时间戳值的消息最先分发给消费者线程；而且，当且仅当该时间戳小于当前的时间，这条消息才会被分发。如果不是这样，分发操作会一直等到当前时间经过时间戳后才会发生。

***图4-6***表示了一个拥有三条排队消息的消息队列，并按照时间戳 t1 < t2 < t3 的顺序排序，其中只有一条消息满足了被分发的条件。一条消息的时间戳只有小于当前时间才有资格被分发。（图中用“ Now ” 表示）

![图4-6](/resources/images/figure-4-6.png)

***图4-6 队列中的消息。最右边的消息最先被处理，每条消息上的箭头用来表示对它下一条消息的引用***

当 Looper 准备取回下一条消息但是消息队列中没有消息满足被分发的条件时，消费者线程将会阻塞。一旦消息满足分发条件，线程即可恢复执行。

生产者线程可以在任何时间将新消息插入消息队列的任何位置，插入的位置基于时间戳。如果新消息的时间戳比消息队列里的所有消息的时间戳都小，那么它将占据队列的第一个位置（也是下一个即将被分发消息的位置）。消息的插入操作总是遵循时间戳排序。

### MessageQueue.IdleHandler

如果没有消息要处理，消费者线程会有一定的空闲时间。比如，***图4-7***表示了消费者线程处于空闲状态的一个时间段。默认情况下，消费者线程在空闲状态下会简单地等待新消息的到来；但是除了等待，它还可以在这些空闲时间内执行其他的任务从而使其得到充分利用。这个特性可以让那些不重要的任务推迟执行，直到其他的消息不再争夺执行时间。

![图4-7](/resources/images/figure-4-7.png)

***图4-7 如果没有消息满足被分发的条件，在下一条排队消息执行之前，该空闲时间段可以用来执行其他任务***

当一条消息被分发，并且没有其他的消息满足被分发的条件，就会形成一个空闲时间段。Android 应用通过 **android.os.MessageQueue.IdleHandler** 接口来持有这段时间。该接口监听消费者线程，当消费者线程空闲时生成回调。通过以下这些调用，该接口与 MessageQueue 建立或者中断关系:

{% highlight java%}
// Get the message queue of the current thread.MessageQueue mq = Looper.myQueue();// Create and register an idle listener.MessageQueue.IdleHandler idleHandler = new MessageQueue.IdleHandler();  mq.addIdleHandler(idleHandler)// Unregister an idle listener.mq.removeIdleHandler(idleHandler)
{% endhighlight %}

IdleHandler 接口只包含一个回调方法：

{% highlight java%}
interface IdleHandler { 
    boolean queueIdle();}
{% endhighlight%}

当消息队列监测到消费者线程有空闲时间时，它会调用所有已注册 IdleHandler 实例的 queueIdle() 方法。回调的实现由应用负责，你应该尽量在回调中避免耗时的任务，以免推迟等待消息的执行。

queueIdle() 的实现必须返回一个布尔值，其意义如下：

**true**

&emsp;&emsp;IdleHandler 保持活跃，它将继续接收空闲时间段的回调。

**false**

&emsp;&emsp;IdleHandler 不再活跃，不再接受任何回调。这和调用 MessageQueue.removeIdleHandler() 的效果是一样的。

### 示例：用 IdleHandler 终结无用线程

当线程在等待新消息时出现空闲时间段，那么所有向消息队列注册过的 IdleHandler 都会被调用。空闲时间段可能发生在第一条消息之前，两条消息之间或者最后一条消息之后。如果有多个内容生产者在消费者线程上顺序的处理数据时，IdleHandler 可以在所有的消息得到处理后终结消费者线程，从而使无用的线程不在占据内存。有了 IdleHandler 的帮助，就没有必要跟踪插入的最后一条消息以及何时可以终结线程了。

***注意：***此用例只适合于当生产者线程向消息队列插入消息没有延迟，从而使得消费者线程在最后一条消息插入之前不会产生空闲时间段这种情况。

下面的 ConsumeAndQuitThread 展示了当没有新消息要处理时，终结线程的情况。

{% highlight java%}
public class ConsumeAndQuitThread extends Thread implements MessageQueue.IdleHandler {
    private static final String THREAD_NAME = "ConsumeAndQuitThread";
    public Handler mConsumerHandler;
    private boolean mIsFirstIdle = true;

    public ConsumeAndQuitThread() {
        super(THREAD_NAME);
    }

    @Override
    public void run() {
        Looper.prepare();
        mConsumerHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) { // Consume data
            }
        };
        Looper.myQueue().addIdleHandler(this);//1.
        Looper.loop();
    }

    @Override
    public boolean queueIdle() {
        if (mIsFirstIdle) {//2.
            mIsFirstIdle = false;
            return true;//3.
        }
        mConsumerHandler.getLooper().quit();//4.
        return false;
    }

    public void enqueueData(int i) {
        mConsumerHandler.sendEmptyMessage(i);
    }
}
{% endhighlight%}

1. 将 IdleHandler 注册到后台线程，以使当线程和 Looper 准备好时消息队列已经设置好。
2. 让 queueIdle 的第一次调用通过，因为它发生在第一条消息到来之前。
3. 返回 true 使得 IdleHandler 通过第一次调用后依然保持注册状态。
4. 终结线程。

消息的插入是由并发的多个线程通过模拟的随机插入时间完成的：

{% highlight java%}
final ConsumeAndQuitThread consumeAndQuitThread = new ConsumeAndQuitThread();
consumeAndQuitThread.start();
for(inti=0;i<10;i++){
    new Thread(new Runnable(){
    @Override
    public void run(){
        for(inti=0;i<10;i++){
            SystemClock.sleep(new Random().nextInt(10));
            consumeAndQuitThread.enqueueData(i);
        }
    }
    }).start();
}
{% endhighlight%}

### Message

在消息队列中的每个 item 都属于 **android.os.Message** 类。这是一个携带着一个数据项或者任务项的容器对象，但不会兼而有之。数据项由消费者线程处理，而任务项只是在出列时简单的被执行，如果你没有其他处理要做的话。

Message 知晓它的接受者，比如 Handler。而且可以通过 **Message.sendToTarget()** 将自己加入到消息队列中：

{% highlight java%}
Message m = Message.obtain(handler, runnable); m.sendToTarget();
{% endhighlight%}

**Data Message**

如***表4-2***所示，数据集含有多个参数可以传递给消费者线程。

***表4-2*** Message 的参数

|    参数名    |    参数类型    |    含义    |
| -----|:----:| ----:|
|    what     |    int        | 消息的 id，表达消息的意图。 |
|  arg1,arg2  |    int     | 一般处理整型数据时的数据值。在至多传递两个整型数给消费者线程的情况下，这种传递比下面描述的 Bundle 更有效率。 |
|   obj   |   Object   | 任意的对象。如果此对象要传递给另一个进程的线程时，它必须实现 Parcelable 接口。 |
|   data   |  Bundle   | 数据值的容器 |
|   replyTo  |  Messenger  | 在其他进程中的 Handler 的引用，使进程间通信成为可能。 |
|   callback  |  Runnable  |   线程要执行的任务。这是从 Handler.post 方法传递过来的一个内部实例域。 |

**Task Message**

Task Message 通过 **java.lang.Runnable** 对象来表示一个在消费者线程上执行的任务。Task Message 除了它自身不能包括任何数据。



