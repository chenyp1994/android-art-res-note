**Activity的生命周期和启动模式**

[TOC]

# Activity的生命周期全面分析

### 1、典型情况下的生命周期分析

**正常情况下，生命周期分析：**

1. onCreate：进行一些初始化操作
2. onRestart: 在Activity从`不可见`重新变为`可见`状态下，该方法执行。
3. onStart: Activity可见，但不能与用户交互，`出现于后台`。
4. onResume：Activity可见，`出现于前台`并开始活动。
5. onPause：正常情况下：onStop会紧接调用。特殊情况下。
6. onStop：Activity即将停止，可以做一`稍微重量级的回收工作`，同样`不能太耗时`。
7. onDestroy：Activity即将被销毁，可以做一些`回收的工作`和`最终资源的销毁`。

**注意：**
1. 新Activity是透明主题时，旧Activity不会走onStop；
2. Activity切换时，旧Activity的onPause会先执行，然后才会启动新的Activity；
3. Activity在异常情况下被回收时，onSaveInstanceState方法会被回调，回调时机是在onStop之前，当Activity被重新创建的时 候，onRestoreInstanceState方法会被回调，时序在onStart之后；
4. 标识Activity任务栈名称的属性：TaskAffinity，默认为应用包名。

### 2、异常情况下的生命周期分析

1. 情况1：资源相关的系统配置发生改变导致Activity被杀死并重新创建，Activity会被销毁（横屏转竖屏）

>关于保存和恢复View层次结构，系统的工作流程是这样的：首先Activity被意外终止，Activity会调用onSaveInstanceState去保存数据，然后Activity 会委托Window去保存数据，接着Window再委托它上面的顶级容器去保存数据。顶层容器是一个VIewGroup，一般来说它很可能是DecorView。最后顶层容器再一一通知它的子元素来保存数据，这个真个数据保存过程就完成了。
（这是一种典型的委托思想，上层委托下层、父容器委托子元素去处理一件事。例如：View的绘制过程、事件分发等。）

2. 情况2：资源内存不足导致低优先级的Activity被杀死

>Activity按照优先级从高到低，分为以下三种：
1. 前台Activity—正在与用户交互的Activity，优先级最高
2. 可见但非前台Activity—比如Activity弹出了一个对话框，导致Activity的可见但是位于后台无法与用户直接交互。
3. 后台Activity—已经被暂停的Activity，比如执行了onStop，优先级最低。因此一些后台工作放在Service中从而保证进程有一定的优先级，这样就不会轻易被系统杀死。若当某项内容发生改变后，我们不想系统重新创建Activity，我们可以让Activity指定configChanges属性。（例：android：configChanges=“orientation|keyboardHidden”）

**注意：**
1. 系统只有在Activity异常终止的时候才会调用onSaveInstanceState和onRestoreInstanceState来存储和回复数据。
2. Activity正常的销毁的时候，系统不会调用onSaveInstanceState，因为被销毁的Activity不可能再次被显示。


# Activity的启动模式

### 1、Activity的LaunchMode(重要)

**目前有四种启动模式：standard、singleTop、singleTask、singleInstance。**

1. standard：标准模式。每次启动一个Activity都会创建一个新的实例，不管是否存在该实例。（若使用ApplicationContext去启动standard模式的Activity会报错，由于非Activity类型的Context并没有所谓的任务栈）解决方案制定`FLAG_ACTIVITY_NEW_TASK`标记位，这样每次启动都会为它`创建一个新的任务栈`，实际上是以SingleTask模式启动的。

2. singleTop：栈顶复用模式。若新的Activity已存在任务栈的栈顶，那么此Activity不会被重新创建，同时它的onNewIntent方法会被回调，通过此方法的参数我们可以去除当前请求的信息。（注意：onCreate、onStart不会被系统调用。）

3. singleTask：栈内复用模式。单实例模式。只要Activity在一个栈中存在，那么多次启动此Activity都不会重新创建实例，会回调onNewIntent。

4. singleInstance：单实例模式。`singleTask加强版`，除singleTask所有特性，`具有此模式的Activity只能单独地位于一个任务栈中`。简单来说，具有此模式的Activity独立于一个新的任务栈，具有栈内复用的特性，后续的请求不会创建新的Activity，除非该栈被系统销毁。

**注意：**

1. 标识Activity任务栈名称的属性：TaskAffinity，默认为应用包名。
2. 当TaskActivity和SingleTask启动模式配对使用的时候，它具有该模式的Activity的目前任务栈的名字，待启动的Activity会运行在名字和TaskActivity相同的任务栈中。
3. 给Activity指定启动模式，第一种是通过AndroidMenifest为Activity指定启动模式；第二种是通过Intent中设置标记为来为Activity指定启动模式。第二种优先级高于第一种。


### 2、Activity的Flags

1. FLAG_ACTIVITY_NEW_TASK（指定“SingleTask”启动模式）
2. FLAG_ACTIVITY_SINGLE_TOP（指定“SingleTop”启动模式）
3. FLAG_ACTIVITY_CLEAR_TOP（在同一个任务栈内所有位于它上面的Activity都要出栈），一般与SingleTask启动模式一起出现，在这启动Activity的实例如果已经存在，那么系统就会调用它的onNewIntent方法。
4. FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS，具有这个标记的Activity不会出现在历史Activity的列表中，当某些情况下我们不希望用户通过历史列表回到我们的Activity的时候这个标记比较有用，等同于XML中指定Activity的属性android：excludeFromRecents=“true”。


# IntentFilter的匹配规则

### 1. Action的匹配规则
>Intent中的action必须能够和过滤规则中的action匹配，这里指的是action的字符串值完全一样。

**注意:**

1. action的匹配要求Intent中的action存在且必须和过滤规则中的其中一个action相同，这里需要注意它和category规则不同。另外，action区分大小写，不相同的话则匹配失败。


### 2. Category的匹配规则（与Action的匹配规则不同）
>要求Intent中如果含有category，那么所有的category都必须和过滤规则中的其中一个category相同。

**注意:**

1. 为了我们的Activity能够接收隐式调用，就必须再intent-filter中指定“android.intent.category.DEFAULT”这个category。


### 3. Data的匹配规则（与Action的匹配规则类似）

如果过滤规则定义了data，那么Intent中必须也要定义可匹配的data。
data由mineType和URI这两部分组成。

`（URI结构：<scheme>://<host>/[<path>|<pathPrefix>|<pathPattern>]）`

**每个数据的含义：**
1. Scheme：URI的模式，譬如http,file,content等，如果URI中没有指定scheme，那么整个URI的其他参数无效，即URI是无效的。
2. Host：URI的主机名，譬如www.baidu.com，如果host未指定，那么整个URI的其他参数无效，即URI是无效的。
3. Port：URI的端口号，譬如80，仅当URI中scheme和host参数的时候port参数才有意义的。
4. 1)	Path、pathPattern和Prefix：表示路径信息，path表示完成路径信息；pathPattern也表示完整的路径信息，但是它里面可以用通配符，但是注意`“*”要写成“\\*”,“\”要写成“\\\\”；`pathPrefix表示路径的前缀信息。

重点记忆：

`<action android:name=“android.intent.cation.MAIN”/>`

`<category android:name=“android.intent.category.LAUNCHER”/>`


