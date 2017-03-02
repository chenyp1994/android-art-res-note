**View的事件体系**

[TOC]


# View基础知识

### 1、View的基础知识

###### 1. 什么是View

>View是一种界面层的控件的一种抽象，它代表一个控件。除了View，还有ViewGroup。
在Android设计中，ViewGroup也集成View，View本身就可以是单个控件也可以是多个控件组成的一组控件，通过这种关系就形成了View的树的结构，和Web的DOM树概念是相似的。

###### 2. View的位置参数

>View的位置主要由它的四个顶点来决定，分别对应于View的四个属性：top(左上纵坐标)、left(左上角横坐标)、right(右下角横坐标)、bottom(右下角纵坐标)。注意：坐标都是相对于父容器来说。

**View的位置参数：**

![View的位置参数](/media/Chapter_3/View的位置参数.jpg)

###### 3. MotionEvent和TouchSlop

1.MotionEvent手指触摸屏幕后所产生的系列事件。

    **典型的事件类型有如下几种：**

      1. ACTION_DOWN----手指刚触摸屏幕；
      2. ACTION_MOVE----手指在屏幕上移动的瞬间；
      3. ACTION_UP-------手指从屏幕上松开的一瞬间。

    **正常情况下，一次手指触摸屏幕的行为会触发一系列的点击事件，考虑如下几种情况：**

      1. 点击屏幕后离开松开，事件序列为DOWN->UP；
      2. 点击屏幕滑动一会再松开，事件序列为DOWN->MOVE->...->MOVE->UP。
`以上的时间序列，都可以用MotionEvent来获取到发生事件的xy坐标。它提供了两组方法，getX/getY（相对于当前View的左上角）和getRawX/getRawY（相对于屏幕左上角）。`


2.TouchSlop是系统所能识别出的被认为是滑动的最小距离，即是当手指在屏幕上滑动时，如果两次滑动之间的距离小于这个常量（根据设备的不同，值可能不同），那么系统就不认为你是在进行滑动操作。可以通过ViewConfiguration.get(getContext()).getScaledTouchSlop()来获取最小滑动距离。（在源码framework/base/core/res/res/values/config.xml的“config_viewConfigurationTouchSlop”的值）。

###### 4. VelocityTracker、GestureDetector和Scroller

**1.VelocityTracker(速度追踪)**

速度追踪，用于追踪手指在滑动过程中的速度，包括水平和垂直方向的速度。

```
使用流程：
1. 在View的onTouchEvent方法中追踪当前点击事件的速度：
VelocityTracker velocityTracker = VelocityTracker.obtain();
velocityTracker.addMovement();
2. 当想知道当前滑动速度时：
注意：（1：获取速度之前必须先计算速度，即必须执行第一个方法，2：速度时指在一段时间内手指滑动的像素数。）
1：velocityTracker.computeCurrentVelocity(1000)；（时间间隔，单位为毫秒）
int xVelocity = (int) velocityTracker.getXVelocity();
int yVelocity = (int) velocityTracker.getYVelocity();
3. 在不需要使用的时候，需要调用clear方法来重置并回收。
velocityTracker.clear();
velocityTracker.recyler();
```


**2.GestureDetector(手势检测器)**

手势监测，用于辅助监测用户的单击、滑动、长按、双击等行为。

```
使用流程：
1)	需要创建一个GestureDetector对象并实现OnGestureListener接口，根据需要还可以实现OnDoubleTapListener从而能够监听双击行为：
GestureDetector mGestureDetector = new GestureDetector(this);
//解决长按屏幕后无法拖动的现象
mGestureDetector.setIsLongpressEnable(false);
2)	接着目标View的onTouchEvent方法，在待监听View的onTouchEvent方法中添加如下实现：
boolean consume = mGestureDector.onTouchEvent(event);
return consume;
3)	除此之外还可以选择实现OnGestureListener和OnDoubleTapListener中的方法。（常用标记为红色）
```
**OnGestureListener和OnDoubleTapListener中的方法：**
![OnGestureListener和OnDoubleTapListener中的方法](/media/Chapter_3/OnGestureListener和OnDoubleTapListener中的方法.jpg)

`说明：实际开发中，可以不适用GestureDetector，可以直接在自己的View的onTouchEvent方法中实现需要的监听。建议，如果只是监听滑动相关，建议自己在onTouchEvent中实现，如果要监听双击行为的话，那么就使用GestureDetector。`

**3.Scroller(弹性滑动对象)**

弹性滑动对象，用于实现View的弹性滑动。
>当使用View的scrollTo/scrollBy方法来进行滑动时，其过程是瞬间完成，这个没有过度效果的滑动，用户体验差。此时就可以用Scroll来实现有过度效果的滑动，注意过程不是瞬间完成，而是需要一定的时间间隔内完成。

**使用过程:**

1. Scroll本身无法让View弹性滑动，需要和View的ComputeScroll方法配合使用才能完成。

在滑动方法内，mScroll.startScroll(scroll, 0, delta(滑动距离), 0, 1000(时间间隔));和invalidate()(重绘)

```
public void computeScroll(){
	if(mScroller.computeScrollOffset()){
		scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
		postInvalidate();
  }
}
```

### 2、View的滑动（常见）

**1、使用scrollTo/scrollBy**

>使用过程：在滑动的过程中，mScrollX的值总是等于View左边缘和View内容左边缘在水平方向的距离，而mScrollY的值总是等于View上边缘和View内容上边缘在垂直方向的距离。（View边缘是指View的位置，由四个顶点组成，而View的内容边缘是指View中的内容的边缘）

**注意：**

1. scrollBy实际上也是调用scrollTo方法，它实现了基于当前位置的相关滑动，而scrollTo则实现了基于所传递的参数的绝对滑动
2. 使用scrollBy和scrollTo来实现View的滑动，只能将View的内容进行移动，并不能将View本身进行移动。(不管怎么滑动，也不可能将当前View滑动到附近View所在的区域)
3. 这两种方法只能修改View内容的位置而不能修改View在布局中的位置。


**2、 使用动画**

**总结：**

>1. 使用动画来做View的滑动，View动画是对View的影响做操作，它并不能真正的改变View的位置参数，包括宽/高，并且如果希望动画后的状态得以保留还必须将fillAfter属性设置为true，否则动画完成后其动画结果会消失。（View动画并不能真正改变View的位置，新的位置只是View的影像而已）
>2. 从Android3.0开始，使用属性动画可以解决上面的问题，但是大多数应用都需要兼容到Android2.2，在Android2.2上无法使用属性动画。
>3. 针对View动画的问题，我们可以在新的位置预先创建和目标Button一模一样的Button，它们属性和点击事件都一样。当实现了一个就把另一个隐藏掉，间接解决问题。



**3、 改变布局参数**
>通过改变LayoutParams实现滑动，通过这种方式去实现View的滑动同样是一种很灵活的方法，但是需要根据不同情况去做不同的处理。


**4、 各种滑动方式的对比**

**总结：**

>1. ScrollTo/scrollBy：操作简单，适合对View内容的滑动。（内容滑动）
>2. 动画：操作简单，主要适用于没有交互的View和实现复杂的动画效果。（无交互）
>3. 改变布局参数：操作稍微复杂，适用于有交互的View。（有交互）
注意：对于全屏滑动，不能使用getX和getY方法，而使用getRawX和getRawY来获取手指当前的坐标。


### 3、弹性滑动

**1、使用scroller**

**工作原理概括：**

Scroller本身并不能实现View的滑动，需要配合View的computeScroll方法才能完成弹性滑动的效果，它不断地让View重绘，而每次重绘距滑动起始时间会有一个时间间隔，通过这个时间间隔Scroller可以得出View当前滑动的位置，知道滑动位置就可以通过scrollTo方法来完成View的滑动。View的每一次重绘都会导致View进行小幅度的滑动，而多次小幅度滑动就组成了弹性滑动。（当我们构建一个Scroller对象并且调用它的startScroll方法，Scroller内部其实什么都没有做，它只是保存我们传递的几个参数，这个方法的参数含义，startX和startY表示的是滑动的起点，dx和dy表示的是滑动的距离，而duration表示的是滑动的时间（整个滑动过程所需要的时间），注意这里的滑动指内容View的滑动而非View本身的位置的改变。）

**使用Scroller实现弹性滑动的解析：**

当View重绘后会在draw方法中调用computeScroll，而computeScroll又会去向Scroller获取当前的scrollX和scrollY；然后通过scrollTo方法实现滑动；接着又调用postInvalidate方法来进行第二次重绘，这个重绘的过程和第一次重绘的过程一样，还是会导致computeScroll方法被调用；然后继续向Scroller获取当前的scrollX和scrollY，并通过scrollTo的方法滑动到新的位置，如此反复，直到整个滑动过程结束。

**2、通过动画**

动画本身就是一个渐变过程。动画滑动只是一个影像转移，因此需要通过在动画的每一帧到来时获取动画完成的比例，然后再根据这个比例计算出当前View所要的滑动距离。注意这里的滑动针对的是View的内容而非View本身。除了能完成弹性动画以外，还可以实现其他动画效果，在onAnimationUpdate方法中加上我们想要的其他操作。


**3、使用延时策略**

核心思想是通过发送一系列延时消息从而达到一种渐变式的效果，具体可以使用Handler或View的postDelayed方法，也可以使用线程的sleep方法。
postDelayed方法来说，我们可以通过它来延时发送一个消息，然后在消息中来进行View的滑动，如果接连不断地发送这种延时消息，那么就可以实现弹性滑动的效果。
Sleep方法来说，通过在while循环中不断地滑动View和sleep，就可以实现弹性滑动的效果。


### 4、View的事件分发机制

**1. 点击事件的传递规则**
>所谓点击事件的事情分发，其实就是对MotionEvent事件的分发过程，即当一个MotionEvent产生之后，系统需要把这个时间传递给一个具体的View，而这个传递的过程就是分发的过程。点击事件的分发过程由三个重要的方法来共同完成：dispatchTouchEvent、onInterceptTouchEvent和onTouchEvent。

```
1->public boolean dispatchTouchEvent(MotionEvent ev);
```
用来进行事情的分发。如果事情能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否消耗当前事件。

```
2->public boolean onInterceptTouchEvent(MotionEvent event);
```
在上述方法内部调用，用于判断是否拦截某个时间，如果当前View拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。

```
3->public boolean onTouchEvent(MotionEvent event);
```
在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列中，当前View无法再次接受到事件。

**它们的关系用伪代码表示：**
```
public boolean dispatchTouchEvent(MotionEvent ev){
	boolean consume = false;
	if(onInterceptTouchEvent(ev)){
		consume = onTouchEvent(ev);
}else{
	Consume = child.dispatchTouchEvent(ev);
}
Return consume;
}
```
当一个点击事件产生后，它的传递过程遵循如下顺序：Activity->Window->View，即事件总是先传递给Activity，Activity再传递给Window，最后Window再传递给顶级View。顶级View接收到事件后，就会按照事件分发机制去分发事件。（当所有元素都不处理这个事件，最终这个事件会传递给Activity处理，即Activity的onTouchEvent方法被调用）。

**关于事件传递的机制的结论：**

>1. 同一个事件序列都是从手指接触屏幕的那一刻起，到手指离开屏幕的那一刻结束，在这个过程中所产生的系列事件，这个时间序列以down事件开始，中间含有数量不等的move事件，最终以up事件结束。
>2. 正常情况下，一个事件序列只能被一个View拦截且消耗。这一条的原因可以参考（3），因为一旦一个元素拦截了某事件，那么同一个事件序列内的所有事件都会直接交给它处理，因此同一个事件序列中的时间不能分别由两个View同时处理，但是通过特殊手段可以做到，比如一个View将本该自己处理的事件通过onTouchEvent强行传递给其他View处理。
>3. 某个View一旦决定拦截，那么这个事件序列都能由它来处理（如果这个事件序列能够传递给它的话），并且它的onInterceptTouchEvent不会再被调用。即说当一个View决定拦截一个事件后，那么系统会把同一个事件序列内的其他方法都直接交给它来处理，因此就不用再调用这个View的onInterceptTouchEvent去询问它是否要拦截了。
>4. 某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件（onTouchEvent返回了false），那么同一事件序列中的其他事件都不会再交给它处理，并且事件将重新交由它的父元素去处理，即父元素的onTouchEvent会被调用。（事件一旦交给了一个View处理，那么它必须被消耗掉，否则同一个事件序列中剩下的时间就不再交给它来处理了，类似上级交给下级处理，下级没处理完，将由上级去处理。）
>5. 如果View不消耗除ACTION_DOWN以外的其他事情，那么这个点击事件会消失，此时父元素的onTOuchEvent并不会被调用，并且当前View可以持续受到后续的事件，最终这些消失的点击事件会传递给Activity处理。
>6. ViewGroup默认不拦截任何时间。Android源码ViewGroup的onInterceptTouchEvent方法默认返回false。
>7. View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，那么它的onTouchEvent方法就会被调用。
>8. View的onTouchEvent默认都会消耗事件（返回true），除非它是不可点击（clickable和longClickable同时为false）。View的longClickable属性默认为false，clickable属性要分情况，比如Button的clickable属性默认为true，而TextView的clickable属性默认为false。
>9. View的enable属性不影响onTouchEvent的默认返回值。哪怕一个View是disable的状态，只要它的clickable或longClickable有一个为true，那么它的onTOuchEvent就会返回true。
>10. onCLick会发生的前提是当前View是可点击的，并且它收到了down和up的事件。
>11. 事件传递过程是由外向内的，即事件总是先传递给父元素，然后再由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的事件分发过程，但是ACTION_DOWN事件除外。

**2. 事件分发的源码解析**

1、Activity对点击事件的分发过程

点击事件用MotionEvent来表示，当一个点击操作发生时，时间最先传递给当前Activity，由Activity的dispatchTouchEvent来进行事件派发，具体的工作是由Activity内部的Window来完成。
Window会将事件传递给décor view， décor view一般就是当前界面的底层容器（即setContentView所设置的View的父容器），通过Activity.getWindow.getDecorView（）可以获得。

**源码：**
```
public boolean dispatchTouchEvent(MotionEvent ev){
	if(ev.getAction() == MotionEvent.ACTION_DOWN){
		onUserInteraction();
  }
  if(getWindow().superDispatchTouchEvent(ev)){
  	return true;
  }
  return onTouchEvent(ev);
}
```

首先事件交给Activity所附属的Window进行分发，如果返回true，整个事件循环就结束了，返回false就意味着事件没人处理，所有View的onTouchEvent都返回false，那么Activity的onTouchEvent就会被调用。


2、Window对点击事件的分发过程

Window的superDispatchTouchEvent是抽象方法，根据源码Window的说明查询，Window的实现类是PhoneWindow。

**源码：**
```
public boolean superDispatchTouchEvent(MotionEvent event){
	  return mDecor. superDispatchTouchEvent(event);
}
```
PhoneWindow将事件直接传递给DecorView(getWindow().getDecorView()返回的View，其继承FrameLayout且是一个父View)。

从这里开始事件已经传递到顶级View了，即在Activity中通过setContentView所设置的View，另外顶级View也叫根View，顶级View，一般来说都是ViewGroup。

3、顶级View对点击事件的分发过程

点击事件如果达到顶级View（一般是一个ViewGroup）以后，会调用ViewGroup的dispatchTouchEvent方法，然后逻辑是这样：如果顶级ViewGroup拦截事件即onInterceptTouchEvent返回true，则事件由ViewGroup处理，这时如果ViewGroup拦截的mOnTouchListener被设置，则onTouch会被调用，否则onTouchEvent会被调用。如果提供的话，onTouch会屏蔽掉onTouchEvent。在onTouchEvent中，如果设置了mOnClickListener，则onClick会被调用。如果顶级ViewGroup不拦截事件，则事件会传递给它所在的点击事件链上的子View，这时子View的dispatchTouchEvent会被调用。

到此为止，事件已经从顶级View传递到了下一层View，接下来的传递过程和顶级View是一直的，如此循环，完成整个事件的分发。

首先ViewGroup对点击事件的分发过程，其主要实现在ViewGroup的dispatchTouchEvent方法中。

**源码：**
```
// Check for interception.
final boolean intercepted;
if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
Final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
   if (!disallowIntercept) {
        intercepted = onInterceptTouchEvent(ev);
        ev.setAction(action); // restore action in case it was changed
   } else {
            intercepted = false;
   }
   } else {
     // There are no touch targets and this action is not an initial down
     // so this view group continues to intercept touches.
     intercepted = true;
}
```
ViewGroup在事件类型为ACTION_DOWN或者mFirstTouchTarget!=null，这两种情况下判断是否拦截当前事件。当事件由ViewGroup的子元素成功处理时，mFirstTouchTarget会被复制并指向子元素（当ViewGroup不拦截事件并将事件交由子元素处理时，mFirstTouchTarget != null），反过来则不成立。当ACTION_MOVE和ACTION_UP事件到来时，由于（actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null）这个条件为false，将导致ViewGroup的onInterceptTouchEvent不会再被调用，并且在同一序列中的其他事件都会默认交给它处理。

特殊情况：FLAG_DISALLOW_INTERCEPT标记位，这个标记位是通过requestDisallowInterceptTouchEvent方法来设置的，一般用于子View中。当其一旦设置后，ViewGroup将无法拦截除了ACTION_DOWN以外的其他点击事件。

**源码：**
```
// Handle an initial down.
if (actionMasked == MotionEvent.ACTION_DOWN) {
    // Throw away all previous state when starting a new touch gesture.
    // The framework may have dropped the up or cancel event for the previous gesture
    // due to an app switch, ANR, or some other state change.
    cancelAndClearTouchTargets(ev);
    resetTouchState();
}
```
ViewGroup会在ACTION_DOWN事件到来时重置状态的操作，而在resetTouchSate方法中会对FLAG_DIALLOW_INTERCEPT进行重置，因此子View调用requestDisallowInterceptTouchEvent方法并不能影响ViewGroup对ACTION_DOWN事件的处理。

总结起来就是两点：第一点，onInterceptTouchEvent不是每次事件都会被调用，如果我们想提前处理所有的点击事件，要选择dispatchTouchEvent方法，只有这个方法能够确保每次都会被调用，前提是事件能够传递到当前的ViewGroup；另外一点，FLAG_DISALLOW_INTERCEPT标记位的作用，当面对滑动冲突时，可以考虑用这种方法去解决问题。

当ViewGroup不拦截事件的时候，事件会向下分发交由它的子View进行处理。

**源码：**
```
final View[] children = mChildren;
for (int i = childrenCount - 1; i >= 0; i--) {
    final int childIndex = customOrder
            ? getChildDrawingOrder(childrenCount, i) : i;
    final View child = (preorderedList == null)
            ? children[childIndex] : preorderedList.get(childIndex);

    // If there is a view that has accessibility focus we want it
    // to get the event first and if not handled we will perform a
    // normal dispatch. We may do a double iteration but this is
    // safer given the timeframe.
    if (childWithAccessibilityFocus != null) {
        if (childWithAccessibilityFocus != child) {
            continue;
        }
        childWithAccessibilityFocus = null;
        i = childrenCount - 1;
    }

    if (!canViewReceivePointerEvents(child)
            || !isTransformedTouchPointInView(x, y, child, null)) {
        ev.setTargetAccessibilityFocus(false);
        continue;
    }

    newTouchTarget = getTouchTarget(child);
    if (newTouchTarget != null) {
        // Child is already receiving touch within its bounds.
        // Give it the new pointer in addition to the ones it is handling.
        newTouchTarget.pointerIdBits |= idBitsToAssign;
        break;
    }

    resetCancelNextUpFlag(child);
    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
        // Child wants to receive touch within its bounds.
        mLastTouchDownTime = ev.getDownTime();
        if (preorderedList != null) {
            // childIndex points into presorted list, find original index
            for (int j = 0; j < childrenCount; j++) {
                if (children[childIndex] == mChildren[j]) {
                    mLastTouchDownIndex = j;
                    break;
                }
            }
        } else {
            mLastTouchDownIndex = childIndex;
        }
        mLastTouchDownX = ev.getX();
        mLastTouchDownY = ev.getY();
        newTouchTarget = addTouchTarget(child, idBitsToAssign);
        alreadyDispatchedToNewTouchTarget = true;
        break;
    }

    // The accessibility focus didn't handle the event, so clear
    // the flag and do a normal dispatch to all children.
    ev.setTargetAccessibilityFocus(false);
}
```
首先遍历ViewGroup的所有子元素，然后判断子元素是否能够接收到点击事件。是否能够接收到点击事件主要由两点来衡量：子元素是否在播动画和点击事件的坐标是否落在子元素的区域内。如果某个元素满足这两个条件，那么事件就会传递给它处理。

dispatchTransformedTouchEvent实际上调用的就是字元素的dispatchTouchEvent方法。

**源码：**
```
if(child == null){
	handled = super.dispatchTouchEvent(event);
} else {
	handled = child.dispatchTouchEvent(event);
}
```
如果子元素的dispatchTouchEvent返回true，这时我们暂时不用考虑事件在子元素内部是怎么分发的，那么mFirstTouchTarget就会被赋值同时跳出for循环，具体如下：
```
newTouchTarget = addTouchTarget(child, idBitsToAssign);
alreadyDispatchedToNewTouchTarget = true;
break;
```
上面完成了mFirstTouchTarget的赋值并终止了对子元素的遍历。如果子元素的dispatchTouchEvent返回false，ViewGroup就会把事件分发给下一个子元素（如果还存在下一个子元素）。

mFirstTouchTarget真正的赋值过程是在addTouchTarget内部完成的，从下面的addTouchTarget方法的内部可以看出，mFirstTouchTarget是一种单链表结构。mFirstTouchTarget是否被复制，将直接影响到ViewGroup对事件的拦截策略，如果mFirstTouchTarget为null，那么ViewGroup就默认拦截接下来同一序列中所有的点击事件。

**源码：**
```
private TouchTarget addTouchTarget(View child, int pointerIdBits) {
    TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
    target.next = mFirstTouchTarget;
    mFirstTouchTarget = target;
    return target;
 }
```
如果遍历所有的子元素后事件都没有被合适地处理，这存在两种情况：第一种是ViewGroup没有子元素；第二种是子元素处理了点击事件，但是在dispatchTouchEvent中返回false，这一版都是因为子元素onTouchEvent中返回false。在这两种情况下，ViewGroup会自己处理点击事件。

**源码：**
```
// Dispatch to touch targets.
if (mFirstTouchTarget == null) {
    // No touch targets so treat this as an ordinary view.
    handled = dispatchTransformedTouchEvent(ev, canceled, null,
            TouchTarget.ALL_POINTER_IDS);
}
```
这里第三个参数child为null，从前面分析，可以知道，它会调用super.dispatchTouchEvent(event)。这里就转到了View的dispatchTouchEvent方法，即点击事件开始交由View来处理。

4、View对点击事件的处理过程

注意这里的View不包含ViewGroup。
```
public boolean dispatchTouchEvent(MotionEvent event) {
		。。。。。。。。
    boolean result = false;
		。。。。。。。。
    if (onFilterTouchEventForSecurity(event)) {
        //noinspection SimplifiableIfStatement
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }

        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
		。。。。。。。。
    return result;
}
```
View对点击事件的处理过程，首先会判断有没有设置OnTouchListener，如果onTouchListener中的onTouch方法返回true，那么onTouchEvent就不会被调用。可见OnTouchListener的优先级高于onTouchEvent，这样做的好处是方便在外界处理点击事件。

不可用状态下的View照样会消耗点击事件。

**源码：**
```
if ((viewFlags & ENABLED_MASK) == DISABLED) {
    if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
        setPressed(false);
    }
    // A disabled view that is clickable still consumes the touch
    // events, it just doesn't respond to them.
    return (((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
}
```
接着，如果View设置代理，还会执行TouchDelegate的onTouchEvent方法，类似onTouchListener。

**源码：**
```
if (mTouchDelegate != null) {
    if (mTouchDelegate.onTouchEvent(event)) {
        return true;
    }
}
```
onTouchEvent中对点击事件的具体处理，如下。

**源码：**
```
if (((viewFlags & CLICKABLE) == CLICKABLE ||
            (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
            (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    。。。。。。

                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // This is a tap, so remove the longpress check
                        removeLongPressCallback();

                        // Only perform take click actions if we were in the pressed state
                        if (!focusTaken) {
                            // Use a Runnable and post this rather than calling
                            // performClick directly. This lets other visual state
                            // of the view update before click actions start.
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }
                    。。。。。
                }
                break;
        }

        return true;
    }
```
只要View的CLICKABLE和LONG_CLICKABLE有一个位true，那么它就会消耗这个事件，即onTouchEvent方法返回true。无论是不是DISABLE状态。当ACTION_UP事件发生时，会触发performClick方法，如果View设置了OnCLickListener，那么performClick方法内部会调用它的onClick方法，如下显示。

**源码：**
```
public boolean performClick() {
    final boolean result;
    final ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnClickListener != null) {
        playSoundEffect(SoundEffectConstants.CLICK);
        li.mOnClickListener.onClick(this);
        result = true;
    } else {
        result = false;
    }

    sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
    return result;
}
```
View的LONG_CLICKABLE属性默认为false，而CLICKABLE属性是否为false和具体的View有关。可点击的View的CLICKABLE为true，反之为false。通过setClickablehe和setLongClickable可改变对应的属性。setOnClickListener会自动将View的CLICKABLE设为true，setOnLongClickListener则会自动设置View的LONG_CLICKABLE为true。

**源码：**
```
//CLICK_ABLE
public void setOnClickListener(@Nullable OnClickListener l) {
    if (!isClickable()) {
        setClickable(true);
    }
    getListenerInfo().mOnClickListener = l;
}
//LONG_CLICKABLE
public void setOnLongClickListener(@Nullable OnLongClickListener l) {
    if (!isLongClickable()) {
        setLongClickable(true);
    }
    getListenerInfo().mOnLongClickListener = l;
}
```

### 5、View的滑动冲突

**1、 常见的滑动冲突场景**

1. 外部滑动方向和内部滑动方向不一致

>ViewPager和Fragment（包含一个ListView）组合的页面滑动效果，本来这种情况下是有滑动冲突的，但是ViewPager内部处理这种滑动冲突。若使用的ScrollView，那就必须手动处理滑动冲突了，否则造成的后果就是内外两层只能有一层能够滑动。除了这种典型的情况，还有外部上下滑动、内部左右滑动等。属于同一类滑动冲突。

2. 外部滑动方向和内部方向一致

>当内外两层都在同一个方向可以滑动的时候，显然存在逻辑问题。因为当手指开始滑动的时候，系统无法知道用户到底是让那一层滑动，所以当手指滑动的时候就会出现问题，要么只有一层能滑动，要么就是内外两层都滑动得很卡顿。在实际开发中，这种场景主要是指内外两层同时能上下滑动或内外两层同时能左右滑动。

3. 1和2的情况嵌套

>内层有一个场景1中的滑动效果，然后外层又有一个场景2中的滑动效果。它是几个单一的滑动冲突的叠加，因此只需要分别处理内层和中层、中层和外层之间的滑动冲突即可。具体处理方式和1、2是相同的。


**2、 滑动冲突的处理规则**

对于场景1，处理规则是：当用户左右滑动时，需要让外部的View拦截点击事件，当用户上下滑动时，需要让内部View拦截点击事件。根据特征解决滑动冲突，具体来说是，根据滑动时水平滑动还是垂直滑动来判断由谁来拦截事件，根据滑动过程中两点之间的坐标就可以得出到底是水平滑动还是垂直滑动。

**比如：**
1)	根据滑动路径和水平所形成的夹角
2)	依据水平方向和垂直方向上的距离差来判断
3)	某些特殊情况还可以依据水平和垂直方向的速度差来判断。

对于场景二，比较特殊，它无法根据滑动的角度、距离差以及速度差来判断，但是这个时候一般都能在业务上找到突破点，比如业务上规定：当处于某种状态时需要外部View响应用户的滑动，而处于另外一种状态时则需要内部View来响应View的滑动，根据这种业务上的需求我们也可以得到相应的处理规则，有了处理规则同样可以进行下一步处理。

对于场景3，它的滑动规则就更复杂了，它无法根据滑动的角度、距离差以及速度差来判断，但是这个时候一般都能在业务上找到突破点，具体方法和场景二一样。


**3、 滑动冲突的解决方式**

1、外部拦截法（必须重写父容器的onInterceptTouchEvent方法，在内部拦截处理）

所谓的外部拦截法是指点击事情都先经过父容器的拦截处理，如果父容器需要此事件就拦截，如果不需要此事件就不拦截，这样就可以解决滑动冲突的问题，这种方法比较符合点击事件的分发机制。

**伪代码演示：**
```
public boolean onInterceptTouchEvent(MotionEvent event){
        boolean intercepted = false;
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()){
            case MotionEvent.ACTION_DOWN:{
                intercepted = false;
                break;
            }
            case MotionEvent.ACTION_MOVE:{
                if(父容器需要当前点击事件){
                    intercepted = true;
                } else {
                    intercepted = false;
                }
                break;
            }
            case MotionEvent.ACTION_UP:{
                intercepted = false;
                break;
            }
            default:
                break;
        }
        mLastXintercept = x;
        mLastYintercept = y;
        return intercepted;
    }
```
代码描述，首先是ACTION_DOWN这个事件，父容器必须返回false，即不拦截ACTION_DOWN事件，这个因为一旦父容器拦截了ACTION_DOWN，那么后续的ACTION_MOVE和ACTION_UP事件都会直接交由父容器处理，这个时候事件没法再传递给子元素了；其次是ACTION_MOVE事件，这个时间可以根据需要来决定是否拦截，如果父容器需要拦截就返回true，否则返回false；最后是ACTION_UP事件，这里必须要返回false，因为ACTION_UP事件本身没有太多意义。

2、内部拦截法

内部拦截法是指父容器不拦截任何时间，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗掉，否则就交由父容器进行处理，这种方法和Android中的时间分发机制不一致，需要配合requestDisallowInterceptTouchEvent方法才能正常工作，使用起来较外部拦截法稍显复杂。

**伪代码演示：**
```
public boolean dispatchTouchEvent(MotionEvent event){
        int x = (int) event.getX();
        int y = (int) event.getY();
        switch (event.getAction()){
            case MotionEvent.ACTION_DOWN:{
                parent.requestDisallowInterceptTouchEvent(true);
                break;
            }
            case MotionEvent.ACTION_MOVE:{
                int deltaX = x - mLastX;
                int deltaY = y - mLastY;
                if(父容器需要此类点击事件){
                    parent.requestDisallowInterceptTouchEvent(false);
                }
                break;
            }
            case MotionEvent.ACTION_UP:{
                intercepted = false;
                break;
            }
            default:
                break;
        }
        mLastX = x;
        mLastY = y;
        return super.dispatchTouchEvent(event);
    }
```

内部拦截法的典型代码，当面对不同的滑动策略时指需要修改里面的条件即可，其他不需要做改动而且也不能有改动。除了子元素需要做处理以外，父元素也要默认拦截除了ACTION_DOWN以外的其他事件，这样当子元素调用parent.requestDisallowInterceptTouchEvent(false)方法时，父元素才能继续拦截所需的事件。

ACTION_DOWN事件并不受FLAG_DISALLOW_INTERCEPT这个标记位的控制，所以一旦父容器拦截ACTION_DOWN事件，那么所有的事件都无法传递到子元素中去，这样内部拦截就无法起到作用了。

**父元素的修改：**
```
public boolean onInterceptTouchEvent(MotionEvent event) {
        int action = event.getAction();
        if (action == MotionEvent.ACTION_DOWN) {
            return false;
        } else {
            return true;
        }
    }
```
>推荐采用外部拦截发来解决常见的滑动冲突
