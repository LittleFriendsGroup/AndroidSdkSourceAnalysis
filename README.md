ANDROID SDK 源码解析
===============================

![](https://github.com/yuxingxin/AndroidWidgetClassGraph/blob/master/img/android.jpg)

#### GitHub小伙伴公众号，欢迎扫码关注！

<img src="https://raw.githubusercontent.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/master/images/qrcode.jpg" width="156" height="156">

-----

##### 概要说明：

* 已发布文章 发表已经整理好的文章，读者可以阅读学习！

* 认领方式 可以在 issues 提你要认领什么内容。

~~* 已认领文章 如果你喜欢的文章被认领，你想参与,你还是可以分析认领，我们选择好的发布，也可以作为校对者。认领方式：可以在 Issues 提你要认领什么内容~~

~~* 待认领文章 是想参与的的同学可以参与进来，如被认领，也可以做校对者，若想解析的内容不在表格，可以联系我们添加分析的内容，方式：在 Issues 提你要认领什么内容~~

##### 校对发布说明：
分析完成后可直接在对应 issue 下回复，可直接原文回复也可是原文链接，校对通过后会直接进行发布。（这样大家可以更灵活自由的安排，同时也可以更快的发布校对好的文章）

##### 转载说明：
这里每一篇文章我们都或多或少的付出了时间、精力分析校对，第一次搞这种源码解析，可能有很多地方做的不好，但是我们用心做了！所以，如果你想转载，至少文章开头写下来源地址：

[https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis)

,还有写下分析者名字！请尊重每一篇文章的劳动成果，谢谢！

## 已发布文章

### 第三期
Class | 分析者 | 校对者 | 版本 | 发布时间
:------------- | :------------- | :------------- | :------------- | :-------------
[ViewGroup 源码解析](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/blob/master/article/ViewGroup%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [7heaven](https://github.com/7heaven) | [Nukc](https://github.com/nukc) | branch nougat-mr2-release | 2017/4/17
[StaticLayout 源码解析](http://jaeger.itscoder.com/android/2016/08/05/staticlayout-source-analyse.html) | [laobie](https://github.com/laobie) | [7heaven](https://github.com/7heaven) | android api 23 | 2017/4/17
[AtomicFile 源码解析](https://github.com/GcsSloop/AndroidNote/blob/master/SourceAnalysis/AtomicFile.md) | [GcsSloop](https://github.com/GcsSloop) | [Nukc](https://github.com/nukc) | android api 25 | 2017/4/17
[Spannable 源码解析](https://github.com/lber19535/SourceAnalysis/blob/master/Spannable%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) | [lber19535](https://github.com/lber19535) | [Nukc](https://github.com/nukc) | android api 24 | 2017/4/17
[Notification 源码解析](http://www.jianshu.com/p/0cb97db7090c) | [huanglongyu](https://github.com/huanglongyu) | [Nukc](https://github.com/nukc) | android api 21 (cm) | 2017/4/17
[SparseArray 源码解析](http://sonaive.me/2016/05/04/sparse-array-analysis/) | [taoliuh](https://github.com/taoliuh) | [Nukc](https://github.com/nukc) | android api 22 | 2017/4/17
[ViewStub 源码解析](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/blob/master/article/ViewStub%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [Nukc](https://github.com/nukc) | [7heaven](https://github.com/7heaven) | android api 25 | 2017/4/17

### 第二期
Class | 分析者 | 校对者 | 版本 | 发布时间
:------------- | :------------- | :------------- | :------------- | :-------------
[MediaPlayer源码解析](https://github.com/lber19535/SourceAnalysis/blob/master/Media%20Player%20%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) | [lber19535](https://github.com/lber19535) | [android-cjj](https://github.com/android-cjj) | android api 22 | 2016/7/25
[NavigationView源码解析](https://github.com/hongyangAndroid/AndroidSdkSourceAnalysis/blob/master/article/NavigationView%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [hongyangAndroid](https://github.com/hongyangAndroid) | [android-cjj](https://github.com/android-cjj) | support-v7-23.1.0 | 2016/7/25
[Service源码解析](https://github.com/asLody/SourceAnalysis/blob/master/Service%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [asLody](https://github.com/asLody) | [liaohuqiu](https://github.com/liaohuqiu) | android api 23 | 2016/7/25
[SharePreferences源码解析](http://blog.csdn.net/yanbober/article/details/47866369) | [yanbober](https://github.com/yanbober) | [android-cjj](https://github.com/android-cjj) | android api 22 | 2016/7/25
[ScrollView源码分析](https://github.com/Skykai521/AndroidSdkSourceAnalysis/blob/master/article/ScrollView%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) | [Skykai521](https://github.com/Skykai521) | [android-cjj](https://github.com/android-cjj) | android api 23 | 2016/7/25
[Handler源码解析](https://github.com/maoruibin/HandlerAnalysis) | [maoruibin](https://github.com/maoruibin) | [android-cjj](https://github.com/android-cjj) | android api 23 | 2016/7/25
[NestedScrollView源码解析](https://github.com/xmuSistone/android-source-analysis/blob/master/NestedScrollView.md) | [xmuSistone](https://github.com/xmuSistone) | [android-cjj](https://github.com/android-cjj) | support-v4-23.1.0 | 2016/7/25
[SQLiteOpenHelper/...源码解析](https://github.com/YZHIWEN/AndroidSdkSourceAnalysis/blob/master/SQLite_Android.md) | [YZHIWEN](https://github.com/YZHIWEN) | [CaMnter](https://github.com/CaMnter) | android api 23 | 2016/7/25
[Bundle源码解析](https://github.com/ASPOOK/BundleAnalysis) | [ASPOOK](https://github.com/ASPOOK) | [CaMnter](https://github.com/CaMnter) | android api 23 | 2016/7/25
[LocalBroadcastManager源码解析](https://github.com/czhzero/AndroidSdkSourceAnalysis/blob/master/article/LocalBroadcastManager%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [czhzero](https://github.com/czhzero) | [CaMnter](https://github.com/CaMnter) | support-v4-23.4.0 | 2016/7/25
[Toast源码解析](https://github.com/WuXiaolong/AndroidSdkSourceAnalysis/blob/master/article/Toast%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [Wuxiaolong](https://github.com/WuXiaolong) | [Nukc](https://github.com/nukc) | android api 23 | 2016/7/25
[TextInputLayout源码解析](https://github.com/wbersaty/TextInputLayout-24) | [wbersaty](https://github.com/wbersaty) | [android-cjj](https://github.com/android-cjj) | design-24.0.0-alpha2 | 2016/7/25
[LayoutInflater...源码解析](https://github.com/peerless2012/SourceAnalysis/blob/master/Android/FrameWork/LayoutInflater%26LayoutInflaterCompat%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [peerless2012](https://github.com/peerless2012) | [android-cjj](https://github.com/android-cjj) | android api 23 | 2016/7/25
[NestedScrolling事件机制源码解析](http://www.jianshu.com/p/6547ec3202bd) | [android-cjj](https://github.com/android-cjj) | [android-cjj](https://github.com/android-cjj/) | design-24.0.0 | 2016/7/25

### 第一期

Class | 分析者 | 校对者 | 版本 | 发布时间
:------------- | :------------- | :------------- | :------------- | :-------------
[Binder源码解析](https://github.com/xdtianyu/SourceAnalysis/blob/master/Binder%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) | [xdtianyu](https://github.com/xdtianyu) |[xdtianyu](https://github.com/xdtianyu) |android api 23| 2016/5/8
[TextView源码解析](https://github.com/7heaven/AndroidSdkSourceAnalysis/blob/master/article/textview%E6%BA%90%E7%A2%BC%E8%A7%A3%E6%9E%90.md) | [7heaven](https://github.com/7heaven) |[7heaven](https://github.com/7heaven) |android api 23| 2016/5/8
[CoordinatorLayout源码解析](https://github.com/desmond1121/AndroidSdkSourceAnalysis/blob/master/article/CoordinatorLayout%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [Desmond Yao](https://github.com/desmond1121) |[轻微](https://github.com/zzz40500) | support-v7-23.2.1| 2016/5/8
[Scroller源码解析](https://github.com/Skykai521/AndroidSdkSourceAnalysis/blob/master/article/Scroller%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) | [Skykai521](https://github.com/Skykai521) |[子墨](https://github.com/zimoguo) | android api 22 | 2016/5/8
[SwipeRefreshLayout 源码解析](https://github.com/hanks-zyh/SwipeRefreshLayout/blob/master/README.md) | [hanks-zyh](https://github.com/hanks-zyh) |[android-cjj](https://github.com/android-cjj/) | support-v7-23.2.1| 2016/5/8
[FloatingActionButton源码解析](https://github.com/Rowandjj/my_awesome_blog/blob/master/fab_anlysis/README.md) | [Rowandjj](https://github.com/Rowandjj) |[CaMnter](https://github.com/CaMnter) | support-v7-23.2.1| 2016/5/8
[AsyncTask源码解析](https://github.com/white37/AndroidSdkSourceAnalysis/blob/master/article/AsyncTask%E5%92%8CAsyncTaskCompat%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [white37](https://github.com/white37) |[android-cjj](https://github.com/android-cjj/) | android api 23 | 2016/5/8
[TabLayout源码解析](https://github.com/Aspsine/AndroidSdkSourceAnalysis/blob/master/article/TabLayout%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [Aspsine](https://github.com/Aspsine) |[android-cjj](https://github.com/android-cjj/) | design-23.2.0 | 2016/5/8
[CompoundButton源码解析](https://github.com/Tikitoo/AndroidSdkSourceAnalysis/blob/master/article/CompoundButton%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) | [Tikitoo](https://github.com/Tikitoo) |[android-cjj](https://github.com/android-cjj/) | android api 23 | 2016/5/8
[LinearLayout源码解析](https://github.com/razerdp/AndroidSourceAnalysis/blob/master/LinearLayout/android_widget_LinearLayout.md) | [razerdp](https://github.com/razerdp) |[android-cjj](https://github.com/android-cjj/) | support-v7-23.2.1 | 2016/5/8
[SearchView源码解析](https://github.com/nukc/SearchViewAnalysis/blob/master/README.md) | [Nukc](https://github.com/nukc) |[android-cjj](https://github.com/android-cjj/) | support-v7-23.2.1 | 2016/5/7
[LruCache源码解析](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/blob/master/article/LruCache%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90.md) | [CaMnter](https://github.com/CaMnter)| [alafighting](https://github.com/alafighting)|support-v4-23.2.1 | 2016/4/24
[ViewDragHelper源码解析](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/blob/master/article/ViewDragHelper%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md) | [达庆凯](https://github.com/Skykai521)| [android-cjj](https://github.com/android-cjj/)|support-v4-21.0 | 2016/4/21   
[BottomSheets源码解析](https://github.com/android-cjj/SourceAnalysis) | [android-cjj](https://github.com/android-cjj/)| [轻微](https://github.com/zzz40500)|design-23.2.0 | 2016/4/20  



## 已认领文章
(写好的童鞋可以加 QQ 群:369144556做校对)
<table>
  <thead>
    <tr>
      <th>Class</th>
      <th>认领者</th>
    </tr>
  </thead>
  <tbody>
     <tr>
      <td>Seekbar源码解析</td>
      <td>JohnTsaiAndroid</td>
    </tr>
    <tr>
      <td>ArrayMap源码解析</td>
      <td>audiebantzhan</td>
    </tr>
    <tr>
      <td>SimpleArrayMap源码解析</td>
      <td>david-wei</td>
    </tr>
    <tr>
      <td>ViewPager源码解析</td>
      <td>cpoopc</td>
    </tr>
     <tr>
      <td>LongSparseArray源码解析</td>
      <td>taoliuh</td>
    </tr>
    <tr>
      <td>Dialog源码解析</td>
      <td>wingjay</td>
    </tr>
    <tr>
      <td>Frame/RelativeLayout源码解析</td>
      <td>wingjay</td>
    </tr>
    <tr>
      <td>Drawable源码解析</td>
      <td>wingjay</td>
    </tr>
     <tr>
    	<td>AppBarLayout源码解析</td>
    	<td>desmond1121</td>
    </tr>
     <tr>
    	<td>ProgressBar源码解析</td>
    	<td>carozhu</td>
    </tr>
    <tr>
    	<td>GestureDetector源码分析</td>
    	<td>lishiwei</td>
    </tr>
    <tr>
    	<td>RecyclerView/ItemTouchHelper源码解析</td>
    	<td>xdtianyu</td>
    </tr>
    <tr>
    	<td>Toolbar源码解析</td>
    	<td>SeniorZhai</td>
    </tr>
     <tr>
    	<td>WebView源码解析</td>
    	<td>markzhai</td>
    </tr>
     <tr>
    	<td>Bitmap源码解析</td>
    	<td>zimoguo</td>
    </tr>
     <tr>
    	<td>AdapterView源码解析</td>
    	<td>ShenghuGong</td>
    </tr>
    <tr>
    	<td>Activity源码解析</td>
    	<td>nekocode</td>
    </tr>
     <tr>
    	<td>Camera源码解析</td>
    	<td>gcgongchao</td>
    </tr>
    <tr>
    	<td>Volley源码解析</td>
    	<td>THEONE10211024</td>
    </tr>
    <tr>
    	<td>AudioPlayer源码解析</td>
    	<td>ayyb1988</td>
    </tr>
    <tr>
    	<td>TimePicker源码解析</td>
    	<td>shixinzhang</td>
    </tr>
    <tr>
    	<td>Log源码解析</td>
    	<td>lypeer</td>
    </tr>
    <tr>
    	<td>Button源码解析</td>
    	<td>pc859107393</td>
    </tr>
     <tr>
    	<td>Animation源码解析</td>
    	<td>binIoter</td>
    </tr>
    <tr>
    	<td>Parcelable源码解析</td>
    	<td>neuyu</td>
    </tr>
    <tr>
    	<td>BroadcastReceiver源码解析</td>
    	<td>tiefeng0606</td>
    </tr>
    <tr>
    	<td>ImageView源码解析</td>
    	<td>976014121</td>
    </tr>
    <tr>
    	<td>ListView源码解析</td>
    	<td>KingJA</td>
    </tr>
    <tr>
    	<td>Intent源码解析</td>
    	<td>imdreamrunner</td>
    </tr>
     <tr>
    	<td>FragmentTabHost源码分析</td>
    	<td>Tikitoo</td>
    </tr>
    <tr>
    	<td>Canvas源码解析</td>
    	<td>heavenxue</td>
    </tr>
    <tr>
    	<td>PopupWindow源码解析</td>
    	<td>GJson</td>
    </tr>
    <tr>
    	<td>AudioRecord源码解析</td>
    	<td>GJson</td>
    </tr>
    <tr>
    	<td>OverScroller源码解析</td>
    	<td>lizardmia</td>
    </tr>
     <tr>
    	<td>Context源码解析</td>
    	<td>messishow</td>
    </tr>
    <tr>
    	<td>Actionbar/AlertController源码解析</td>
    	<td>rickdynasty</td>
    </tr>
    <tr>
      <td>SnackBar源码解析</td>
      <td>cnLGMing</td>
    </tr>
    <tr>
      <td>LauncherActivity源码解析</td>
      <td>kaiyangjia</td>
    </tr>
    <tr>
      <td>Html源码解析</td>
      <td>DennyCai</td>
    </tr>
    <tr>
      <td>EditText源码解析</td>
      <td>johnwatsondev</td>
    </tr>
    <tr>
      <td>TextureView源码解析</td>
      <td>BeEagle</td>
    </tr>
     <tr>
        <td>DownloadManager源码解析</td>
        <td>xiaohongmaosimida</td>
    </tr>
    <tr>
     <td>ImageButton源码解析</td>
      <td>chenbinzhou</td>
    </tr>
      <tr>
      <td>PopupMenu源码解析</td>
      <td>jimmyguo</td>
    </tr>
    <tr>
     <td>AlarmManager源码解析</td>
     <td>huanglongyu</td>
    </tr>
     <tr>
     <td>Glide源码解析</td>
     <td>Krbit</td>
    </tr>
    <tr>
     <td>DataBinding源码解析</td>
     <td>xdsjs</td>
    </tr>
     <tr>
     <td>PreferenceActivity源码解析</td>
      <td>FightingLarry</td>
    </tr>
    </tbody>
</table>

## 待认领文章

**Sdk**

<table>
  <thead>
    <tr>
      <th>Class</th>
      <th>认领状态</th>
    </tr>
  </thead>
  <tbody>
  <tr>
     <td>ActionBar源码解析</td>
     <td>未认领</td>
    </tr>
   <tr>
     <td>AccountManager源码解析</td>
     <td>未认领</td>
    </tr>
    <tr>
     <td>BluetoothSocket源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
     <td>BoringLayout源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
     <td>DynamicLayout源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
     <td>Paint源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
     <td>Selector原理(Drawable源码解析)</td>
      <td>未认领</td>
    </tr>
    <tr>
     <td>Spinner源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
     <td>TabHost源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
     <td>TableLayout源码解析</td>
      <td>未认领</td>
    </tr>
  </tbody>
</table>

**v4**

<table>
  <thead>
    <tr>
      <th>Class</th>
      <th>认领状态</th>
    </tr>
  </thead>
  <tbody>
    <tr>
    <td>CircularArray源码解析</td>
     <td>未认领</td>
    </tr>
    <tr>
    <td>CircularIntArray源码解析</td>
     <td>未认领</td>
    </tr>
    <tr>
    <td>MapCollections源码解析</td>
    <td>未认领</td>
    </tr>
  </tbody>
</table>

**v7**
<table>
  <thead>
    <tr>
      <th>Class</th>
      <th>认领状态</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>ActionMenuView源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
      <td>ActionBarDrawerToggle源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
      <td>ButtonBarLayout源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
      <td>DrawerArrowDrawable源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
      <td>ListMenuItemView源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
      <td>ActionMenuView源码解析</td>
      <td>未认领</td>
    </tr>
    <tr>
      <td>WindowDecorActionBar源码解析</td>
      <td>未认领</td>
    </tr>
    </tbody>
</table>

**design**

<table>
  <thead>
    <tr>
      <th>Class</th>
      <th>认领状态</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>CollapsingToolbarLayout源码解析</td>
      <td>未认领</td>
    </tr>
   </tbody>
</table>

### 联系方式：
源码解析群 369144556

------

## 许可协议

- [署名-非商业性使用-相同方式共享 4.0 国际](https://creativecommons.org/licenses/by-nc-sa/4.0/legalcode)

------
