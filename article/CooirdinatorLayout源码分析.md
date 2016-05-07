# CoordinatorLayout 源码分析

`CoordinatorLayout`有一些很有意思的特性，设置anchor、NestedScroll配合Toolbar/TabLayout的显隐or伸缩、Fab的移动等。今天咱就来一探究竟！

## 1. 从LayoutParam开始

`CoordinatorLayout.LayoutParam`中有一些不太一样的属性和元素，在此先进行介绍。

### 1.1 特殊属性

| 属性 | 对应xml属性|用途 |
|:--:| :--: | :--: |
|`AndchorId`| `layout_anchor` &`layout_anchorGravity`| 布局时根据自身`gravity` 与 `layout_anchorGravity`放置在被anchor的View中 |
|`Behavior` | `layout_behavior` | 辅助Coordinator对View进行layout、nestedScroll的处理 |
|`KeyLine` | `layout_keyline` & `keylines` | 给Coordinator设置了`keylines`（整数数组）后，可以为子View设置`layout_keyline="i"`使其的水平位置根据对应`keylines[i]`进行layout。 |
|`LastChildRect` | 无 | 记录每一次Layout的位置，从而判断是否新的一帧改变了位置 |

注：

`keyline`是一个非常奇怪的属性，我在看源码时才第一次看到到这玩意，网上的资料也非常之少。分析下来，就是如果设置了keyline，那么gravity就会被无视，直接放置在对应的水平位置keyline上。CoordinatorLayout里面也没有其他的特性是根据keyline实现的，个人认为没卵用，本文对它的分析基本都会略过。

### 1.2 依赖关系

假设此时有两个View: **A** 和**B**，那么有两种情况会导致依赖关系：

- **A**的`anchor`是**B** ；
- **A**的`behavior`对**B**有依赖（比如`FloatingActionButton`依赖`SnackBar`)。

依赖关系建立的前提是两个View在同一个`Coordinatorlayout`中。

`CoordinatorLayout`中维护了一个`mDependencySortedChildren`列表，里面含有所有的子View，**按依赖关系排序，被依赖者排在前面**。我们可以看一下用来排序的`Comparator`：

```java
final Comparator<View> mLayoutDependencyComparator = new Comparator<View>() {
    @Override
    public int compare(View lhs, View rhs) {
        if (lhs == rhs) {
            return 0;
        } else if (((LayoutParams) lhs.getLayoutParams()).dependsOn(
                CoordinatorLayout.this, lhs, rhs)) {
            return 1;
        } else if (((LayoutParams) rhs.getLayoutParams()).dependsOn(
                CoordinatorLayout.this, rhs, lhs)) {
            return -1;
        } else {
            return 0;
        }
    }
};
```

>注意，在建立`mDependencySortedChildren`并排序完成之后（在measure的第一步处理完成），每次对子View的遍历**都是通过它进行顺序遍历**，保证了被依赖的View最先被处理。

### 1.3 Behavior

在`CoordinatorLayout`中定义了`Behavior`类，它是用来辅助layout的工具。如果一个CoordinatorLayout的直接子View设置了`Behavior`（或者通过类注解`@DefaultBehavior`指定`Behavior`），则该Behavior会储存在该View的`LayoutParam`中。

> 注意：不是CoordinatorLayout的直接子View，设置Behavior是无效的。你可以看到任何一处对于Behavior的处理都是直接`getChildCount（）`遍历。

在Behavior中有几类功能，我们一一进行介绍：

#### 1.3.1 触摸响应类

`Behavior`中有两个函数：`onInterceptTouchEvent`、 `onTouchEvent`。在CoordinatorLayout每次触发对应事件的时候会选择一个最适合的子View的Behavior执行对应函数。我们来看一下CoordinatorLayout是怎么分发和处理Touch事件的：

***intercept***
```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    MotionEvent cancelEvent = null;

    final int action = MotionEventCompat.getActionMasked(ev);

    // 重置响应的Behavoir
    if (action == MotionEvent.ACTION_DOWN) {
        resetTouchBehaviors();
    }

	// 在这里选择一个最佳Behavior进行处理
    final boolean intercepted = performIntercept(ev, TYPE_ON_INTERCEPT);

    if (cancelEvent != null) {
        cancelEvent.recycle();
    }

	// 重置响应的Behavior
    if (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL) {
        resetTouchBehaviors();
    }

    return intercepted;
}
```

在`performIntercept`去选择一个最适合的Behavior来进行处理，这个方法不仅用于`onInterceptTouchEvent`，并且也用于`onTouchEvent`，根据传入`type`不同来识别对应方法。我们来看看它的逻辑：

```java
private boolean performIntercept(MotionEvent ev, final int type) {
    boolean intercepted = false;
    boolean newBlock = false;

    MotionEvent cancelEvent = null;

    final int action = MotionEventCompat.getActionMasked(ev);

    final List<View> topmostChildList = mTempList1;

    // API>=21时，使用elevation由低到高排列View；API<21时，按View添加顺序排列
    getTopSortedChildren(topmostChildList);

    final int childCount = topmostChildList.size();
    for (int i = 0; i < childCount; i++) {
        final View child = topmostChildList.get(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        final Behavior b = lp.getBehavior();

        // ...(省略代码) 如果此次判定intercept，则对上次的Behavior发送CANCEL事件。

		// 根据传入type不同调用不同的方法
        if (!intercepted && b != null) {
            switch (type) {
                case TYPE_ON_INTERCEPT:
                    intercepted = b.onInterceptTouchEvent(this, child, ev);
                    break;
                case TYPE_ON_TOUCH:
                    intercepted = b.onTouchEvent(this, child, ev);
                    break;
            }
            if (intercepted) {
                mBehaviorTouchView = child;
            }
        }

		//...(省略代码) 如果Behavior.blocksInteractionBelow()返回true，则不处理后续的事件。
    }

    topmostChildList.clear();

    return intercepted;
}
```

#### 1.3.2 依赖关系类

这部分比较简单，就俩函数：
 + `layoutDependsOn`：返回`true`则表示对另一个View有依赖关系；
 + `onDependentViewChanged`&`onDependentViewRemoved`：如果被依赖的View在正常layout之后仍有size/position上的变化，或者被remove掉，都会触发对应方法。

那么问题来了，CoordinatorLayout是怎么监听这个被依赖的View改变的事件的呢？

原来它里面有一个`ViewTreeObserver.OnPreDrawListener`，它在`onMeasure`的时候被添加到了`ViewTreeObserver`中，这样每一帧被绘制出来之前都会调用这个回调。

```java
class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
    @Override
    public boolean onPreDraw() {
        dispatchOnDependentViewChanged(false);
        return true;
    }
}
```

这个`dispatchOnDependentViewChanged`里面代码比较多，就不放上来了，总结下来就是这样：

根据依赖关系遍历子View，对每一个View做如下操作

- 判断一下新的布局边界与lastChildRect是否相同，是则记录新的布局边界为lastChildRect，并继续后续流程，否则跳过；
- 对于之后每一个View，如果它依赖于本View，则调用它的`Behavior.onDependentViewChanged`（如果有Behavior的话）。

至于`onDependentViewRemoved`，是在初始化的时候就会调用`ViewGroup.setOnHierarchyChangeListener()`方法设置一个`OnHierarchyChangeListener`，这样每次add和remove子View的时候就会接收到回调，同时对相应依赖关系的View进行处理。


#### 1.3.3 布局类

 `onMeasureChild`&`onLayoutChild`：如果重写了该方法并返回`true`，则CoordinatorLayout会使用Behavior对这个子View进行measure/layout。具体的可以见下面的**Measure&Layout**

#### 1.3.4 嵌套滑动类

 CoordinatorLayout实现了`NestedScrollingParent`，当CoordinatorLayout内有一个支持NestedScroll的子View时，它的嵌套滑动事件通过`NestedScrollingParent`的回调分发到各直接子View的Behavior处理。虽然`Behavior`类没有实现`NestedScrollingParent`，但是实际上它的方法都有。有兴趣的同学可以去看看这个类，我们这里重点讲CoordinatorLayout的分发过程。

各个事件的分发过程类似，此处就举一个例子：

```java
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    boolean handled = false;

    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        final View view = getChildAt(i);
        final LayoutParams lp = (LayoutParams) view.getLayoutParams();
        final Behavior viewBehavior = lp.getBehavior();
        if (viewBehavior != null) {
            final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child, target,
                    nestedScrollAxes);
            handled |= accepted;

            lp.acceptNestedScroll(accepted);
        } else {
            lp.acceptNestedScroll(false);
        }
    }
    return handled;
}
```

非常简单吧，就遍历一下直接子View，每个都调一下对应的回调方法，只要有任何一个子View的behavior消耗了这个事件，就算消耗了这个事件。

##2. Measure&Layout

我们知道，ViewGroup要把子View准确地放置到屏幕上都是要走`onMeasure` + `onLayout`的，那么我们看看`CoordinatorLayout`在这里干了什么。

在看懂时请确保你明白`measure/layout`的意义以及基本用法，否则可能会导致身体不适=。=

### 2.1 Measure

最直接的就是看代码，如果不喜欢，可以跳过代码看总结。：

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    prepareChildren(); /* 解析依赖关系，并用1.2中提到的Comparator对View按依赖关系进行排序 */
    ensurePreDrawListener(); /* 若PreDrawListener未添加，则添加到ViewTreeObserver */

    //...(省略代码) 解析padding+measureSpec

    final int childCount = mDependencySortedChildren.size();
    for (int i = 0; i < childCount; i++) {
        final View child = mDependencySortedChildren.get(i);
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();

		//...(省略代码)处理keyline

        int childWidthMeasureSpec = widthMeasureSpec;
        int childHeightMeasureSpec = heightMeasureSpec;

		//...(省略代码) 处理由于fitSystemWindows带来的padding

		/* 如果childView有Behavior并且它的onMeasureChild返回true，则由behavior来对childView进行measure，否则就自己measure. */
        final Behavior b = lp.getBehavior();
        if (b == null || !b.onMeasureChild(this, child, childWidthMeasureSpec, keylineWidthUsed,
                childHeightMeasureSpec, 0)) {
            onMeasureChild(child, childWidthMeasureSpec, keylineWidthUsed,
                    childHeightMeasureSpec, 0);
        }

		/* 取最大的child width/height 加上margin 作为已经消耗的尺寸。 */
        widthUsed = Math.max(widthUsed, widthPadding + child.getMeasuredWidth() +
                lp.leftMargin + lp.rightMargin);
        heightUsed = Math.max(heightUsed, heightPadding + child.getMeasuredHeight() +
                lp.topMargin + lp.bottomMargin);

        childState = ViewCompat.combineMeasuredStates(childState,
                ViewCompat.getMeasuredState(child));
    }

	/* 设置自身的measure尺寸 */
    final int width = ViewCompat.resolveSizeAndState(widthUsed, widthMeasureSpec,
            childState & ViewCompat.MEASURED_STATE_MASK);
    final int height = ViewCompat.resolveSizeAndState(heightUsed, heightMeasureSpec,
            childState << ViewCompat.MEASURED_HEIGHT_STATE_SHIFT);
    setMeasuredDimension(width, height);
}
```

总结下来，onMeasure干了这么几件事：

1. 根据依赖关系对所有子View进行排序
2. 保证OnPreDrawListener被添加
3. 按依赖关系遍历子View:
	- 如果子View有Behavior，并且它的`onMeasureChild`返回true，则使用Behavior进行measure；否则直接使用measureSpec对子View进行measure；
	- 取子VIew最大的measure尺寸为已使用的measure尺寸。
4. 更新本身的Measure尺寸。

### 2.2 Layout

在`onLayout`中，我们可以看到`CoordinatorLayout`会对每一个子View依照以下判断顺序进行layout：

1. 如果子View设置了`Behavior`，并且该`Behavior`的`behavior.onLayoutChild`返回`true`，则使用`behavior.onLayoutChild`对该子View进行layout；
2. 如果Behavior不进行layout，则进入自身的`onLayoutChild()`，内部依次进行如下判断：
- 如果子View设置了`Anchor`，则调用`layoutChildWithAnchor`（根据anchor进行layout）；
- 如果子View含有`keyline`，则调用`layoutChildWithKeyline`（根据keyline进行layout）；
- 如果以上判断都不符合，则直接将View根据**padding/margin/measure结果**按照`Gravity`放置。

我们一一来看一下这些过程。

#### 2.2.1 使用Behavior进行layout

默认的Behavior的`onLayoutChild`都是返回`false`的，那么我们看看`FloatingActionButton`的默认Behavior是怎么处理的吧：

``` java
 @Override
 public boolean onLayoutChild(CoordinatorLayout parent, FloatingActionButton child,
         int layoutDirection) {

     // 检查该FAB是否依赖AppBarLayout
     final List<View> dependencies = parent.getDependencies(child);
     for (int i = 0, count = dependencies.size(); i < count; i++) {
         final View dependency = dependencies.get(i);
         if (dependency instanceof AppBarLayout
                 && updateFabVisibility(parent, (AppBarLayout) dependency, child)) {
             break;
         }
     }

     // 调用CoordinatorLayout的onLayoutChild对FAB进行layout
     parent.onLayoutChild(child, layoutDirection);
     // 在API < 21时，需要手动offset来让出阴影的位置
     offsetIfNeeded(parent, child);
     return true;
 }
```

这里主要是处理了如果FAB设置了`AppBarLayout`为anchor时（此时会对`AppBarLayout`有依赖），则当`AppBarLayout`的高度不足以显示FAB时将其隐藏）。

之后它会手动调用CoordinatorLayout自身的`onLayoutChild`方法进行layout，即上述判断的第二步，那我们继续往下看。

#### 2.2.2 使用Anchor进行layout

如果View设置了anchor，那么都会调用`layoutWithAnchor`进行layout，代码与解释如下：

```java
private void layoutChildWithAnchor(View child, View anchor, int layoutDirection) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();

    final Rect anchorRect = mTempRect1;
    final Rect childRect = mTempRect2;

    /* 1. 找到被anchor的View的布局边界 */
    getDescendantRect(anchor, anchorRect);

    /* 2. 获取到被anchor的View布局边界之后，配合layout_anchorGravity与自身的gravity获取到最终要layout到的边界 */
    getDesiredAnchoredChildRect(child, layoutDirection, anchorRect, childRect);
    child.layout(childRect.left, childRect.top, childRect.right, childRect.bottom);
}
```

这里用到的两个关键函数就是`getDescendantRect`与`getDesiredAnchoredChildRect`，它们的目的在我添加的注释中进行了解释，为保证文章的可读性就不再把代码放上来了，有兴趣的同学可以再自己去挖掘相应代码~~

#### 2.2.3 直接layout

如果之前的都不符合，就会走到这一步，我们看看它是怎么layout的：

```java
 private void layoutChild(View child, int layoutDirection) {
     final LayoutParams lp = (LayoutParams) child.getLayoutParams();
     final Rect parent = mTempRect1;
     parent.set(getPaddingLeft() + lp.leftMargin,
             getPaddingTop() + lp.topMargin,
             getWidth() - getPaddingRight() - lp.rightMargin,
             getHeight() - getPaddingBottom() - lp.bottomMargin);

     //...(省略代码) 处理由于fitsSystemWindows带来的inset

	 // 按照Gravity与measure尺寸在父控件里面找到自己的位置，并进行layout。
     final Rect out = mTempRect2;
     GravityCompat.apply(resolveGravity(lp.gravity), child.getMeasuredWidth(),
             child.getMeasuredHeight(), parent, out, layoutDirection);
     child.layout(out.left, out.top, out.right, out.bottom);
 }
```

##3 总结

CoordinatorLayout的特性总结下来就是两个方面：

1. 可以设置anchor，被依赖的View变化自身也会变化；
2. 可以设置behavior，当内部有支持嵌套滑动的控件时处理NestedScroll事件；

这两个特性导致的子View之间的依赖关系让界面的交互更有意思。有兴趣的同学可以再去看`AppBarLayout`、`FloatingActionButton`、`SnackBar`的源码~~
