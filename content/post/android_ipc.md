---
title: "Android中的IPC"
date: 2020-01-08T21:34:06+08:00
draft: false
tags: ["Android", "IPC"]
categories: ["工作"]
---

# 什么是IPC

IPC是Interprocess communication的缩写，即进程间通讯。

# Linux现有IPC方式
**管道**：在创建时分配一个page大小的内存，缓存区大小比较有限

**消息队列**：信息复制两次，额外的CPU消耗；不适合频繁或信息量大的通讯

**共享内存**：无需复制，共享缓冲区直接附加到进程虚拟地址空间，速度快，但进程间的同步问题操作系统无法实现，必须各进程利用同步工具解决

**套接字**：作为更通用的接口，传输效率低，主要用于不同机器或跨网络的通讯

**信号量**：常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源，因此，主要作为进程间以及同一进程内不同线程之间的同步手段

**信号**：不适用与信息交换，更适合与进程中断控制，比如非法内存访问、杀死某个进程等

## Linux中传统IPC通讯原理
首先，我们要知道，Linux中进程之间是有隔离的，而且每个进程的进程空间都会分为“用户空间”和“内核空间”，对应着“用户态”和“内核态”，而“系统调用”则是用户空间访问内核空间的唯一方式，系统调用主要通过如下两个函数实现：
- copy_from_user() //将数据从用户空间拷贝到内核空间
- copy_to_user() //将数据从内核空间拷贝到用户空间

接下来，我们就可以研究传统IPC通讯的原理了，如下图所示：
![](https://imgoldjii.oss-cn-beijing.aliyuncs.com/5b391412f26f4a4aa5d68a1db0dd0a04.jpg)

消息方将要发送的数据存放在内存缓存区，通过系统调用进入内核态，然后内核程序在内核空间分配内存，开辟一块内存缓存区，调用copy_from_user()将数据从用户空间的内存缓存区拷贝到内核空间的内核缓存区中。同样的，接收方进程在接受数据时在自己的用户空间开辟一块内存缓存区，然后内核程序调用copy_to_user()将数据从内核缓存区拷贝到接受进程的内存缓存区。这时两个进程间就完成了一次数据传输，我们称完成了一次进程间通讯

# Android中的IPC
通常，一个App只有一个进程，但Android是可以实现多进程的，比如某些通讯App会单独开辟一个常驻后台的进程。发展迅速，这种做法越来越常见

## Android中如何多进程
通常，只有一种方法，即在AndroidMenifest中指定新的`android:process`属性

具体如下：
```xml
<activity android:name=".Activity1">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<activity android:name=".Activity2"
    android:process=":remote"/>
<activity android:name=".Activity3"
    android:process="com.example.myapplication.remote"/>

------------------------------
上述代码创建了三个进程：
1. com.example.myapplication
2. com.example.myapplication.remote
3. com.example.myapplication.remote

后两个进程的区别：
- 用":"创建的是私有进程，其他应用不可以和他在一个进程共存
- 写全包名的是全局进程，其他应用可以通过shareUID的方式共存在同一进程

解释一下什么是UID：
Android系统会给每一个应用分配一个唯一的UID，具有相同UID且签名相同的应用才能共享数据
```

还有一种非常规的方法：通过JNI在native层中去fork新进程

## Android多进程引发的问题
在介绍AndroidIPC之前，先说一下为什么需要这些方式，原有的Android通讯机制或者LinuxIPC满足不了Android多进程吗？

Android系统会给每一个进程分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致了很多问题：
- 静态成员和单例模式失效
- 线程同步机制失效
- SP并发操作导致数据可靠性下降
- Application多次创建（相当于重启了App）

为了解决上述问题，引出了如下AndroidIPC的方式
- 文件共享
- ContentProvider
- Bundle
- Messager
- AIDL
- Binder

除文件共享外，这些IPC的方式，底层其实都是使用的Binder机制

---
## 文件共享
显而易见，就是通过两个进程读/写同一个文件来实现交换数据

**缺点**：并发操作可能会导致文件数据的有效性

**场景**：适用于对数据同步要求不高的进程之间通讯，并且要妥善解决并发读写的问题

---
## ContentProvider
主要是以表格的形式来组织数据，包含多张表，一行对应一条记录、一列对应一条记录中的一个字段，与数据库类似。除去表格形式还支持图片、视频等文件数据

组织封装完数据后会提供统一的存取接口，使得其他进程可以忽略底层数据存储的方式，仅通过统一接口来操作数据

通常与他的辅助工具类一起实现通讯，可以实现进程间通讯或进程内通讯

**原理**：Binder机制，外部调用CURD方法的话是运行在ContentProvider进程的Binder线程池中（不是主线程）

**优点**：对数据进行安全的封装；提供统一的存取数据的接口供其他进程调用（无需考虑底层测数据存储方式是SQLite还是内存存储之类的）

**场景**：一对多的进程间共享数据，比如获取/修改系统亮度

**用法**：[Android校招面试指南-ContentProvider全方位分析](https://lrh1993.gitbooks.io/android_interview_guide/content/android/basis/ContentProvider.html)

---
## Bundle
Bundle中数据以key-value键值对的形式存在，我们通常在Activity、线程之间使用它，其实由于Bundle实现了Parcelable接口，所以也可以在进程之间使用，只需配置一下目标包名信息即可

参考如下：
```java
Bundle bundle = new Bundle();
bundle.puString("test",  "来自A");
Intent intent = new Intent(Intent.ACTION_MAIN);
intent.addCategory(Intent.CATEGORY_LAUNCHER);
ComponentName cn = new ComponentName("com.test", "com.test.MainActivity");
intent.setComponent(cn);
intent.puExtras(bundle);
startActivity(intent);
```

当要使用Bundle传递对象时，必须序列化，即实现Serializable/Parcelable接口

**关于序列化**：[Oldjii的笔记-Android序列化](http://note.youdao.com/noteshare?id=5521000160fbda9125b03909587fb2b6)

**原理**：Bundle只是一个信息的载体，内部维护了一个Map<String, Object>

**场景**：四大组件间的进程间通讯

---
## Messager
Messager可以翻译为“信使”，通过它可以在不同进程间传递Message对象，在Message中放如数据，就可以实现进程间通讯了，这是一种轻量级的IPC方案

**与Handler的关系**：Messager底层使用了AIDL的方式，但和普通的AIDL不同的是，它是利用Handler进行处理的，其实这就是它不支持并发的原因；Handler是线程间通讯的一种机制，其本身是不支持进程间通讯（IPC）的

想了解Handler机制，可以参考：[Oldjii的笔记-Handler机制解析](http://note.youdao.com/noteshare?id=5eb43fdb54d3c64cb933cb5280ba5201)

**原理**：底层实现是AIDL

**优点**：安全，不支持并发（既可以说是优点也可以说是缺点）；封装AIDL，使用简单，不需要AIDL文件；支持实时通讯

**缺点**：串行处理客户端发来的消息，服务端不存在并发情况；数据通过Message传递所以只能传递Bundle支持的数据类型

**场景**：低并发的一对多即使通讯

**使用**：[Messenger轻量级IPC方案](https://www.jianshu.com/p/52ec31fcf0a6)

---
## AIDL
AIDL是Android Interface Description Language的缩写，即Android接口定义语言，用于定义跨进程通讯中双方皆认可的编程接口

**与Messager的关系**：
Messager是基于AIDL封装的，但服务端仅支持串行处理消息，如果有大量的并发请求，那么Messager就不合适了；而且，使用Messager的目标主要是传递消息，但IPC不仅仅如此，还可能会跨进程调用服务端的方法，这种情况Messager就无法满足了，而直接使用AIDL是没有问题的

**场景**：一对多进程间通讯

原理：基于Binder封装

**使用**：通过编写AIDL文件来设计想要暴露的接口，编译后会自动生成响应的java文件，服务器将接口的具体实现写在Stub中，用iBinder对象传递给客户端，客户端bindService时，用asInterface的形式将iBinder还原成接口，再调用其中的方法

详情参考：[Android校招面试指南-Binder机制及AIDL使用](https://lrh1993.gitbooks.io/android_interview_guide/content/android/advance/binder.html)

---
## Binder
Binder机制是Android独有的一种跨进程通讯的方式，是Linux中没有的

Android系统内部本身就是使用Binder来实现IPC，Framework层中XXXManager和XXXManagetService之间等等都是利用的Binder机制，其中XXXManager是Client端、XXXManagetService是Server端，比如ActivityManager、ActivityManagerService、WindowManager、WindowManagerService、PackageManager、PackageManagerService，乃至于Native Framework层的MediaPlay与MediaPlayService也是一样的

详情请参考：[Android手机从开机到APP启动经过的流程](http://jiguankai.cn/2019/09/17/Android手机从开机到APP启动经过的流程/)

在应用层，开发者也可以利用Binder实现IPC，其实Messager、AIDL这些方式底层基于Binder的，Binder也可以直接使用，下面是三者使用的场景
- AIDL：需要不同应用的客户端通过IPC通讯访问你的服务，并且需要支持多线程的情况
- 直接使用Binder：不需要同时对几个应用进行IPC操作的情况
- Messager：需要实现IPC，但不需要处理多线程的情况

**直接使用**：[Android Binder的极简使用](https://www.jianshu.com/p/2d6ddd6a3399)

### Binder底层原理
在看这部分时，建议你先翻到上面去回顾一下“传统IPC的原理”，这样可以更好的理解Binder的通讯原理

Binder与传统IPC不同的地方主要在于使用了“动态内核可加载模块”和“内存映射”

#### 动态内核可加载模块
Linux的动态内核可加载模块（LKM）是一段具有独立功能的程序，它可以被单独编译，但是不能单独运行，该模块在运行时被链接到内核作为内核的一部分运行。这样Android系统就可以通过动态添加一个内核模块运行在内核空间，用户进程之间则通过这个内核模块作为桥梁来实现通讯

这个负责各用户进程Binder通讯的内核模块就是Binder驱动（Binder Dirver）

#### 内存映射
内存映射简单来说就是，将用户空间的一块内存区域映射到内核空间，映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间，反之亦然。这样利用内存映射就可以减少数据拷贝的次数

BinderIPC中的内存映射是通过mmap()来实现的，mmap()是操作系统中一种内存映射的方式，mmap()通常是用在物理介质的文件系统中，但Binder并不存在物理介质，也无法实现利用mmap()在物理介质和用户空间之间建立映射关系。mmap()在Binder的作用是用来在内核空间创建数据接受的缓存空间

#### 完整流程
了解了上面的概念，就可以理解完整BinderIPC的通讯原理了

一次完整的BinderIPC通讯过程：
1. 首先Binder驱动在内核空间创建一个数据接受缓存区
2. 接着在内核空间开辟一块内核缓存区，并建立内核缓存区与数据接收缓存区之间的映射关系，以及内核中的数据接收缓存区与接收进程用户空间的映射关系
3. 发送方通过系统调用copy_from_user()将数据copy到内核中的内核缓存区，由于内核缓存区与接收进程的用户空间存在内存映射，所以也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通讯

如下图所示：
![](https://imgoldjii.oss-cn-beijing.aliyuncs.com/20200108145347.png)


### Binder通讯流程
#### Client/Server/ServiceManager/Binder Driver
Binder 是基于 C/S 架构的。由一系列的组件组成，包括 Client、Server、ServiceManager、Binder Driver，其中 Client、Server、ServiceManager 运行在用户空间，Binder Driver运行在内核空间。其中ServiceManager和Binder Driver由系统提供，而Client、server由应用程序来实现。Client
、Server、ServiceManager均是通过系统调用open()、mmap()、ioctl()来访问设备文件"/dev/binder"，从而实现与Binder Driver的交互来间接的实现跨进程通讯

![](https://imgoldjii.oss-cn-beijing.aliyuncs.com/20200108150804.png)

这里需要你理解一下ServiceManager这个组件

Client、Server、ServiceManager、Binder Driver这几个组件在通信过程中扮演的角色就如同互联网中服务器（Server）、客户端（Client）、DNS域名服务器（ServiceManager）以及路由器（Binder Driver）之前的关系。以下内存摘自《Android Binder 设计与实现》

> ServiceManager 与实名 Binder ServiceManager 和 DNS 类似，作用是将字符形式的 Binder 名字转化成 Client 中对该 Binder 的引用，使得 Client 能够通过 Binder 的名字获得对 Binder 实体的引用。注册了名字的 Binder 叫实名 Binder，就像网站一样除了除了有 IP 地址以外还有自己的网址。Server 创建了 Binder，并为它起一个字符形式，可读易记得名字，将这个 Binder 实体连同名字一起以数据包的形式通过 Binder 驱动发送给 ServiceManager ，通知 ServiceManager 注册一个名为“张三”的 Binder，它位于某个 Server 中。驱动为这个穿越进程边界的 Binder 创建位于内核中的实体节点以及 ServiceManager 对实体的引用，将名字以及新建的引用打包传给 ServiceManager。ServiceManger 收到数据后从中取出名字和引用填入查找表
>

#### binder_xxx()函数介绍
**init()**：创建/dev/binder设备节点 

**open()**：获取Binder Driver的文件描述符

**mmap()**：在内核分配一块内存，用于存放数据

**ioctl()**：将IPC数据作为参数传递给Binder Driver

**ioctl命令**：
```markdown
BINDER_WRITE_READ           -->     收发Binder IPC数据
BINDER_SET_MAX_THREADS      -->     设置Binder线程最大个数
BINDER_SET_CONTEXT_MGR      -->     设置Service Manager节点
BINDER_THREAD_EXIT          -->     释放Binder线程
BINDER_VERSION              -->     获取Binder版本信息
BINDER_SET_IDLE_TIMEOUT     -->     没有使用
BINDER_SET_IDLE_PRIORITY    -->     没有使用
```

更多关于Binder源码的内容请参考：[Binder系列1—Binder Driver初探](http://gityuan.com/2015/11/01/binder-driver/)

#### 完整的Binder通讯过程
一次完整的Binder通讯过程如下：
1. 首先，一个进程使用ioctl()命令（BINDER_SET_CONTEXT_MGR）通过Binder将自己注册成为ServiceManager
2. Server通过Binder Driver向ServiceManager注册Binder（Server中的Binder实体），表明可以对外提供服务，驱动为这个Binder创建位于内核中的实体节点以及ServiceManager对实体的引用，将名字以及新建的引用打包给ServiceManager、ServiceManager将其填入查找表
3. Client通过名字，在Binder Driver的帮助下从ServiceManager中获取到对Binder实体的引用，通过这个引用就能实现和Server的通讯

![](https://imgoldjii.oss-cn-beijing.aliyuncs.com/20200108154444.png)

---
## 为什么Android系统选用Binder机制作为IPC的方式？
可以从性能、安全两个方面来回答这个问题

##### 性能方面
在移动设备上，广泛的使用跨进程通讯对通讯机制的性能有很严格的要求，Binder相对于传统的Socket方式，更加高效。因为Binder数据拷贝只需一次，而管道、消息队列、Socket等需要2此，虽然共享内容方式一次内存拷贝都不要，但是实现方式过于复杂，不适合该场景。

##### 安全方面
传统的进程通讯方式对于通讯双方的身份并没有作出严格的验证，比如Socket通讯的IP地址是客户端手动填入，很容易进行伪造。然而，Binder机制从协议本身就支持对通讯双方做身份校验，从而大大的提高了安全性