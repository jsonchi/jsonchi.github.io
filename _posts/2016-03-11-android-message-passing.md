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

#### 数据消息

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

#### 任务消息

任务消息通过 **java.lang.Runnable** 对象来表示一个在消费者线程上执行的任务。任务消息除了它自身不能包括任何数据。

一个消息队列可以包含任意数据消息和任务消息的组合。消费者线程以一种线性的顺序处理它们，而与消息的类型无关。如果一个消息是数据消息，那消费者线程处理相应的数据。任务消息通过让 Runnable 在消费者线程执行来处理，但是消费者线程不会像处理数据消息那样在 Handler.handleMessage(Message) 方法内接受要处理的任务消息。

消息的生命周期非常简单：生产者线程创建消息并最终交由消费者线程处理。这种描述足以包括大多数的情况，但是当有问题发生时，对消息处理的深入理解就是值得的了。现在让我们来看看在一个消息的生命周期内究竟发生了什么。如***图 4-8***所示，这个生命周期可以分为四个重要的状态。为了实现消息的重复利用，运行时在一个应用域内的池中存储这些消息对象；这样就避免了在每次传递消息时创建新的实例。消息对象的处理时间一般很短，很多消息能够在一个时间单元内得到处理。

![图4-8](/resources/images/figure-4-8.png)

***图 4-8 消息的生命周期***

消息这种状态的转换部分由应用控制，部分由平台控制。注意这些状态不是 observable 的，应用也不能观察到从一个状态到另一个状态的转换（尽管有方法可以观察到消息的动作）。因此，当处理消息时，应用不应该对消息的当前状态做任何假设。

##### 初始化

在初始化状态下，一个具有可变状态的消息对象被创建，并且，如果它是一个数据消息的话，用相应的数据填充。应用负责通过以下的调用创建消息对象。他们从对象池中取出对象：

- 显式的对象构造：

{% highlight java%}
Message m = new Message();
{% endhighlight%}

- 工厂方法：
 + 空消息：
  
    {% highlight java%}
Message m = Message.obtain();
	{% endhighlight%}
	
 + 数据消息：
  
    {% highlight java%}Message m = Message.obtain(Handler h);Message m = Message.obtain(Handler h, int what);Message m = Message.obtain(Handler h, int what, Object o);Message m = Message.obtain(Handler h, int what, int arg1, int arg2); Message m = Message.obtain(Handler h, int what, int arg1, int arg2,Object o);
    {% endhighlight%}
	
 + 任务消息：
  	
    {% highlight java%}
Message m = Message.obtain(Handler h, Runnable task);
    {% endhighlight%}
    
 + 复制构造函数：
  
    {% highlight java%}
Message m = Message.obtain(Message originalMsg);
    {% endhighlight%}
	
##### 排队

生产者线程将消息插入队列，并等待被分发给消费者线程。

##### 已分发

在这个状态下，Looper 已经从消息队列中取回消息并且将其从队列中移除了。该消息已被分发给消费者线程，并且消费者线程当前正在处理它。对于此操作并没有相关的 API，因为分发是受 Looper 控制的，不受应用影响。当 Looper 分发消息时，它检查消息的发送信息并将其分发给正确的接受者。一旦消息分发完成，交由消费者线程处理此消息。

##### 已回收

在生命周期的这个点，消息的状态被清除而且消息实例回到消息池中。当在消费者线程上的操作完成时，Looper 负责处理消息的回收。消息的回收由运行时负责，不应该由应用去显式的操作。

***注意*** 一旦消息被插入到消息队列中，消息的内容就不应该被改变。理论上，在消息被分发之前，改变消息的内容是有效的。然而，因为消息的状态不是 observable 的，所以，当生产者线程尝试改变消息的内容时，消费者线程可能正在处理此消息，从而导致线程安全问题。更坏的情况是此消息已经被回收，因为它会返回消息池中，很有可能被另一个生产者将其插入另一个队列。

### Looper

**android.os.Looper** 将消息队列中的消息分发给相关联的 handler。所有满足分发资格的消息，如***图4-6***所示，都会被 Looper 分发。只要消息队列中有满足分发资格的消息，Looper 就会保证消费者线程收到这些消息。当没有消息满足分发资格时，消费者线程将会阻塞，直到有消息满足分发资格。

消费者线程不会直接与消息队列交互。取而代之的是，当 Looper 与线程关联（attach）时，消息队列被添加到该线程中。Looper 负责管理消息队列并将消息分发给消费者线程。


默认情况下，只有 UI 主线程拥有 Looper，应用中创建的线程需要显式的获得一个 Looper。为一个线程创建 Looper 时，该 Looper 被关联到一个消息队列。Looper 扮演了消息队列和线程之间的中间人角色。创建操作在线程的 run 方法中完成：

{% highlight java%}
class ConsumerThread extends Thread {
    @Override
    public void run() {
        Looper.prepare();//1.
        // Handler creation omitted.
        Looper.loop();//2.
    }
}
{% endhighlight%}

1. 第一步通过静态方法 prepare() 创建 Looper，它将创建一个消息队列并跟当前的线程建立关联。此时，消息队列已经准备好插入消息，但是它们还没有分发给消费者线程。
2. 开始处理消息队列中的消息。这是一个阻塞方法，从而保证 run() 方法不会执行完毕。当 run() 方法阻塞时，Looper 将消息分发给消费者线程处理。


一个线程只能与一个 Looper 建立关联，当为线程创建第二个 Looper 时会发生运行时错误。因此，一个线程只能有一个消息队列，这就意味着由多个生产者线程发送的消息在消费者线程上会顺序地执行。所以，当前正在执行的消息会推迟接下来的消息的执行，直到它被执行完。不应该执行一个耗时的消息，如果它会推迟队列中重要任务的执行的话。

#### Looper 的终结

Looper 可以通过 quit 或者 quitSafely 方法请求停止对消息的处理：quit() 停止此 Looper 对队列中任何消息（所有排队的消息，包括满足分发资格的消息）的分发。而 quitSafely 方法只会丢弃那些没有满足分发资格的消息，那些满足分发资格的消息会在 Looper 终结前得到处理。

***注意*** quitSafely 在 API 18 (Jelly Bean 4.3)时才加入。在这之前的 API 只支持 quit。

Looper 的终结并不会终结线程。它只是退出 Looper.loop() 方法，使得线程恢复到调用 loop 方法的那个方法内运行。但是你不能启动一个老的 Looper 或者一个新的 Looper，因此线程不会再处理消息。当你调用 Looper.prepare() 时，会抛出 RuntimeException 异常，因为该线程已经有一个关联的 Looper。如果你调用 Looper.loop() 方法，它将会阻塞，而且也不会有消息从队列中被分发。

#### UI 主线程的 Looper

UI 主线程是唯一一个在默认情况下与一个 Looper 关联的线程。它和应用内创建的一般线程无异，但是 Looper 在应用的组件初始化之前就会与此线程建立关联。

UI 主线程和一般的线程在实际应用中有以下区别:

* 它可以通过 Looper.getMainLooper() 方法在任何地方获得。
* 它不会被会总结。Looper.quit() 会抛出 RuntimeException。
* 运行时通过 Looper.prepareMainLooper() 将一个 Looper 与 UI 主线程建立关联。因此，尝试将 main looper 关联到其他线程会抛出异常。

### Handler

到目前为止，我们一直关注的是 Android 线程间通信的内部情况，但是一个应用大多数情况下是与 **android.os.Handler** 交互的。它是一个双面的 API，既负责信息的插入又负责信息的处理。就像***图 4-5***所示的那样，它在生产者和消费者线程都会被调用用来：

* 创建消息
* 向队列中插入消息
* 在消费者线程处理消息
* 管理队列中的消息

#### 创建 Handler

Handler 通过和 Looper，消息队列以及消息交互发挥它的职责。如***图 4-4***所示，和它唯一联系的是 Looper，Looper 又与消息队列联系。如果没有 Looper，Handler 不会发生作用；他们不会向消息队列中插入消息，因此也不会收到任何要处理的消息。因此，Handler 在构造时就与一个 Looper 绑定在一起：

* 没有显式 Looper 的构造函数会绑定到当前线程的 Looper：

{% highlight java%}
new Handler();new Handler(Handler.Callback)
{% endhighlight%}

* 有显式 Looper 的构造函数的 Handler 绑定到那个 Looper：

{% highlight java%}
new Handler(Looper);new Handler(Looper, Handler.Callback);
{% endhighlight%}

如果没有显式 Looper 参数的构造函数在一个没有 Looper 的线程内调用，Handler 不会绑定任何东西，并且会导致 RuntimeException。Handler 和 Looper 一旦绑定就是 final 的。

一个线程可以有多个 Handler；它们插入的消息在一个消息队列中共存，但是会被分发给正确的 Handler 实例，如图***图 4-9***所示：

![图4-9](/resources/images/figure-4-9.png)

***图 4-9***多个 Handler 共用一个 Looper，插入消息的 Handler 和 处理消息的 Handler 是同一个。

***注意***多个 Handler 并不会并行执行，插入的消息仍在同一个队列并且会被顺序的执行。

#### Message 的创建

为简单起见，Handler 为在消息的**初始化**那一节的工厂方法提供了包装函数来创建消息对象：

{% highlight java%}
    Message obtainMessage(int what, int arg1, int arg2)

    Message obtainMessage()

    Message obtainMessage(int what, int arg1, int arg2, Object obj)

    Message obtainMessage(int what)

    Message obtainMessage(int what, Object obj)
{% endhighlight%}

从 Handle 获得的消息来自消息池，并且隐式的与请求它的 Handler 实例关联起来。这种关联使得 Looper 能够将每条消息分发给正确的 Handler。

#### Message 的插入

因消息的类型不同，Handler 可以有多种方式将消息插入消息队列。任务消息通过前缀为 ***post*** 的方法插入，而数据消息通过前缀为 ***send*** 的方法插入：

* 添加一个任务消息到队列：

{% highlight java%}
    boolean post(Runnable r)
    boolean postAtFrontOfQueue(Runnable r)

    boolean postAtTime(Runnable r, Object token, long uptimeMillis)

    boolean postAtTime(Runnable r, long uptimeMillis)

    boolean postDelayed(Runnable r, long delayMillis)
{% endhighlight%}

* 添加一个数据对象到队列：

{% highlight java%}
    boolean sendMessage(Message msg)

    boolean sendMessageAtFrontOfQueue(Message msg)

    boolean sendMessageAtTime(Message msg, long uptimeMillis)

    boolean sendMessageDelayed(Message msg, long delayMillis)
{% endhighlight%}

* 添加一个简单的数据对象到队列：

{% highlight java%}
    boolean sendEmptyMessage(int what)

    boolean sendEmptyMessageAtTime(int what, long uptimeMillis)

    boolean sendEmptyMessageDelayed(int what, long delayMillis)
{% endhighlight%}

所有的插入方法都会在队列中放入一个新的消息对象，即使应用并没有显式的创建那个消息对象。任务消息 ***post*** 的 Runnable 和数据消息 ***send*** 的 what 会被包装成消息对象，因为那是消息队列只允许的数据类型。

为了指明消息何时满足被分发的资格，插入的消息都会带有一个时间参数。消息在消息队列中的顺序基于该时间参数，这是应用能够影响分发顺序的唯一的方法：

**default**

&emsp;&emsp;马上获得被分发的资格。

**at_front**

&emsp;&emsp;这条消息在时间为 0 时获得被分发的资格。因此它将是下一条被分发的消息，除非在它被处理之前又一条消息插在了前面。

**delay**

&emsp;&emsp;此时间长度过后，消息获得被分发的资格。

**uptime**

&emsp;&emsp;消息获得被分发资格的绝对时间。

即使可以显式的设定延迟和绝对时间，处理消息所需的时间依然是不确定的。这取决于已经存在的需要先处理的消息和操作系统的调度。

插入一个消息到队列并不是万无一失的。***表 4-3***列出了一些一般的错误。

***表 4-3***

|    失败    |    导致的错误    |  典型的应用问题  |
| -----|:----:| ----:|
| 消息没有 Handler | RuntimeException | 消息通过 Message.obtain() 创建并没有指定 Handler |
| 消息已经被分发且处理 | RuntimeException | 同一条消息被插入两次 |
| Looper 已退出 | 返回 false | 消息在 Looper.quit() 调用之后插入 |

***注意*** Handler 的 dispatchMessage 方法是 Looper 用来将消息分发给消费者线程的。如果应用直接调用该方法，消息将会在当前调用此方法的线程处理而不是消费者线程。

#### 示例：双向消息传递

HandlerExampleActivity 模拟了当用户点击一个按钮触发一个耗时操作的场景。耗时操作在后台线程执行，同时 UI 显示一个进度条；当后台线程将返回的结果发给 UI 主线程时，移除进度条。

Activity 的创建如下：

{% highlight java%}
public class HandlerExampleActivity extends Activity {
    private final static int SHOW_PROGRESS_BAR = 1;
    private final static int HIDE_PROGRESS_BAR = 0;
    private BackgroundThread mBackgroundThread;
    private TextView mText;
    private Button mButton;
    private ProgressBar mProgressBar;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_handler_example);
        mBackgroundThread = new BackgroundThread();
        mBackgroundThread.start();//1.
        mText = (TextView) findViewById(R.id.text);
        mProgressBar = (ProgressBar) findViewById(R.id.progress);
        mButton = (Button) findViewById(R.id.button);
        mButton.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick(View v) {
                mBackgroundThread.doWork();//2.
            }
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        mBackgroundThread.exit();//3.
    }
    // ... The rest of the Activity is defined further down
}
{% endhighlight%}

1. 当 HandlerExampleActivity 创建时，启动一个含有消息队列的线程，用来处理来自 UI 主线程的任务。
2. 当用户点击按钮，一个新任务发送给后台线程。由于任务会在后台线程顺序地执行，所以多次点击按钮会导致任务在得到处理之前排队。
3. 当 HandlerExampleActivity 销毁时，停止后台线程。

BackgroundThread 在 HandlerExampleActivity 的生命周期内运行，接收来自 UI 主线程的消息。它没有暴露内部的 Handler，取而代之的是在公共方法 doWork 和 exit 中包装了对 Handler 的访问：

{% highlight java%}
private class BackgroundThread extends Thread {
    private Handler mBackgroundHandler;

    public void run() {//1.
        Looper.prepare();
        mBackgroundHandler = new Handler();//2.
        Looper.loop();
    }

    public void doWork() {
        mBackgroundHandler.post(new Runnable() {//3.
            @Override
            public void run() {
                Message uiMsg = mUiHandler.obtainMessage(
                        SHOW_PROGRESS_BAR, 0, 0, null);//4.
                mUiHandler.sendMessage(uiMsg);//5.
                Random r = new Random();
                int randomInt = r.nextInt(5000);
                SystemClock.sleep(randomInt);//6.
                uiMsg = mUiHandler.obtainMessage(HIDE_PROGRESS_BAR, randomInt, 0, null);//7.
                mUiHandler.sendMessage(uiMsg);//8.
            }
        });
    }

    public void exit() {//9.
        mBackgroundHandler.getLooper().quit();
    }
}
{% endhighlight%}

1. 将线程与一个 Looper 关联
2. Handler 只处理 Runnable 接口，因此无需实现 Handler.handleMessage
3. 发送一个耗时任务给后台线程
4. 创建一个只有 what 参数的消息对象—— SHOW_PROGRESS_BAR 用来告诉 UI 主线程显示进度条
5. 将开始消息发送给 UI 主线程
6. 模拟一个随机时间长度的耗时任务
7. 创建一个含有结果 randomInt(传递给 arg1 参数) 的消息对象，what 参数用来告诉 UI 主线程移除进度条
8. 通知 UI 主线程任务执行完毕并返回一个结果
9. 退出 Looper 以使线程终结

UI 主线程定义了自己的 Handler 接收后台线程的消息来更新 UI：

{% highlight java%}
    private final Handler mUiHandler = new Handler() {
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case SHOW_PROGRESS_BAR://1.
                    mProgressBar.setVisibility(View.VISIBLE);
                    break;
                case HIDE_PROGRESS_BAR://2.
                    mText.setText(String.valueOf(msg.arg1));
                    mProgressBar.setVisibility(View.INVISIBLE);
                    break;
            }
        }
    };
{% endhighlight%}

1. 显示进度条
2. 隐藏进度条并显示返回结果

#### Message 的处理

Looper 分发的消息交由消费者线程的 Handler 来处理。消息的类型决定了处理的方式：

***任务消息***

&emsp;&emsp;任务消息不含数据，是包含一个 Runnable 接口。因此，处理操作在 Runnable 接口的 run 方法内定义，且无需调用 Handler.handleMessage() 方法就能自动的在消费者线程执行。

***数据消息***

&emsp;&emsp;当消息中含有数据时，Handler 就作为消息的接收者并且负责处理消息。消费者线程通过重写 Handler.handleMessage(Message msg) 方法来处理此数据。有两种方式可以达到此目的，以下将会对此说明。

一种方式是在创建 Handler 时定义 handleMessage 方法。一旦消息队列准备好，就应该定义此方法（Looper.prepare() 之后，Looper.loop() 之前）

处理数据消息的模板代码如下：

{% highlight java%}
class ConsumerThread extends Thread {
    Handler mHandler;

    @Override
    public void run() {
        Looper.prepare();
        mHandler = new Handler() {
            public void handleMessage(Message msg) { 
                // Process data message here
            }
        };)
        Looper.loop();
    }
}
{% endhighlight%}

在以上的代码中，Handler 是作为匿名内部类定义的，但是用普通类或者内部类亦可。

另一种方便的方式是实现 Handler.Callback 接口：

{% highlight java%}
public interface Callback {
    public boolean handleMessage(Message msg);
}
{% endhighlight%}

有了 Callback 接口，就没有必要继承 Handler 类。取而代之的是，Callback 的接口实现可以传递给 Handler 的构造函数，以此接收分发过来的消息：

{% highlight java%}
public class HandlerCallbackActivity extends Activity implements Handler.Callback {
    Handler mUiHandler;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mUiHandler = new Handler(this);
    }

    @Override
    public boolean handleMessage(Message message) { 
        // Process messages
        return true;
    }
}
{% endhighlight%}

如果消息处理完成，Callback.handleMessage 应该返回 true，以此保证对此消息不会有进一步的处理。如果返回 false，消息将会传递给 Handler.handleMessage 作进一步处理。注意 Callback 并没有重写 Handler.handleMessage，而是添加了一个消息的预处理器，此预处理器会在 Handler 自己的 handleMessage 方法之前调用。在 Handler 接收到消息之前，Callback 的预处理器可以打断和改变消息。以下的代码展示了如何用 Callback 打断消息的传递：

{% highlight java%}
public class HandlerCallbackActivity extends Activity implements Handler.Callback {//1.
    @Override
    public boolean handleMessage(Message msg) {//2.
        switch (msg.what) {
            case 1:
                msg.what = 11;
                return true;
            default:
                msg.what = 22;
                return false;
        }
    }

    // Invoked on button click
    public void onHandlerCallback(View v) {
        Handler handler = new Handler(this) {
            @Override
            public void handleMessage(Message msg) { 
                // Process message //3.
            }
        };
        handler.sendEmptyMessage(1);//4.
        handler.sendEmptyMessage(2);//5.
    }
}
{% endhighlight%}

1. HandlerCallbackActivity 实现 Callback 接口打断消息
2. 如果 msg.what 为1，返回 true（意味着消息得到处理了）；否则，将 msg.what 的值改为 22 并且返回 false（消息未被处理，它会被传递给 Handler 的 handleMessage 的实现）
3. 在第二个 Handler 内处理消息
4. 插入 msg.what == 1 的消息，此消息将会被 Callback 打断，因为返回的是 true
5. 插入 msg.what == 2 的消息，此消息将会在被 Callback 更改后传递给第二个 Handler

#### 移除队列中的消息

当一条消息加入队列后，只要没有被 Looper 从队列中取出，生产者线程就可以通过 Handler 类的一个方法将它从队列中移除。有时候，应用需要清除消息队列中所有的消息，这是可能的；但大多数情况下，需要更精确的操作：只清除队列中的一部分消息。为此，需要区别出正确的消息。因此，如***表 4-4***所示，消息可以通过一些特定的标识符区别开来。

***表 4-4***

|    标识符类型    |    含义    |  适用的消息类型  |
| -----|:----:| ----:|
| Handler | 消息的接收者 | 任务消息和数据消息 |
| Object | 消息的标签 | 任务消息和数据消息 |
| Integer | 消息的 what 参数 | 数据消息 |
| Runnable | 要执行的任务 | 任务消息 |

Handler 标识符对所有的消息是必须的，因为一条消息总是需要知道它将被分发给哪个 Handler。这个条件隐式的限制了每个 Handler 只能移除属于它自己的消息。一个 Handler 不可能移除消息队列中另一个 Handler 插入的消息。

Handler 类中管理消息队列的方法如下：

* 从消息队列中移除一条任务消息：

{% highlight java%}
    removeCallbacks(Runnable r)

    removeCallbacks(Runnable r, Object token)
{% endhighlight%}

* 从消息队列中移除一条数据消息：

{% highlight java%}
    removeMessages(int what)

    removeMessages(int what, Object object)
{% endhighlight%}

* 从消息队列中移除任务消息和数据消息：

{% highlight java%}
    removeCallbacksAndMessages(Object token)
{% endhighlight%}

Object 标识符既可用于任务消息也可用于数据消息。因此，它可以作为一种标签赋给消息，从而允许你稍后移除掉所有用同一个 Object 打标签的消息。

比如，以下代码片段在消息队列中插入了两条消息使得稍后可以基于标签移除掉它们：

{% highlight java%}
Object tag = new Object();//1.

    Handler handler = new Handler() {

        public void handleMessage(Message msg) {
            // Process message
            Log.d("Example", "Processing message");
        }
    };

    Message message = handler.obtainMessage(0, tag);//2.
    handler.sendMessage(message);
    handler.postAtTime(new Runnable(){//3.
        public void run(){
        // Left empty for brevity
    }
    },tag,SystemClock.uptimeMillis());

    handler.removeCallbacksAndMessages(tag);//4.
{% endhighlight%}

1. 消息的标签标识符
2. 在消息实例中，Object 既被作为数据的容器也被隐式的定义为消息的标签
3. 发送一条带有显式消息标签的任务消息
4. 移除掉所有标签为 tag 的消息

如上所述，在你请求移除一条消息之前，你无法知道这条消息是否被分发和处理。一旦消息被分发，将其插入队列的生产者线程就不能停止任务消息的执行或者数据消息的处理。

#### 观察消息队列

观察排队中的消息以及 Looper 将消息分发给相应的 Handler 的过程是可行的。Android 平台提供了两种观察机制。让我们以例子熟悉它们。

第一个例子展示了如何将当前排队中的消息的快照打印到日志中。

##### 获取当前消息队列的快照

此例在 Activity 创建时，创建了一个工作线程。当用户按下按钮，导致 onClick 被调用，使六条消息以不同的方式被添加到队列中。然后我们观察消息队列的状态：

{% highlight java%}
public class MQDebugActivity extends Activity {
    private static final String TAG = "EAT";
    Handler mWorkerHandler;

    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_mqdebug);
        Thread t = new Thread() {
            @Override
            public void run() {
                Looper.prepare();
                mWorkerHandler = new Handler() {
                    @Override
                    public void handleMessage(Message msg) {
                        Log.d(TAG, "handleMessage - what = " + msg.what);
                    }
                };
                Looper.loop();
            }
        };
        t.start();
    }

    // Called on button click, i.e. from the UI thread.
    public void onClick(View v) {
        mWorkerHandler.sendEmptyMessageDelayed(1, 2000);
        mWorkerHandler.sendEmptyMessage(2);
        mWorkerHandler.obtainMessage(3, 0, 0, new Object()).sendToTarget();
        mWorkerHandler.sendEmptyMessageDelayed(4, 300);
        mWorkerHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                Log.d(TAG, "Execute");
            }
        }, 400);
        mWorkerHandler.sendEmptyMessage(5);
        mWorkerHandler.dump(new LogPrinter(Log.DEBUG, TAG), "");
    }
}
{% endhighlight%}

如***图 4-10***所示，六条带有参数的消息被添加到队列中。

![图4-10](/resources/images/figure-4-10.png)

***图 4-10***

消息添加队列之后，快照就被打印到日志中。只有排队中的消息才会被观察到。因此实际被观察的消息的数量依赖于有多少消息已经被分发给 Handler。其中三条消息被立即地插入到队列，使得获取快照时这三条消息获得了被分发的资格。

运行上面代码输出以下的日志：

{% highlight java%}
49.397:handleMessage-what=2
49.397:handleMessage-what=3
49.397:handleMessage-what=5
49.397:Handler(com.eat.MQDebugActivity$1$1){412cb3d8}@5994288
49.407:Looper{412cb070}
49.407:mRun=true
49.407:mThread=Thread[Thread-111,5,main]
49.407:mQueue=android.os.MessageQueue@412cb090
49.407:Message 0:{what=4when=+293ms}
49.407:Message 1:{what=0when=+394ms}
49.407:Message 2:{what=1when=+1s990ms}
49.407:(Total messages:3)
49.707:handleMessage-what=4
49.808:Execute
51.407:handleMessage-what=1
{% endhighlight%}

消息队列的快照告诉我们 what 参数为 0，1和4的消息处于排队状态。它们都是在添加到队列时设置了分发的延迟。这是一个合理的结果，因为 Handler 的处理时间很短暂——只是打印到日志中。

快照也展示了队列中的每条消息还有多久将会满足被分发的资格。比如，下一条要被分发的消息是 what==4 的消息，它将在 293ms 内被分发。那些始终在排队中但是已经达到被分发的资格的消息在日志中的时间为负值。

##### 跟踪消息队列的处理

消息的处理过程也可以打印到日志中。消息队列的打印功能来自 Looper 类。以下的调用可以开启当前队列的打印功能：
 
{% highlight java%}
Looper.myLooper().setMessageLogging(new LogPrinter(Log.DEBUG, TAG));
{% endhighlight%}

现在让我们看一个跟踪一条发送到 UI 主线程的消息的例子：

{% highlight java%}
mHandler.post(new Runnable(){
        @Override
        public void run(){
            Log.d(TAG,"Executing Runnable");
        }
    });
    mHandler.sendEmptyMessage(42);
{% endhighlight%}

此例发送了两条消息到队列中：一条空消息跟着一条任务消息。正如预期的那样，任务消息先被处理，并被第一个打印出来：

{% highlight java%}
 >>>>> Dispatching to Handler (android.os.Handler) {4111ef40}          com.eat.MessageTracingActivity$1@41130820: 0    Executing Runnable <<<<< Finished to Handler (android.os.Handler) {4111ef40}          com.eat.MessageTracingActivity$1@41130820
{% endhighlight%}

消息的开始和结束是通过三个属性区别开的：

* Handler 实例

&emsp;&emsp;android.os.Handler 4111ef40

* Task 实例

&emsp;&emsp;com.eat.MessageTracingActivity$1@41130820

* what 参数

&emsp;&emsp;0（任务消息无 what 参数）

相同地，what 参数为42的消息日志如下：

{% highlight java%}
  >>>>> Dispatching to Handler (android.os.Handler) {4111ef40} null: 42  <<<<< Finished to Handler (android.os.Handler) {4111ef40} null
{% endhighlight%}

结合以上跟踪消息的分发和获取消息队列快照两种方法，应用可以细致地观察消息的传递。














	



