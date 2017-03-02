**IPC 机制**

[TOC]


#Android IPC简介
> IPC是Inter-Process Communication的缩写,含义为进程间通信或者跨进程通信,是指两个进程之间进行数据交换的过程。IPC不是Android中所独有的，任何一个操作系统都需要有响应的IPC机制。在Android中最有特色的进程间通讯方式就是Binder了，通过Binder可以轻松地实现进程间通信，当然同一个设备上的两个进程也可以通过Socket通信也是可以。


**多进程的情况有两种：**

1. 一个应用因为某些原因自身需要采用多进程模式来实现。原因：可能是有些模块由于特殊需要运行在单独的进程中，又或者为了加大一个应用可用的内存所以需要通过多进程来获取多分内存空间。
2. 当前应用需要向其他应用获取数据，由于两个应用，所以必须要采用跨进程的方式来获取所需的数据，甚至我们通过系统提供的ContentProvider去查询数据的时候，其实也是一种进程间的通信。


#Android中多进程模式
### 1、开启多进程模式
> 在Android中使用多进程只有一种方式，那就是给四大组件在AndroidMenifest中指定android:process属性，除此没有别的方法，也就是说我们无法给一个线程或者一个实体类指定其运行时的进程。还有一种非常规的夺金成方法，通过JNI在native层去fork一个新的进程，但其属于特殊情况。

**Activity标记进程的两种方式：**

1. android:process=”:remote”
2. android:process=”com.xxx(包名).remote”。

区别是，”:”的含义是指要在当前的进程前面附加上当前报名（简写），进程名以”:”开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中，而进程名不以”:”开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。

**注意：**

1. Android系统会为每一个应用分配一个唯一的UID具有相同UID的应用才能共享数据。
2. 两个应用通过ShareUID跑在同一个进程中是有要求的，需要两个应用有相同的ShareUID并且签名相同才可以。这个情况下，它们可以相互访问对方的私有数据。

### 2、多进程模式的运行机制
> 所有运行在不同进程中的四大组件，只要它们之间需要通过内存来共享数据，都会共享失败，这也是多进程所带来的主要影响。正常情况下，四大组件中间不可能不通过一些中间层来共享数据，那么通过简单地指定进程名来开启多进程都会无法确定运行。

**一般来说，使用多进程会造成如下几方面的问题：**

1. 静态成员和单例模式完全失效
2. 线程同步机制完全失效
3. SharePreference的可靠性下降
4. Application会多次创建

**注意：运行在同一个进程中的组件是属于同一个虚拟机和同一个Application的，同理，运行在不同进程中的组件是属于两个不同的虚拟机和Application的。**


#IPC基础概念介绍
### 1、Serializable接口

Serializable方式实现对象序列化，需要采用ObjectOutputStream(writeObject方法)和ObjectInputStream(readObject)来实现。serialVersionUID是用来辅助序列化和反序列化的过程的，原则上序列化的数据中的serialVersionUID只有和当前类的serialVersionUID相同才能够正常的被反序列化。


**serialVersionUID的详细工作机制：**
序列化的时候系统会把当前类的serialVersionUID写入序列化的文件中（也可能是其他中介），当反序列化的时候系统会去检测文件中的serialVersionUID，看它是否和当前类的serialVersionUID一致，如果一致就说明序列化的类的版本和当前类的版本是相同，这个时候可以成功反序列化，否则就说明当前类和序列化的类相比发生了某些变换。

**注意：**

1. 一般来说，我们应该手动去指定serialVersionUID的值，也可以让工具根据当前类的结构自动去生产它的hash值，这样反序列化和序列化时两者的serialVersionUID是相同的，因此可以正常进行反序列化。
2. 系统的默认序列化过程也是可以改变的，通过实现writeObject（ObjectOutputStream out）和readObject（ObjectInputStream in）即可重写系统默认的序列化和反序列化过程。

### 2、Parcelable接口
一个类的对象实现这个接口就可以实现序列化并可以通过Intent和Binder传递。（Parcel内部包装了可序列化的数据，可以在Binder中自由传输）

表2-1 Parcelable 的方法说明：
![](/media/Chapter_2/Parcelable的方法说明.jpg)

**注意：**

1. Parcelable是Android中的序列化方式，因此更适合用在Android平台上，它的缺点使用起来很麻烦，但是它的效率很高，首选Parcelable。
2. Parcelable主要用在内存序列化上，通过`Parcelable将对象序列化到存储设备中`或者`将对象序列化后通过网络传输`也可以，但是过程稍显复杂，在这两种情况下建议使用Serializable。

### 3、Binder
Binder是Android中的一个类，它实现了IBinder接口。从IPC角度来说，Binder是Android中的一种跨进程通信方式，也可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux没有；从AndroidFrameWork角度来说，Binder是ServiceManager连接各种Manager（ActivityManager、WindowManager等等）和相应的ManagerService的桥梁；从Android应用层来说，Binder是客户端和服务器端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或数据，这里的服务包括普通服务(Massager)和基于AIDL的服务。（主要运用于Service，普通的Service中是不设计进程间通信。）

**Binder工作机制：（图来自网上）**

![](/media/Chapter_2/Binder工作机制.jpg)



#Android中的IPC方式
### 1、使用Bundle（最简单）
一种特殊的使用场景：将原本需要在A进程的计算任务转移到B进程的后台Service中去执行，这样就可以避免了进程间通信的问题，而且只用了很小的代价。

### 2、使用文件共享
两个进程间通过读/写同一个文件来交换数据。（一个进程写另一个进程读），但SharedPreferences是个特例，虽说本质上是文件的一种，但是由于系统对它的读/写有一定的缓存策略，即在内存中会有一份SharedPreferences文件的缓存，因此在多进程模式下，系统对它的读/写就变得不可靠，当面对高并发的读/写访问，SharedPreferences有很多几率会丢失数据，因此，`不建议在进程间通信中使用SharePreferences`。


### 3、使用Massager（为了传递消息，不适合大量并发请求）
Messager（信使），通过它可以在不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递了。

**实现Messager的几个步骤，分为服务端和客户端：**

##### 1. 服务器进程
> 首先，在服务端创建一个Service来处理客户端的连接请求，同时创建一个Handler并通过它来创建一个Messager对象，然后在Service的onBind中返回这个Messager对象底层的Binder。

##### 2. 客户端进程
> 首先绑定服务端的Service，绑定成功后在服务端返回的IBinder对象创建一个Messager，通过这个Messager就可以向服务端发送消息了，发送消息的类型是Message对象。如果需要服务端能够回应客户端，就和服务端一样，我们还需要创建一个Handler并创建一个新的Messager，并把这个Messager对象通过message的replyTo参数传递给服务端，服务端通过这个replyTo参数就可以回应客户端。

**Messager的工作原理：（图来自网上）**

![](/media/Chapter_2/Messager的工作原理.jpg)


### 4、使用AIDL（实现跨进程的方法调用）
AIDL也是Messager的底层实现，Messager（系统封装）本质也是AIDL。

**使用AIDL进行进程间通信的流程，分为服务端和客户端两方面：**

##### 1. 服务端

> 首先要创建一个Service用来监听客户端的连接请求，然后创建一个AIDL文件，将暴露给客户端的接口在这个AIDL文件中声明，最后在Service中实现这个AIDL接口即可。

##### 2. 客户端

>首先需要绑定服务端的Service，绑定成功后，将服务端返回的Binder对象转成AIDL接口所属的类型，然后即可调用AIDL中的方法了。
##### 3. AIDL接口的创建

>创建一个后缀AIDL的文件，在里面声明接口和方法。AIDL文件支持的类型：基本类型（int、long、char、boolean、double等）；String和CharSequence；List：只支持ArrayList，里面每个元素都必须能够被AIDL支持；Map：只支持HashMap，里面的每个元素都必须被AIDL支持，包括key和value；Parcelable对象；AIDL接口本身。

##### 4. 远程服务Service的实现

>实现Service，初始化消息或信息，然后创建一个Binder对象并在onBind中返回它，这个对象继承AIDL对象并实现内部的AIDL方法。**注意：**

1. 对于数据并发，需要使用CopyOnWriteArrayList（支持并发读/写）类似的方法。
2. 服务端可以使用CopyOnWriteArrayList和ConcurrentHashMap来进行自动线程同步，客户端拿到的依然是ArrayList和HashMap；
3. 该Service是运行在独立进程中的，它和客户端的Activity不在同一个进程中。

##### 5. 客户端的实现
首先要绑定远程服务，绑定成功将服务端返回的Binder对象转换成AIDL接口。客户端要注册AIDL接口到远程服务端，同时也要在Activity退出时接触注册，当服务端发现有新的信息，服务端会通过接口的方法回调数据到客户端，但是此方法是在客户端的Binder线程池中执行，为了方便进行UI操作，需要通过Handler切换回主线程去执行操作。

**注意：**

1. 提供一个AIDL接口，每个用户都需要实现这个接口并且向服务端的提醒功能，用户也可以随时取消这种提醒。（订阅—通知）
2. 在多进程的条件下，解除注册接口是不奏效的，因为对象是不能跨进程直接传输的，对象的跨进程传输本质是反序列化的过程，这就是为什么自定义对象都必须要实现Parcelable接口的原因。
3. 实现解除注册功能，需要使用RemoteCallbackList，它是系统专门提供的用于删除跨进程的Listener的接口。RemoteCallbackList是一个泛型，支持管理任意的AIDL接口。工作原理：它的内部有一个Map接口专门用来保存所有AIDL回调，Map的key是IBinder类型，value是Callback类型。Callback中封装了真正的远程Listener。客户端解注册时，我们需要遍历服务器所有的Listener，找出和解注册Listener具有相同Binder对象的服务器Listener并把它删除即可。它还具有一个功能：当客户端进程终止后，它能够自动移除客户端所注册的Listener；其内部实现线程同步的功能，在注册和解注册时，不需要做课外的线程同步工作。（用RemoteCallbackList代替CopyOnWriteArrayList）
4. 客户端的onServiceConnected和onServiceDisconnected方法运行在UI线程中，不可以在它们直接调用服务端的好事方法。
5. 由于服务端的方法本身就运行在服务端的Binder线程池中，（服务端本来就可以执行大量耗时操作），切记不要在服务端方法中开线程去进行异步任务。
6. Binder可能意外死亡，一般都是由于服务端进程意外停止了。此时我们需要重连。
    **重连两个方法：**
    1. 给Binder设置DeathRecipient监听，当binder死亡时，我们会受到binderDied方法的回调，在binderDied方法中我们可以重连远程服务。（不能访问UI）
    2. 在onServiceDisconnected中重连远程服务。

7. 如何在AIDL中使用权限验证功能。（默认：任何人都可以链接）
    **常用两个方法：**
    1. 在onBinder中进行验证，验证不通过就直接返回null。例如：自定义Permission判断。
    2. 在服务端的onTransact方法中进行权限验证，如果验证失败就直接返回false，这样服务端就不会之中执行AIDL中的方法从而达到保护客户端的效果。例如：采用第一种方法，还可以采用Uid和Pid来做验证，通过getCallingUid和getCallingPid可以拿到客户端所属的应用的Uid和Pid，通过这个方法可以做一些验证工作，比如验证包名。


### 5、使用ContentProvider
ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，非常适合进程间通信。其底层实现同样也是Binder。

**注意：**

1. 创建一个自定义ContentProvider需要继承ContentProvider并实现六个抽象方法：onCreate、query、update、insert、delete、getType（用来返回一个Uri请求所对应的MINE类型）；
2. ContentProvider主要以表格的形式来组织数据，并且可以包含多个表，对于每个表格来说，它们都具有行和列的层次性，行往往对应一条记录，二列对应一条记录中的一个字段，这一点类似数据库。
3. ContentProvider权限还可以细分到读权限和写权限，分别对应android:readPermission和android:writePermission属性。
**AndroidManifest的例子：**

```
<provider		Android:name=”.provider.xxx”		Android:authorities=”xx.xx.xx.xx”（Provider的唯一标识）		Android:permission=”xx.xx.xx”（自定义权限）		Android：process=”:provider”（进程）></provider>
```

4. 当通过update、insert、delete引起数据源的改变，ContentProvider需要通过自身的notifyChange方法来通知外接当前的ContentProvider中的数据已经改变。还有，要观察一个ContentProvider中的数据改变情况，可以通过ContentProvider的registerContentObserver方法来注册观察者，通过unregisterContentObserver方法来解除观察者。
5. query、update、insert、delete四大方法是存在多线程并发访问的，因此方法内部需要做好线程同步。

### 6、使用Socket
Socket也称为“套接字”，是网络通信中的概念，它分为流式套接字和用户数据报套接字两种，分别对应网络的传输控制层中的TCP和UDP协议。

**注意：**

1. TCP协议是面向连接的协议，提供稳定的双向通信功能，TCP连接的简历需要通过“三次握手”才能完成，为了提供稳定的数据传输功能，其本身提供了超市重传机制，因此具有很高的稳定性；
2. UDP是无连接的，提供不稳定的单项通信功能，当然UDP也可以实现双向通信的功能。
3. 在性能上，UDP具有更好的效率，其缺点是不保证数据一定能够正确传输，尤其在网络拥塞的情况下。


#Binder连接池
对于将所有的AIDL放在同一个Service中去管理。整个工作机制是这样的：每个业务模块创建自己的AIDL接口并实现此接口，这个时候不同业务模块之间是不能有耦合的，所有实现细节我们要单独开来，然后向服务端提供自己的唯一标识和其对应的Binder对象；对服务端来说，只需要一个Service就可以了，服务端提供一个queryBinder接口，这个接口能够根据业务模块的特征来返回相应的Binder对爱心那个给它们，不同的业务模块拿到所需的Binder对象后就可以进行远程方法调用了。

**注意：**

1. Binder连接池的主要作用就是将每个业务模块的Binder请求统一转发到远程Service中去执行，从而避免了重复创建Service的过程。

**如下图：（图来自网上）**
![](/media/Chapter_2/Binder连接池工作原理.jpg)

2. BinderPool能够极大地提高AIDL的开发效率，并且可以避免大量Service创建，因此，建议在AIDL开发工作中引入BinderPoor机制。


#选用合适的IPC方式

![](/media/Chapter_2/选用合适的IPC方式.jpg)
