ViewDragHelper源码解析
＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝＝

### 1.简介
我们了解了`ViewDragHelper`是可以帮助我们处理各种拖拽事件的类.使用好`ViewDragHelper`能帮助我们做出各种酷炫的交互,今天我们就来分析一下`ViewDragHelper`的使用与实现:

### 2.使用方法

我们这里就以[翔总的这篇文章](http://blog.csdn.net/lmj623565791/article/details/46858663)中的例子来介绍一下`ViewDragHelper`的使用.另外,本文中的demo可以在
[这里找到](https://github.com/Skykai521/ViewDragHelperDemo)

首先我们创建一个`DragLayout`类并继承自`LinearLayout`,然后我们准备在`DragLayout`放置三个`View`第一个用来被我们拖动然后停止在松手的位置,第二个可以被我们拖动,松手的时候滑动到指定位置,第三个只可以通过触摸边缘来进行拖动,

```java

public class DragLayout extends LinearLayout {

    private ViewDragHelper mDragger;
    private View mDragView;
    private View mAutoBackView;
    private View mEdgeTrackerView;

    private Point mAutoBackOriginPos = new Point();

    public DragLayout(Context context) {
        this(context, null);
    }

    public DragLayout(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public DragLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        initViewDragHelper();
    }

    private void initViewDragHelper() {
        mDragger = ViewDragHelper.create(this,myCallback);
        mDragger.setEdgeTrackingEnabled(ViewDragHelper.EDGE_ALL);
    }

    ViewDragHelper.Callback myCallback = new ViewDragHelper.Callback() {
        @Override
        //child为当前触摸区域下的View,如果返回true,就可以拖拽.
        public boolean tryCaptureView(View child, int pointerId) {
            return child == mDragView || child == mAutoBackView;
        }

        //松手时的回调
        @Override
        public void onViewReleased(View releasedChild, float xvel, float yvel) {
            if (releasedChild == mAutoBackView) {
                mDragger.settleCapturedViewAt(mAutoBackOriginPos.x, mAutoBackOriginPos.y);
                invalidate();
            }
        }
        
        //边缘触摸开始时的回调
        @Override
        public void onEdgeDragStarted(int edgeFlags, int pointerId) {
            mDragger.captureChildView(mEdgeTrackerView, pointerId);
        }

        //获取水平方向允许拖拽的区域,这里是父布局的宽-子控件的宽
        @Override
        public int getViewHorizontalDragRange(View child) {
            return getMeasuredWidth() - child.getMeasuredWidth();
        }

        //获取垂直方向允许拖拽的范围
        @Override
        public int getViewVerticalDragRange(View child) {
            return getMeasuredHeight() - child.getMeasuredHeight();
        }

        //left为child即将移动到的水平位置的值,但是返回值会最终决定移动到的值
        //这里直接返回了left
        @Override
        public int clampViewPositionHorizontal(View child, int left, int dx) {
            return left;
        }
        //同上只是这里是垂直方向
        @Override
        public int clampViewPositionVertical(View child, int top, int dy) {
            return top;
        }
    };

    @Override
    public void computeScroll() {
        if (mDragger.continueSettling(true)) {
            invalidate();
        }
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return mDragger.shouldInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        mDragger.processTouchEvent(event);
        return true;
    }

    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        mDragView = getChildAt(0);
        mAutoBackView = getChildAt(1);
        mEdgeTrackerView = getChildAt(2);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        super.onLayout(changed, l, t, r, b);
        mAutoBackOriginPos.x = mAutoBackView.getLeft();
        mAutoBackOriginPos.y = mAutoBackView.getTop();
    }
}

```
1. 我们首先在构造方法里传入了当前类的对象和我们定义的`ViewDragHelper.Callback`对象初始化了我们的`ViewDragHelper`,然后我们希望所有的边缘触摸都能触发`mEdgeTrackerView`的拖动,所以我们紧接着调用了`mDragger.setEdgeTrackingEnabled(ViewDragHelper.EDGE_ALL);`方法.
2. 在我们定义的`Callback`中,有多个回调方法,每个回调方法都有它的作用,在代码里注释比较清楚了，我们下面也会解析每一个`Callback`中回调方法的作用.
3. 第三步我们需要在`onInterceptTouchEvent()`方法和`onTouchEvent()`将事件委托给`ViewDragHelper`去处理,这样`ViewDragHelper`才能根据响应的事件并回调我们自己编写的`Callback`接口来进行响应的处理,
4. 由于`ViewDragHelper`中的滑动是交给`Srcoller`类来处理的所以这里我们要重写`computeScroll()`方法,配合`Scroller`完成滚动动画.
5. 最后在`onFinishInflate()`里获取到我们的`View`对象即可.

### 3.类关系图

由于就一个类类图我们就不画了,但是作为一个强迫症患者,这个标题必须有...

### 4.源码分析
#### 1.ViewDragHelper.Callback的实现
在分析`ViewDragHelper`之前,我们先来分析一下`Callback`的定义,看看`Callback`都定义了哪些方法:
```java

    public static abstract class Callback {

        //当View的拖拽状态改变时回调,state为STATE_IDLE,STATE_DRAGGING,STATE_SETTLING的一种
        //STATE_IDLE:　当前未被拖拽
        //STATE_DRAGGING：正在被拖拽
        //STATE_SETTLING:　被拖拽后需要被安放到一个位置中的状态
        public void onViewDragStateChanged(int state) {}

        //当View被拖拽位置发生改变时回调
        //changedView ：被拖拽的View
        //left : 被拖拽后View的left边缘坐标
        //top : 被拖拽后View的top边缘坐标
        //dx : 拖动的x偏移量
        //dy : 拖动的y偏移量
        public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {}

        //当一个View被捕获到准备开始拖动时回调,
        //capturedChild : 捕获的View
        //activePointerId : 对应的PointerId
        public void onViewCaptured(View capturedChild, int activePointerId) {}

        //当被捕获拖拽的View被释放是回调
        //releasedChild : 被释放的View
        //xvel : 释放View的x方向上的加速度
        //yvel : 释放View的y方向上的加速度
        public void onViewReleased(View releasedChild, float xvel, float yvel) {}

        //如果parentView订阅了边缘触摸,则如果有边缘触摸就回调的接口
        //edgeFlags : 当前触摸的flag 有: EDGE_LEFT,EDGE_TOP,EDGE_RIGHT,EDGE_BOTTOM
        //pointerId : 用来描述边缘触摸操作的id
        public void onEdgeTouched(int edgeFlags, int pointerId) {}

        //是否锁定该边缘的触摸,默认返回false,返回true表示锁定
        public boolean onEdgeLock(int edgeFlags) {
            return false;
        }

        //边缘触摸开始时回调
        //edgeFlags : 当前触摸的flag 有: EDGE_LEFT,EDGE_TOP,EDGE_RIGHT,EDGE_BOTTOM
        //pointerId : 用来描述边缘触摸操作的id
        public void onEdgeDragStarted(int edgeFlags, int pointerId) {}

        //在寻找当前触摸点下的子View时会调用此方法，寻找到的View会提供给tryCaptureViewForDrag()来尝试捕获。
        //如果需要改变子View的遍历查询顺序可改写此方法，例如让下层的View优先于上层的View被选中。
        public int getOrderedChildIndex(int index) {
            return index;
        }

        //获取被拖拽View child 的水平拖拽范围,返回0表示无法被水平拖拽
        public int getViewHorizontalDragRange(View child) {
            return 0;
        }

        //获取被拖拽View child 的垂直拖拽范围,返回0表示无法被水平拖拽
        public int getViewVerticalDragRange(View child) {
            return 0;
        }

        //尝试捕获被拖拽的View
        public abstract boolean tryCaptureView(View child, int pointerId);

        //决定拖拽View在水平方向上应该移动到的位置
        //child : 被拖拽的View
        //left : 期望移动到位置的View的left值
        //dx : 移动的水平距离
        //返回值 : 直接决定View在水平方向的位置
        public int clampViewPositionHorizontal(View child, int left, int dx) {
            return 0;
        }

        //决定拖拽View在垂直方向上应该移动到的位置
        //child : 被拖拽的View
        //top : 期望移动到位置的View的top值
        //dy : 移动的垂直距离
        //返回值 : 直接决定View在垂直方向的位置
        public int clampViewPositionVertical(View child, int top, int dy) {
            return 0;
        }
    }

```
想必注释已经很清楚了,正是这些回调方法,再结合`ViewDragHelper`中的各种方法,来帮助我们实现各种各样的拖拽的效果。

#### 2.shouldInterceptTouchEvent()方法的实现

在这里我们假设大家都清楚了`Android`的事件分发机制,如果不清楚请看[这里](http://blog.csdn.net/guolin_blog/article/details/9097463),要想处理触摸事件,我们需要在`onInterceptTouchEvent(MotionEvent ev)`方法里判断是否需要拦截这次触摸事件,如果此方法返回`true`则触摸事件将会交给`onTouchEvent(MotionEvent event)`处理,这样我们就能处理触摸事件了,所以我们在上面的使用方法里会这样写:

```java

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return mDragger.shouldInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        mDragger.processTouchEvent(event);
        return true;
    }
```
这样就将是否拦截触摸事件,以及处理触摸事件委托给`ViewDragHelper`来处理了,所以我们先来看看`ViewDragHelper`中`shouldInterceptTouchEvent();`方法的实现:

```java

    public boolean shouldInterceptTouchEvent(MotionEvent ev) {
        //获取action
        final int action = MotionEventCompat.getActionMasked(ev);
        //获取action对应的index
        final int actionIndex = MotionEventCompat.getActionIndex(ev);

        //如果是按下的action则重置一些信息,包括各种事件点的数组
        if (action == MotionEvent.ACTION_DOWN) {
            // Reset things for a new event stream, just in case we didn't get
            // the whole previous stream.
            cancel();
        }
        //初始化mVelocityTracker
        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);

        //根据action来做相应的处理
        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                final float x = ev.getX();
                final float y = ev.getY();
                //获取这个事件对应的pointerId,一般情况下只有一个手指触摸时为0
                //两个手指触摸时第二个手指触摸返回的pointerId为1，以此类推
                final int pointerId = MotionEventCompat.getPointerId(ev, 0);
                //保存点的数据
                //TODO (1)
                saveInitialMotion(x, y, pointerId);
                //获取当前触摸点下最顶层的子View
                //TODO (2)
                final View toCapture = findTopChildUnder((int) x, (int) y);

                //如果toCapture是已经捕获的View,而且正在处于被释放状态
                //那么就重新捕获
                if (toCapture == mCapturedView && mDragState == STATE_SETTLING) {
                    tryCaptureViewForDrag(toCapture, pointerId);
                }

                //如果触摸了边缘,回调callback的onEdgeTouched()方法
                final int edgesTouched = mInitialEdgesTouched[pointerId];
                if ((edgesTouched & mTrackingEdges) != 0) {
                    mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
                }
                break;
            }

            //当又有一个手指触摸时
            case MotionEventCompat.ACTION_POINTER_DOWN: {
                final int pointerId = MotionEventCompat.getPointerId(ev, actionIndex);
                final float x = MotionEventCompat.getX(ev, actionIndex);
                final float y = MotionEventCompat.getY(ev, actionIndex);

                //保存触摸信息
                saveInitialMotion(x, y, pointerId);

                //因为同一时间ViewDragHelper只能操控一个View,所以当有新的手指触摸时
                //只讨论当无触摸发生时,回调边缘触摸的callback
                //或者正在处于释放状态时重新捕获View
                if (mDragState == STATE_IDLE) {
                    final int edgesTouched = mInitialEdgesTouched[pointerId];
                    if ((edgesTouched & mTrackingEdges) != 0) {
                        mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
                    }
                } else if (mDragState == STATE_SETTLING) {
                    // Catch a settling view if possible.
                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    if (toCapture == mCapturedView) {
                        tryCaptureViewForDrag(toCapture, pointerId);
                    }
                }
                break;
            }

            //当手指移动时
            case MotionEvent.ACTION_MOVE: {
                if (mInitialMotionX == null || mInitialMotionY == null) break;

                // First to cross a touch slop over a draggable view wins. Also report edge drags.
                //得到触摸点的数量,并循环处理,只处理第一个发生了拖拽的事件
                final int pointerCount = MotionEventCompat.getPointerCount(ev);
                for (int i = 0; i < pointerCount; i++) {
                    final int pointerId = MotionEventCompat.getPointerId(ev, i);
                    final float x = MotionEventCompat.getX(ev, i);
                    final float y = MotionEventCompat.getY(ev, i);
                    //获得拖拽偏移量
                    final float dx = x - mInitialMotionX[pointerId];
                    final float dy = y - mInitialMotionY[pointerId];
                    //获取当前触摸点下最顶层的子View
                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    //如果找到了最顶层View,并且产生了拖动(checkTouchSlop()返回true)
                    //TODO (3)
                    final boolean pastSlop = toCapture != null && checkTouchSlop(toCapture, dx, dy);
                    if (pastSlop) {
                        //根据callback的四个方法getView[Horizontal|Vertical]DragRange和
                        //clampViewPosition[Horizontal|Vertical]来检查是否可以拖动
                        final int oldLeft = toCapture.getLeft();
                        final int targetLeft = oldLeft + (int) dx;
                        final int newLeft = mCallback.clampViewPositionHorizontal(toCapture,
                                targetLeft, (int) dx);
                        final int oldTop = toCapture.getTop();
                        final int targetTop = oldTop + (int) dy;
                        final int newTop = mCallback.clampViewPositionVertical(toCapture, targetTop,
                                (int) dy);
                        final int horizontalDragRange = mCallback.getViewHorizontalDragRange(
                                toCapture);
                        final int verticalDragRange = mCallback.getViewVerticalDragRange(toCapture);
                        //如果都不允许移动则跳出循环
                        if ((horizontalDragRange == 0 || horizontalDragRange > 0
                                && newLeft == oldLeft) && (verticalDragRange == 0
                                || verticalDragRange > 0 && newTop == oldTop)) {
                            break;
                        }
                    }
                    //记录并回调是否有边缘触摸
                    reportNewEdgeDrags(dx, dy, pointerId);
                    if (mDragState == STATE_DRAGGING) {
                        // Callback might have started an edge drag
                        break;
                    }
                    //如果产生了拖动则调用tryCaptureViewForDrag()
                    //TODO (4)
                    if (pastSlop && tryCaptureViewForDrag(toCapture, pointerId)) {
                        break;
                    }
                }
                //保存触摸点的信息
                saveLastMotion(ev);
                break;
            }

            //当有一个手指抬起时,清除这个手指的触摸数据
            case MotionEventCompat.ACTION_POINTER_UP: {
                final int pointerId = MotionEventCompat.getPointerId(ev, actionIndex);
                clearMotionHistory(pointerId);
                break;
            }

            //清除所有触摸数据
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL: {
                cancel();
                break;
            }
        }

        //如果mDragState等于正在拖拽则返回true
        return mDragState == STATE_DRAGGING;
    }

```
上面就是整个`shouldInterceptTouchEvent()`的实现,上面的注释也足够清楚了,我们这里就先不分析某一种触摸事件,大家可以看到我上面留了几个TODO,下文会一起分析,这里我假设大家都已经对触摸事件分发处理都有充分的理解了,我们下面就直接看`ViewDragHelper`里`processTouchEvent()`方法的实现.

#### 3.processTouchEvent()方法的实现

```java

    public void processTouchEvent(MotionEvent ev) {
        final int action = MotionEventCompat.getActionMasked(ev);
        final int actionIndex = MotionEventCompat.getActionIndex(ev);

        ...（省去部分代码）
        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                ...（省去部分代码）
                break;
            }

            case MotionEventCompat.ACTION_POINTER_DOWN: {
                ...（省去部分代码）
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                //如果现在已经是拖拽状态
                if (mDragState == STATE_DRAGGING) {
                    final int index = MotionEventCompat.findPointerIndex(ev, mActivePointerId);
                    final float x = MotionEventCompat.getX(ev, index);
                    final float y = MotionEventCompat.getY(ev, index);
                    final int idx = (int) (x - mLastMotionX[mActivePointerId]);
                    final int idy = (int) (y - mLastMotionY[mActivePointerId]);

                    //拖拽至指定位置
                    //TODO (5)
                    dragTo(mCapturedView.getLeft() + idx, mCapturedView.getTop() + idy, idx, idy);

                    saveLastMotion(ev);
                } else {
                    // Check to see if any pointer is now over a draggable view.
                    //如果还不是拖拽状态,就检测是否经过了一个View
                    final int pointerCount = MotionEventCompat.getPointerCount(ev);
                    for (int i = 0; i < pointerCount; i++) {
                        final int pointerId = MotionEventCompat.getPointerId(ev, i);
                        final float x = MotionEventCompat.getX(ev, i);
                        final float y = MotionEventCompat.getY(ev, i);
                        final float dx = x - mInitialMotionX[pointerId];
                        final float dy = y - mInitialMotionY[pointerId];

                        reportNewEdgeDrags(dx, dy, pointerId);
                        if (mDragState == STATE_DRAGGING) {
                            // Callback might have started an edge drag.
                            break;
                        }

                        final View toCapture = findTopChildUnder((int) x, (int) y);
                        if (checkTouchSlop(toCapture, dx, dy) &&
                                tryCaptureViewForDrag(toCapture, pointerId)) {
                            break;
                        }
                    }
                    saveLastMotion(ev);
                }
                break;
            }
            //当多个手指中的一个手机松开时
            case MotionEventCompat.ACTION_POINTER_UP: {
                final int pointerId = MotionEventCompat.getPointerId(ev, actionIndex);
                //如果当前点正在被拖拽,则再剩余还在触摸的点钟寻找是否正在View上
                if (mDragState == STATE_DRAGGING && pointerId == mActivePointerId) {
                    // Try to find another pointer that's still holding on to the captured view.
                    int newActivePointer = INVALID_POINTER;
                    final int pointerCount = MotionEventCompat.getPointerCount(ev);
                    for (int i = 0; i < pointerCount; i++) {
                        final int id = MotionEventCompat.getPointerId(ev, i);
                        if (id == mActivePointerId) {
                            // This one's going away, skip.
                            continue;
                        }

                        final float x = MotionEventCompat.getX(ev, i);
                        final float y = MotionEventCompat.getY(ev, i);
                        if (findTopChildUnder((int) x, (int) y) == mCapturedView &&
                                tryCaptureViewForDrag(mCapturedView, id)) {
                            newActivePointer = mActivePointerId;
                            break;
                        }
                    }

                    if (newActivePointer == INVALID_POINTER) {
                        // We didn't find another pointer still touching the view, release it.
                        //如果没找到则释放View
                        //TODO (6)
                        releaseViewForPointerUp();
                    }
                }
                clearMotionHistory(pointerId);
                break;
            }

            case MotionEvent.ACTION_UP: {
                //如果是拖拽状态的释放则调用
                //releaseViewForPointerUp()
                if (mDragState == STATE_DRAGGING) {
                    releaseViewForPointerUp();
                }
                cancel();
                break;
            }

            case MotionEvent.ACTION_CANCEL: {
                if (mDragState == STATE_DRAGGING) {
                    dispatchViewReleased(0, 0);
                }
                cancel();
                break;
            }
        }
    }

```

上面就是`processTouchEvent()`方法的实现,我们省去了部分大致与`shouldInterceptTouchEvent()`相同的逻辑代码,通过事件传递机制我们知道,如果程序已经进入到`processTouchEvent()`中,也就意味着触摸事件就不会再向下传递,都会交给此方法处理,所以在这里我们就需要处理拖拽事件了,通过上面的注释,我们也看到了在`MotionEvent.ACTION_MOVE`,`MotionEventCompat.ACTION_POINTER_UP`,`MotionEvent.ACTION_UP`和`MotionEvent.ACTION_CANCEL`都分别进行了处理    ,我们知道触摸事件大致的流程是:

    ACTION_DOWN -> ACTION_MOVE -> ... -> ACTION_MOVE -> ACTION_UP

再配合事件的分发机制,我们就能很清晰的分析出一次完整的事件调用过程,所以整个`ViewDragHelper`的拖拽过程也能很清晰的分为三个步骤:

    捕获拖拽目标View -> 拖拽目标View -> 处理目标View释放操作

最后我们再分析上面两段代码的6个TODO:

#### 4.saveInitialMotion()方法

```java
    private void saveInitialMotion(float x, float y, int pointerId) {
        //确保各个数组的大小足够存放数据
        ensureMotionHistorySizeForId(pointerId);
        //保存x坐标
        mInitialMotionX[pointerId] = mLastMotionX[pointerId] = x;
        //保存y坐标
        mInitialMotionY[pointerId] = mLastMotionY[pointerId] = y;
        //保存是否触摸到边缘
        mInitialEdgesTouched[pointerId] = getEdgesTouched((int) x, (int) y);
        //保存当前id是否在触摸,用于后续验证
        mPointersDown |= 1 << pointerId;
    }

```
#### 5.findTopChildUnder()方法

```java
    public View findTopChildUnder(int x, int y) {
        final int childCount = mParentView.getChildCount();
        for (int i = childCount - 1; i >= 0; i--) {
            final View child = mParentView.getChildAt(mCallback.getOrderedChildIndex(i));
            if (x >= child.getLeft() && x < child.getRight() &&
                    y >= child.getTop() && y < child.getBottom()) {
                return child;
            }
        }
        return null;
    }
```
代码很简单就是根据`x`和`y`坐标和来找到指定`View`,注意这里回调了`callback`中的`getOrderedChildIndex()`方法,所以我们可以在这里返回指定的`View`的`index`.

#### 6.checkTouchSlop()方法

```java
    private boolean checkTouchSlop(View child, float dx, float dy) {
        if (child == null) {
            return false;
        }
        final boolean checkHorizontal = mCallback.getViewHorizontalDragRange(child) > 0;
        final boolean checkVertical = mCallback.getViewVerticalDragRange(child) > 0;

        if (checkHorizontal && checkVertical) {
            return dx * dx + dy * dy > mTouchSlop * mTouchSlop;
        } else if (checkHorizontal) {
            return Math.abs(dx) > mTouchSlop;
        } else if (checkVertical) {
            return Math.abs(dy) > mTouchSlop;
        }
        return false;
    }
```
用来根据`mTouchSlop`最小拖动的距离来判断是否属于拖动,`mTouchSlop`根据我们设定的灵敏度决定.

#### 7.tryCaptureViewForDrag()方法

```java

    boolean tryCaptureViewForDrag(View toCapture, int pointerId) {
        //如果已经捕获该View 直接返回true
        if (toCapture == mCapturedView && mActivePointerId == pointerId) {
            // Already done!
            return true;
        }
        //根据mCallback.tryCaptureView()方法来最终决定是否可以捕获View
        if (toCapture != null && mCallback.tryCaptureView(toCapture, pointerId)) {
            mActivePointerId = pointerId;
            //如果可以则调用captureChildView(),并返回true
            captureChildView(toCapture, pointerId);
            return true;
        }
        return false;
    }

```
可以看到如果可以捕获`View`则调用了`captureChildView()`方法:
    
```java

    public void captureChildView(View childView, int activePointerId) {
        if (childView.getParent() != mParentView) {
            throw new IllegalArgumentException("captureChildView: parameter must be a descendant " +
                    "of the ViewDragHelper's tracked parent view (" + mParentView + ")");
        }
        //赋值mCapturedView
        mCapturedView = childView;
        mActivePointerId = activePointerId;
        //回调callback
        mCallback.onViewCaptured(childView, activePointerId);
        //设定mDragState的状态为STATE_DRAGGING
        setDragState(STATE_DRAGGING);
    }

```
如果程序执行到这里,就证明`View`已经处于拖拽状态了,后续的触摸操作,将直接根据`mDragState`为`STATE_DRAGGING`的状态处理.

#### 8.dragTo()方法的实现

```java

    private void dragTo(int left, int top, int dx, int dy) {
        int clampedX = left;
        int clampedY = top;
        final int oldLeft = mCapturedView.getLeft();
        final int oldTop = mCapturedView.getTop();
        if (dx != 0) {
            //回调callback来决定View最终被拖拽的x方向上的偏移量
            clampedX = mCallback.clampViewPositionHorizontal(mCapturedView, left, dx);
            //移动View
            ViewCompat.offsetLeftAndRight(mCapturedView, clampedX - oldLeft);
        }
        if (dy != 0) {
            //回调callback来决定View最终被拖拽的y方向上的偏移量
            clampedY = mCallback.clampViewPositionVertical(mCapturedView, top, dy);
            //移动View
            ViewCompat.offsetTopAndBottom(mCapturedView, clampedY - oldTop);
        }

        if (dx != 0 || dy != 0) {
            final int clampedDx = clampedX - oldLeft;
            final int clampedDy = clampedY - oldTop;
            //回调callback
            mCallback.onViewPositionChanged(mCapturedView, clampedX, clampedY,
                    clampedDx, clampedDy);
        }
    }

```
因为`dragTo()`方法是在`processTouchEvent()`中的`MotionEvent.ACTION_MOVE case`被调用所以当程序运行到这里时`View`就会不断的被拖动了。如果一旦手指释放则最终会调用`releaseViewForPointerUp()`方法

#### 8.releaseViewForPointerUp()方法的实现

```java

    private void releaseViewForPointerUp() {
        //计算出当前x和y方向上的加速度
        mVelocityTracker.computeCurrentVelocity(1000, mMaxVelocity);
        final float xvel = clampMag(
                VelocityTrackerCompat.getXVelocity(mVelocityTracker, mActivePointerId),
                mMinVelocity, mMaxVelocity);
        final float yvel = clampMag(
                VelocityTrackerCompat.getYVelocity(mVelocityTracker, mActivePointerId),
                mMinVelocity, mMaxVelocity);
        dispatchViewReleased(xvel, yvel);
    }

```
计算完加速度后就调用了`dispatchViewReleased()`:

```java

    private void dispatchViewReleased(float xvel, float yvel) {
        //设定当前正处于释放阶段
        mReleaseInProgress = true;
        //回调callback的onViewReleased()方法
        mCallback.onViewReleased(mCapturedView, xvel, yvel);
        mReleaseInProgress = false;

        //设定状态
        if (mDragState == STATE_DRAGGING) {
            // onViewReleased didn't call a method that would have changed this. Go idle.
            //如果onViewReleased()中没有调用任何方法,则状态设定为STATE_IDLE
            setDragState(STATE_IDLE);
        }
    }

```
所以最后释放后的处理交给了`callback`中的`onViewReleased()`方法,如果我们什么都不做,那么这个被拖拽的`View`就是停止在当前位置,或者我们可以调用`ViewDragHelper`提供给我们的这几个方法:

- settleCapturedViewAt(int finalLeft, int finalTop)
以松手前的滑动速度为初速动，让捕获到的View自动滚动到指定位置。只能在Callback的onViewReleased()中调用。
- flingCapturedView(int minLeft, int minTop, int maxLeft, int maxTop)
以松手前的滑动速度为初速动，让捕获到的View在指定范围内fling。只能在Callback的onViewReleased()中调用。
- smoothSlideViewTo(View child, int finalLeft, int finalTop)
指定某个View自动滚动到指定的位置，初速度为0，可在任何地方调用。

引用自[这篇文章](http://www.cnphp6.com/archives/87727),具体释放后的原理我们就不分析了,其实就是配合`Scroller`这个类来实现,具体也可以参照上面这篇文章。好,我们关于`ViewDragHelper`的源码分析就到这里.

### 5.开源项目中的使用

`ViewDragHelper`在各种关于拖拽和各种手势动画的开源库中使用广泛,我这里就简要列出一些,大家可以多去看看是如何使用`ViewDragHelper`的:

- [SwipeBackLayout](https://github.com/ikew0ng/SwipeBackLayout)
- [android-card-slide-panel](https://github.com/xmuSistone/android-card-slide-panel)
- [FlowingDrawer](https://github.com/mxn21/FlowingDrawer)

### 6.个人评价
`ViewDragHelper`的出现,大大简化了我们开发相关触摸和拖拽功能的复杂度和代码量,帮助我们比较容易的实现各种效果,让我们开发酷炫的交互更加容易了。但是从一些开源项目中发现,`ViewDragHelper`中还是有一些不足之处,比如给`Scroller`提供了一个固定的`Interpolator`,导致如果我们想实现例如反弹效果的话,还要把`ViewDragHelper`的代码拷贝一份并修改`Interpolator`,这样做肯定是不太好的.当然建议我们自己修改一个`ViewDragHelper`后如果项目里有多处使用,可以包装成一个提供给我们自己项目的模块使用,防止出现更多的多余代码.
