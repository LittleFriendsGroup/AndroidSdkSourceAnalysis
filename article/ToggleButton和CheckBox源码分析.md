## ToggleButton

ToggleButton 是一种具有指示灯的开/关按钮。因为它继承自CompoundButton，所以CompoundButton 的属性它也都可以使用，不懂CompoundButton 可以看[这篇文章](CompoundButton源码分析.md)。它的底部有个类似似分割线的东西。像下面这张图。
![http://ww4.sinaimg.cn/large/68622377gw1f37nsm4mhsg20f0048ta0.gif?width=100](http://ww4.sinaimg.cn/large/68622377gw1f37nsm4mhsg20f0048ta0.gif)



### **属性**

| [android:disabledAlpha](http://developer.android.com/reference/android/widget/ToggleButton.html#attr_android:disabledAlpha) | 设置按钮在禁用时透明度（我们后面会详细说到）（enable=false） |
| ---------------------------------------- | ------------------------------------ |
| [android:textOff](http://developer.android.com/reference/android/widget/ToggleButton.html#attr_android:textOff) | 没有选中的文字                              |
| [android:textOn](http://developer.android.com/reference/android/widget/ToggleButton.html#attr_android:textOn) | 选中的文字                                |



### 分析

```java
public ToggleButton(Context context, AttributeSet attrs) {
    this(context, attrs, com.android.internal.R.attr.buttonStyleToggle);
}
```



**com.android.internal.R.attr.buttonStyleToggle**

\res\values\theme_material.xml（不同的版本会不一样）

```xml
...
<item name="buttonStyleToggle">@style/Widget.Material.Button.Toggle</item>
...
```



**\data\res\values\styles_material.xml**
```xml
<style name="Widget.Material.Button.Toggle">
    <item name="background">@drawable/btn_toggle_material</item>
    <item name="textOn">@string/capital_on</item>
    <item name="textOff">@string/capital_off</item>
</style>
```



**data\res\drawable\btn_toggle_material.xml**

```xml
<inset xmlns:android="http://schemas.android.com/apk/res/android"
       android:insetLeft="@dimen/button_inset_horizontal_material"
       android:insetTop="@dimen/button_inset_vertical_material"
       android:insetRight="@dimen/button_inset_horizontal_material"
       android:insetBottom="@dimen/button_inset_vertical_material">
    <layer-list android:paddingMode="stack">
        <item>
            <ripple android:color="?attr/colorControlHighlight">
                <item>
                    <shape android:shape="rectangle"
                           android:tint="?attr/colorButtonNormal">
                        <corners android:topLeftRadius="@dimen/control_corner_material"
                                 android:topRightRadius="@dimen/control_corner_material"/>
                        <solid android:color="@color/white" />
                        <padding android:left="@dimen/button_padding_horizontal_material"
                                 android:top="@dimen/button_padding_vertical_material"
                                 android:right="@dimen/button_padding_horizontal_material"
                                 android:bottom="@dimen/button_padding_vertical_material" />
                    </shape>
                </item>
            </ripple>
        </item>
        <item android:gravity="bottom|fill_horizontal">
            <shape android:shape="rectangle">
                <size android:height="2dp" />
                <solid android:color="@color/control_checkable_material" />
            </shape>
        </item>
    </layer-list>
</inset>
```

仔细发现会有一个<layer-list>（可以多个图层），有两个<Item>，第一个<item> 设置ripple 具有涟漪效果，设置Drawable，着色，还有其他属性，第二个<Item> 就是下面的底部指示器，然后会根据checked 不同的属性来设置颜色。



**@color/control_checkable_material**

```xml
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_enabled="false"
          android:alpha="?attr/disabledAlpha"
          android:color="?attr/colorControlNormal" />
    <item android:state_checked="true"
          android:color="?attr/colorControlActivated" />
    <item android:color="?attr/colorControlNormal" />
</selector>
```

而最终两个颜色值是colorControlNormal 和colorControlActivated，如果你不自定义设置，会使用默认的，如果你想自定义样式，需要在Theme 中设置这两个颜色的值。



### **设置样式**

```java
@Override
public void setChecked(boolean checked) {
    super.setChecked(checked);
    syncTextState();
}

private void syncTextState() {
    boolean checked = isChecked();
    if (checked && mTextOn != null) {
        setText(mTextOn);
    } else if (!checked && mTextOff != null) {
        setText(mTextOff);
    }
}
```

因为状态监听已经在父类里面实现了，所以在setChecked() 中，只需重写自己新的属性，会根据选中状态。来设置文字。



### **设置指示器**

```java
// 设置背景资源
@Override
public void setBackgroundDrawable(Drawable d) {
    super.setBackgroundDrawable(d);
    updateReferenceToIndicatorDrawable(d);
}

// 根据Drawable 来获取LayerDrawable，进而通过id toggle 来获取指示器Drawable。
private void updateReferenceToIndicatorDrawable(Drawable backgroundDrawable) {
    if (backgroundDrawable instanceof LayerDrawable) {
        LayerDrawable layerDrawable = (LayerDrawable) backgroundDrawable;
        mIndicatorDrawable =
                layerDrawable.findDrawableByLayerId(com.android.internal.R.id.toggle);
    } else {
        mIndicatorDrawable = null;
    }
}

// Drawable 状态改变时，根据disabledAlpha 来设置指示器颜色的透明度。NO_ALPHA 默认为0XFF（255）。
@Override
protected void drawableStateChanged() {
    super.drawableStateChanged();

    if (mIndicatorDrawable != null) {
        mIndicatorDrawable.setAlpha(isEnabled() ? NO_ALPHA : (int) (NO_ALPHA * mDisabledAlpha));
    }
}
```

除了设置上面的文字，还有背景（包括下面的指示器）。

我们调用setBackgroundDrawable() 来设置背景资源，然后调用updateReferenceToIndicatorDrawable() 通过id toggle 来获取指示器Drawable。最后在drawableStateChanged() 中根据disabledAlpha 来设置指示器颜色的透明度。



### **问题**

在使用的过程中，设置disabledAlpha 属性，并且enble="false"，发现指示器的颜色的透明度并没有改变，后来在调试中发现mIndicatorDrawable 一直为空，所以想要使用这个属性，必须自定义背景资源，然后调用setBackgroundDrawable() 来设置，并且将下面指示器的Drawable id 设置为R.id.toggle，才能起作用。详细的Demo 可以去[我的Github 中查看]()，最后的效果图就是这样的效果。
![http://ww1.sinaimg.cn/large/68622377gw1f37ntueifsj20f10bv0t0.jpg?width=100](http://ww1.sinaimg.cn/large/68622377gw1f37ntueifsj20f10bv0t0.jpg)



## RadioButton/CheckBox

**CheckBox**

Checkbox 是复选框，可由切换的启用/禁用开关。

```java
public CheckBox(Context context, AttributeSet attrs) {
    this(context, attrs, com.android.internal.R.attr.checkboxStyle);
}
```

CheckBox 继承自CompoundButton，就重写了这个方法，自定义了样式（Style），其他都没有改变。



com.android.internal.R.attr.checkboxStyle

```xml
style name="Widget.Material.CompoundButton.CheckBox" parent="Widget.CompoundButton.CheckBox">
    <item name="background">@drawable/control_background_40dp_material</item>
</style>
```



@drawable/control_background_40dp_material

```xml
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
        android:color="@color/control_highlight_material"
        android:radius="20dp" />
```



@color/control_highlight_material

```xml
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_checked="true"
          android:state_enabled="true"
          android:alpha="@dimen/highlight_alpha_material_colored"
          android:color="?attr/colorControlActivated" />
    <item android:color="?attr/colorControlHighlight" />
</selector>
```

首先我们看style 中的background 属性，是给Checkbox 着色的。最后会考到最终的颜色是colorControlActivated 和colorControlHighlight。



```xml
<style name="Widget.CompoundButton.CheckBox">
    <item name="button">?attr/listChoiceIndicatorMultiple</item>
</style>
```



?attr/listChoiceIndicatorMultiple

```xml
<item name="listChoiceIndicatorMultiple">@drawable/btn_check_material_anim</item>
```



@drawable/btn_check_material_anim

```xml
<animated-selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item
        android:id="@+id/checked"
        android:state_checked="true"
        android:drawable="@drawable/ic_checkbox_checked" />
    <item
        android:id="@+id/unchecked"
        android:drawable="@drawable/ic_checkbox_unchecked" />
    <transition
        android:fromId="@+id/unchecked"
        android:toId="@+id/checked"
        android:drawable="@drawable/ic_checkbox_unchecked_to_checked_animation" />
    <transition
        android:fromId="@+id/checked"
        android:toId="@+id/unchecked"
        android:drawable="@drawable/ic_checkbox_checked_to_unchecked_animation" />
</animated-selector>
```

我们可以看到两个Item，分别是选中和未选中的，每个Item 是由Vector来画出的，并且使用@color/control_checkable_material（上面提到过） 来着色的的。状态相互切换，是由两个动画来实现的。
