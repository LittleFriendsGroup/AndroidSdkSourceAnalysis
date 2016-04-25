## CompoundButton 源码分析
CompoundButton 是一个有两种状态（选中和未选中 / checkd unchecked）的Button。当你按下（pressed）或者点击（clicked），它的状态会自动改变。



### 特点

- 是一个抽象类（abstract），所以我们不能直接使用它，只有自定义实现或者系统已经提供的它的一直子类（ToggleButton，Checkbox，RadioButton 等等）。
- 继承自Button，而Button 继承自TextView，所以Button，TextView 的特性CompoundButton 都是具备的。
- 实现自Checkable 接口（interface），利用它可以设置状态（setChecked(boolean checked)），获取状态（isChecked()）和切换状态（toggle()）。

最后的效果图就是这样的。<br />
![](http://ww2.sinaimg.cn/large/68622377gw1f36nyso3hsg20c0034dfy.gif)



### 分析

```java
// 选中和未选中的状态
private boolean mChecked;

private boolean mBroadcasting;

private Drawable mButtonDrawable;

private ColorStateList mButtonTintList = null;
// 就是水波纹和背景颜色混合的方式
private PorterDuff.Mode mButtonTintMode = null;

private boolean mHasButtonTint = false;
private boolean mHasButtonTintMode = false;

// 状态监听
private OnCheckedChangeListener mOnCheckedChangeListener;
private OnCheckedChangeListener mOnCheckedChangeWidgetListener;
```
这是一些局部变量，在后面的分析会用到。


我们先来看看CompoundButton 自定义控件有哪些属性
**\data\res\values\attrs.xml**
```xml
 <declare-styleable name="CompoundButton">
      <!-- 设置状态  true: 选中; false: 未选中 -->
      <attr name="checked" format="boolean" />
      <!-- 绘制按钮图形，一般为Drawable 资源 (e.g. checkbox, radio button, etc). -->
      <attr name="button" format="reference" />
      <!-- 对绘制的按钮图形着色 -->
      <attr name="buttonTint" format="color" />
      <!-- 对着色设置模式 -->
      <attr name="buttonTintMode">
          <enum name="src_over" value="3" />
          ...
      </attr>
  </declare-styleable>
```


然后再来看看怎么绘制，先来看看构造方法
```java
public CompoundButton(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context, attrs, defStyleAttr, defStyleRes);

    // 这里的获取自定义的CompoundButton 就是上面的定义的CompoundButton
    final TypedArray a = context.obtainStyledAttributes(
            attrs, com.android.internal.R.styleable.CompoundButton, defStyleAttr, defStyleRes);

    // 用于绘制按钮图形
    final Drawable d = a.getDrawable(com.android.internal.R.styleable.CompoundButton_button);
    if (d != null) {
        setButtonDrawable(d);
    }

    // 对绘制的按钮图形着色设置模式
    if (a.hasValue(R.styleable.CompoundButton_buttonTintMode)) {
        mButtonTintMode = Drawable.parseTintMode(a.getInt(
                R.styleable.CompoundButton_buttonTintMode, -1), mButtonTintMode);
        mHasButtonTintMode = true;
    }

    // 对绘制的按钮图形着色
    if (a.hasValue(R.styleable.CompoundButton_buttonTint)) {
        mButtonTintList = a.getColorStateList(R.styleable.CompoundButton_buttonTint);
        mHasButtonTint = true;
    }

    // 设置状态
    final boolean checked = a.getBoolean(
            com.android.internal.R.styleable.CompoundButton_checked, false);
    setChecked(checked);

    a.re cycle();

    applyButtonTint();
}
```

在构造方法中，获取自定义属性的各个属性，

- button：用于绘制按钮图形，然后调用setButtonDrawable() 来绘制。
- buttonTint：绘制的按钮着色。使用一个boolean 标识符来设置的，然后会在applyButtonTint() 中统一处理。它们两个分别用作给和
- buttonTintMode：和设置着色模式。设置方式和buttonTint 几乎一样。不过它的一些属性，参考这篇文章来看看具体不同的着色模式效果是怎么样的。[android5.x新特性之Tinting](http://www.cnblogs.com/fuly550871915/p/5002759.html)
- checked：是设置选中状态，在setChecked() 中设置。




### 绘制按钮图形

```java
@Nullable
public void setButtonDrawable(@Nullable Drawable drawable) {
    if (mButtonDrawable != drawable) {
        if (mButtonDrawable != null) {
            // 取消对View 的引用
            mButtonDrawable.setCallback(null);
            // 取消绘制对象相关联的调度,当我们重新绘制一个Drawable，可以调用此方法
            unscheduleDrawable(mButtonDrawable);
        }

        mButtonDrawable = drawable;

        if (drawable != null) {
            drawable.setCallback(this);
            drawable.setLayoutDirection(getLayoutDirection());
            if (drawable.isStateful()) {
                drawable.setState(getDrawableState());
            }
            drawable.setVisible(getVisibility() == VISIBLE, false);
            setMinHeight(drawable.getIntrinsicHeight());
            applyButtonTint();
        }
    }
}
```
这个方法，就是用于绘制按钮图形，首先会判断我们设置的Drawable 和初始会的mButtonDrawable 是否相等，如果不等，将会对初始化的mButtonDrawable 对象，取消对View 的引用（setCallback(null)），并且取消mButtonDrawable 相关联调度（unscheduleDrawable()），然后将我们新设置的Drawable 赋值给mButtonDrawable 对象，然后再设置引用，设置布局的方向，然后判断如果状态改变时，重新设置状态。最后调用applyButtonTint()。



### 着色

```java
private void applyButtonTint() {
    if (mButtonDrawable != null && (mHasButtonTint || mHasButtonTintMode)) {
        mButtonDrawable = mButtonDrawable.mutate();

        if (mHasButtonTint) {
            mButtonDrawable.setTintList(mButtonTintList);
        }

        if (mHasButtonTintMode) {
            mButtonDrawable.setTintMode(mButtonTintMode);
        }

        // The drawable (or one of its children) may not have been
        // stateful before applying the tint, so let's try again.
        if (mButtonDrawable.isStateful()) {
            mButtonDrawable.setState(getDrawableState());
        }
    }
}
```
这个方法主要就是设置mButtonDrawable 的Tint（着色）和TintMode（着色模式），之前在构造方法，setButtonDrawable() 方法中都会调用此方法，因为这两个属性都是基于mButtonDrawable 来设置的，而这个两个属性是根据两个Boolean 属性mHasButtonTint 和mHasButtonTintMode 来识别的，然后为true，就表示设置。而他们两个属性也有setButtonTintList 和setButtonTintMode() 方法来设置两个属性，将两个boolean 属性设置为true，并且调用applyButtonTint() 来设置的。



### **绘制**

```java
protected void onDraw(Canvas canvas) {
    final Drawable buttonDrawable = mButtonDrawable;
    if (buttonDrawable != null) {
        final int verticalGravity = getGravity() & Gravity.VERTICAL_GRAVITY_MASK;
        final int drawableHeight = buttonDrawable.getIntrinsicHeight();
        final int drawableWidth = buttonDrawable.getIntrinsicWidth();

        final int top;
        switch (verticalGravity) {
            case Gravity.BOTTOM:
                top = getHeight() - drawableHeight;
                break;
            case Gravity.CENTER_VERTICAL:
                top = (getHeight() - drawableHeight) / 2;
                break;
            default:
                top = 0;
        }
        final int bottom = top + drawableHeight;
        final int left = isLayoutRtl() ? getWidth() - drawableWidth : 0;
        final int right = isLayoutRtl() ? getWidth() : drawableWidth;

        buttonDrawable.setBounds(left, top, right, bottom);

        final Drawable background = getBackground();
        if (background != null) {
            background.setHotspotBounds(left, top, right, bottom);
        }
    }

    super.onDraw(canvas);

    if (buttonDrawable != null) {
        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        if (scrollX == 0 && scrollY == 0) {
            buttonDrawable.draw(canvas);
        } else {
            canvas.translate(scrollX, scrollY);
            buttonDrawable.draw(canvas);
            canvas.translate(-scrollX, -scrollY);
        }
    }
}
```
这些属性都初始化好了，那就可以来绘制了，我们都知道自定义重写onDraw() 方法来绘制视图，CompoundButton 也重写了此方法，将我们设置了各种属性的mButtonDrawable 复制给局部变量buttonDrawable，然后根据对其方式（Gravity） 属性来来具体绘制buttonDrawable。然后调用父类的onDraw()，最后在根据时候是滑动通过Canvas 来绘制，如果水平和垂直滑动为0，则直接绘制即可，如果不为零则需要调用translate 对canvas 的重新绘制。



### **设置选中(checked)状态**

```java
public void setChecked(boolean checked) {
    if (mChecked != checked) {
        mChecked = checked;
        refreshDrawableState();
        notifyViewAccessibilityStateChangedIfNeeded(
                AccessibilityEvent.CONTENT_CHANGE_TYPE_UNDEFINED);

        // 避免多次调用setChecked() 来多次调用回调监听
        if (mBroadcasting) {
            return;
        }

        mBroadcasting = true;
        if (mOnCheckedChangeListener != null) {
            mOnCheckedChangeListener.onCheckedChanged(this, mChecked);
        }
        if (mOnCheckedChangeWidgetListener != null) {
            mOnCheckedChangeWidgetListener.onCheckedChanged(this, mChecked);
        }

        mBroadcasting = false;            
    }
}


// 设置监状态听
public void setOnCheckedChangeListener(OnCheckedChangeListener listener) {
    mOnCheckedChangeListener = listener;
}

void setOnCheckedChangeWidgetListener(OnCheckedChangeListener listener) {
    mOnCheckedChangeWidgetListener = listener;
}

```
设置状态，就是传入一个Boolean 值来设置状态，如果和初始化的mChecked 相反，才会调用，然后调用refreshDrawableState() 来刷新绘制的状态，然后下面就是设置状态改变的监听，通过mBroadcasting 属性来避免多次设置回调，每次调用mBroadcasting，如果为true，则返回。发现有两个监听，我们仔细看下面setXxxListener() 的方法，而下面那个setOnCheckedChangeWidgetListener() 方法不是Public，所以说不是对外开放的，我们是不能调用的，文档中说明是仅供内部使用。因此我们想要监听选中状态，可以使用setOnCheckedChangeListener()。重写onCheckedChanged(CompoundButton buttonView, boolean isChecked) 即可。对了，除了setChecked() 可以设置状态，toggle() 每次也会setChecked() 方法，每次都会讲状态设置为相反的。还有isChecked() 方法，判断当前的选中状态。



### **状态保存**

```java
static class SavedState extends BaseSavedState {
    boolean checked;
    SavedState(Parcelable superState) {
        super(superState);
    }

    private SavedState(Parcel in) {
        super(in);
        checked = (Boolean)in.readValue(null);
    }

    @Override
    public void writeToParcel(Parcel out, int flags) {
        super.writeToParcel(out, flags);
        out.writeValue(checked);
    }

    public static final Parcelable.Creator<SavedState> CREATOR
            = new Parcelable.Creator<SavedState>() {
        public SavedState createFromParcel(Parcel in) {
            return new SavedState(in);
        }

        public SavedState[] newArray(int size) {
            return new SavedState[size];
        }
    };
}

@Override
public Parcelable onSaveInstanceState() {
    Parcelable superState = super.onSaveInstanceState();
    SavedState ss = new SavedState(superState);
    ss.checked = isChecked();
    return ss;
}

@Override
public void onRestoreInstanceState(Parcelable state) {
    SavedState ss = (SavedState) state;

    super.onRestoreInstanceState(ss.getSuperState());
    setChecked(ss.checked);
    requestLayout();
}
```
保存状态是自定义一个SavedState，继承自BaseSavedState，然后Parcelable 将Boobean 类型checked 属性序列化，判断是否选中，在onSaveInstanceState() 中，保存，然后在onRestoreInstanceState() 获取序列化的属性，重新调用setChecked() 设置属性。




## Checkbox/ToggleButton
Checkbox 和ToggleButton 的实现那都是继承自CompoundButton，可以看下面这两篇文章。
- [ToggleButton/Checkbox 的源码分析](ToggleButton和CheckBox源码分析)
