# ViewStub

ViewStub 是一个看不见的，没有大小，不占布局位置的 View，可以用来懒加载布局。当 ViewStub 变得可见或 ```inflate()``` 的时候，布局就会被加载（替换 ViewStub）。因此，ViewStub 一直存在于视图层次结构中直到调用了 ```setVisibility(int)``` 或 ```inflate()```。

我们先来看看构造方法：

```java
    public ViewStub(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context);

        final TypedArray a = context.obtainStyledAttributes(attrs,
                R.styleable.ViewStub, defStyleAttr, defStyleRes);
        // 要被加载的布局 Id
        mInflatedId = a.getResourceId(R.styleable.ViewStub_inflatedId, NO_ID);
        // 要被加载的布局
        mLayoutResource = a.getResourceId(R.styleable.ViewStub_layout, 0);
        // ViewStub 的 Id
        mID = a.getResourceId(R.styleable.ViewStub_id, NO_ID);
        a.recycle();

        // 初始状态为 GONE
        setVisibility(GONE);
        // 设置为不会绘制
        setWillNotDraw(true);
    }
```

接下来就看看关键的方法，ViewStub 很轻量，代码真的很少，很值得去看看具体实现，花不了多少时间。

```java
    // 复写了 setVisibility(int) 方法
    @Override
    @android.view.RemotableViewMethod
    public void setVisibility(int visibility) {
        // private WeakReference<View> mInflatedViewRef;
        // mInflatedViewRef 是对布局的弱引用
        if (mInflatedViewRef != null) {
            // 如果不为 null,就拿到懒加载的 View
            View view = mInflatedViewRef.get();
            if (view != null) {
                // 然后就直接对 View 进行 setVisibility 操作
                view.setVisibility(visibility);
            } else {
                // 如果为 null，就抛出异常
                throw new IllegalStateException("setVisibility called on un-referenced view");
            }
        } else {
            super.setVisibility(visibility);
            // 之前说过，setVisibility(int) 也可以进行加载布局
            if (visibility == VISIBLE || visibility == INVISIBLE) {
                // 因为在这里调用了 inflate()
                inflate();
            }
        }
    }
```

inflate() 是关键的加载实现

```java
    public View inflate() {
        // 获取父视图
        final ViewParent viewParent = getParent();
        
        if (viewParent != null && viewParent instanceof ViewGroup) {
            // 如果没有指定布局，就会抛出异常
            if (mLayoutResource != 0) {
                // viewParent 需为 ViewGroup
                final ViewGroup parent = (ViewGroup) viewParent;
                final LayoutInflater factory;
                if (mInflater != null) {
                    factory = mInflater;
                } else {
                    // 如果没有指定 LayoutInflater
                    factory = LayoutInflater.from(mContext);
                }
                // 获取布局
                final View view = factory.inflate(mLayoutResource, parent,
                        false);
                // 为 view 设置 Id
                if (mInflatedId != NO_ID) {
                    view.setId(mInflatedId);
                }
                // 计算出 ViewStub 在 parent 中的位置
                final int index = parent.indexOfChild(this);
                // 把 ViewStub 从 parent 中移除
                parent.removeViewInLayout(this);
                
                // 接下来就是把 view 加到 parent 的 index 位置中
                final ViewGroup.LayoutParams layoutParams = getLayoutParams();
                if (layoutParams != null) {
                    // 如果 ViewStub 的 layoutParams 不为空
                    // 就设置给 view
                    parent.addView(view, index, layoutParams);
                } else {
                    parent.addView(view, index);
                }
                
                // mInflatedViewRef 就是在这里对 view 进行了弱引用
                mInflatedViewRef = new WeakReference<View>(view);

                if (mInflateListener != null) {
                    // 回调
                    mInflateListener.onInflate(this, view);
                }

                return view;
            } else {
                throw new IllegalArgumentException("ViewStub must have a valid layoutResource");
            }
        } else {
            throw new IllegalStateException("ViewStub must have a non-null ViewGroup viewParent");
        }
    }
```

好了，主要的都分析完了，知道原理之后就可以自己动手写一个加强版的 ViewStub 了，例如我以前写的一个 [StateView](https://github.com/nukc/StateView)