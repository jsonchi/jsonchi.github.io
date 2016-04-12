---
layout: post
date: 2016-04-09 22:46:10 +0800
title:  "Android Applicaiton Excution"

---

Android 是一个能够同时运行多个应用并且能够使用户在不同应用间无缝切换的多用户多任务的操作系统。其中，Linux 内核负责处理多任务，应用的执行则基于 Linux 进程。

## Linux 进程

Linux 为每一个用户分配一个独一无二的用户 ID，该 ID 根本上来说是操作系统得以将各个用户分别开来的一个数字。每个用户都可以访问被某些特定权限保护下的私有资源，但不能访问其他用户的私有资源（root 用户除外，这儿我们不考虑它）。因此，沙箱用来隔离每个用户。在 Android 系统中，每个应用包都有一个独一无二的用户 ID；在 Android 系统中的一个应用对应于 Linux 下一个独一无二的用户。

Android 系统为每一个进程添加了一个运行时环境，比如，用于每个应用实例的 Dalvik 虚拟机。***图1-3*** 展示了 Linux 进程、虚拟机和应用三者之间的关系。

![图1-3](/resources/images/figure-1-3.png)

***图1-3 应用在不同的进程和虚拟机内运行***

默认情况下，应用和进程是一对一的关系，但如果需要，一个应用在不同的进程中运行也是可能的，而且，不同的应用也可以运行在同一个进程。

## 应用的生命周期

Android 应用的生命周期被封装在 Linux 进程中，这在 Java 中被映射到 **android.app.Application** 类。每个应用在运行时调用 onCreate() 方法时，Application 对象启动。理想状况下，当运行时调用 onTerminate() 时，应用终结，但一个应用不能依赖于此；因为，位于应用之下的 Linux 进程有可能在运行时调用 onTerminate() 方法之前就被杀死了。Application 对象是进程中第一个初始化也是最后一个被销毁的组件。

### 应用的启动

当应用的一个组件初始化执行时，一个应用就启动了。任何组件都可以成为应用的入口；一旦第一个组件被触发执行，一个 Linux 进程就会启动（除非它已经在运行），启动顺序如下：

1. 启动 Linux 进程
2. 创建运行时
3. 创建 Application 实例
4. 创建应用的入口组件

创建一个新的 Linux 进程和运行时并不是一个瞬时的操作，所以，它会降低性能并影响用户体验。因此，系统为了减少 Android 应用的启动时间，会在系统启动时启动一个叫 Zygote 的特殊进程。Zygote 进程预加载了所有的核心库。新的应用进程是从 Zygote 进程 fork 出来的，而不需要拷贝核心库，因为这些核心库在所有应用中共享。

### 应用的终结

一个进程，在应用启动时创建，在系统想要释放资源时终结。因为一个用户可能在稍后的任何时间启动一个应用，所以运行时会避免销毁这个应用的所有资源，除非运行中的应用数量确实导致了系统确实的资源短缺。所以，即使一个应用所有的组件被销毁，它也不会自动终结。

当系统资源短缺时，由运行时决定哪个进程被杀死。这个决定做出的依据基于系统会根据应用的可见性及其当前运行的组件为每一个进程设置一个排名这一事实。在以下的排名中，排名靠后的进程会在排名靠前的进程之前被强制退出。排名从前往后依次是：

* Foreground

&emsp;&emsp;应用当前有一个可见的组件，在一个远程进程中的 Service 绑定了当前的一个 Activity，或者 BroadcastReceiver 正在运行。

* Visible 

&emsp;&emsp;应用有一个部分可见的组件。

* Service

&emsp;&emsp;Service 在后台执行，且没有与可见的组件绑定。

* Background

&emsp;&emsp;一个不可见的 Activity。这一进程排名包含了大多数的应用。

* Empty

&emsp;&emsp;一个没有活跃组件的进程。空进程用于加快启动时间，但是会在系统回收资源时第一个被终结。

实际上，这种排名机制保证了可见的应用在系统耗尽资源时不会被终结。

### 示例：两个交互的应用的生命周期

此例展示了 P1 和 P2 两个进程以一种典型的方式交互（***图1-4***）时的生命周期。客户应用 P1 调起服务应用 P2 的一个 Service。客户应用 P1 由一个广播的 Intent 触发启动；启动时，进程启动了一个 BroadcastReceiver 和 Application 实例。然后一个 Activity 启动，在这整个时间段内，P1 一直拥有最高的进程排名：Forground。

![图1-4](/resources/images/figure-1-4.png)

***图1-4 客户应用启动另一个进程中的 Service***

该 Activity 将工作移交给运行在另一个进程即P2的 Service。因此，应用将工作分配给了两个不同的进程。当 P2 的 Service 保持运行时，P1 的 Activity 可被终结。

一旦所有的组件终结，这两个进程的排名都会变为 Empty，从而使得当系统回收资源时这俩进程可以被终结。

执行过程中进程排名的变化如***表1-1***所示：

| 应用的状态 | P1 的进程排名 | P2 的进程排名 |
| -----|:----:| ----:|
| P1 由 BroadcastReceiver 启动 | Foreground | N/A |
| P1 启动 Activity | Foreground | N/A |
| P1 启动 P2 的 Service | Foreground | Foreground |
| P1 的 Activity 销毁 | Empty | Service |
| P2 的 Service 停止 | Empty | Empty |

需要注意的一点是：由 Linux 进程定义的真正的应用生命周期和我们感知到的生命周期是有区别的。当我们觉得应用被终结时，系统仍然能维持它们进程的运行。如果系统资源允许，为了缩短应用启动的时间，空进程会一直处于逗留状态而不是被终结。


