LruCache 源码解析
==

## 1. 简介

> Lru 是 Least Recently Used 最近最少使用算法。

>曾经，在各大缓存图片的框架没流行的时候。有一种很常用的内存缓存技术：SoftReference 和 WeakReference（软引用和弱引用）。但是走到了 Android 2.3（Level 9）时代，垃圾回收机制更倾向于回收 SoftReference 或 WeakReference 的对象。后来，又来到了 Android3.0，图片缓存在内容中，因为不知道要在是什么时候释放内存，没有策略，没用一种可以预见的场合去将其释放。这就造成了内存溢出。


## 2. 使用方法

**当成一个 Map 用就可以了，只不过实现了 Lru 缓存策略**

使用的时候记住几点即可：
- **1.（必填）**你需要提供一个缓存容量作为构造参数
- **2.（个人认为必填）**  覆写  `sizeOf` 方法 ，自定义设计一条数据放进来的容量计算。
- **2.（选填）** 覆写 `entryRemoved` 方法 ，你可以知道最少使用的缓存被清除时的数据（ evicted, key, oldValue, newVaule ）
- **3.（记住）**LruCache是线程安全的，在内部的 get、put、remove 包括 trimToSize 都是安全的（因为都上锁了）。
- **4.（选填）** 还有就是覆写 `create` 方法 

一般做到 **1、2、3就足够了，4可以无视** 。


以下是 一个 **LruCache 实现 Bitmap 小缓存的案例**, `entryRemoved` 里的自定义逻辑可以无视，这里是我的展示 demo 里的自定义 `entryRemoved` 逻辑(｡>﹏<｡)
```java
private static final float ONE_MIB = 1024 * 1024;
// 7MB
private static final int CACHE_SIZE = (int) (7 * ONE_MIB);
private LruCache<String, Bitmap> bitmapCache;
this.bitmapCache = new LruCache<String, Bitmap>(CACHE_SIZE) {
    protected int sizeOf(String key, Bitmap value) {
        return value.getByteCount();
    }


    /**
     * 1.当被回收或者删掉时调用。该方法当value被回收释放存储空间时被remove调用
     * 或者替换条目值时put调用，默认实现什么都没做。
     * 2.该方法没用同步调用，如果其他线程访问缓存时，该方法也会执行。
     * 3.evicted=true：如果该条目被删除空间 （表示 进行了trimToSize or remove）  evicted=false：put冲突后 或 get里成功create后 导致
     * 4.newValue!=null，那么则被put()或get()调用。
     */
    @Override
    protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
        mEntryRemovedInfoText.setText(
                String.format(Locale.getDefault(), LRU_CACHE_ENTRY_REMOVED_INFO_FORMAT,
                        evicted, key, oldValue != null ? oldValue.hashCode() : "null",
                        newValue != null ? newValue.hashCode() : "null"));
        // 见上述 3.
        if (evicted) {
            // 进行了trimToSize or remove (一般是溢出了 或 key-value被删除了 )
            if (recentList.contains(key)) {
                recentList.remove(key);
                refreshText(mRecentInfoText, LRU_CACHE_RECENT_FORMAT, recentList);
            }
        } else {
            // put冲突后 或 get里成功create 后
            recentList.remove(key);
            refreshText(mRecentInfoText, LRU_CACHE_RECENT_FORMAT, recentList);
        }
        if (cacheList.contains(key)) {
            cacheList.remove(key);
            refreshText(mCacheDataText, LRU_CACHE_CACHE_DATA_FORMAT, cacheList);
        }
    }
};
```

## 3. 效果展示

### 3.1 效果一（验证 Lru，最近没访问的，在溢出时优先被清理）
<img src="http://ww1.sinaimg.cn/large/006lPEc9jw1f36odh8wjdg31401z4u0x.gif" width="320x"/> 
<img src="http://ww4.sinaimg.cn/large/006lPEc9jw1f36p56qcjzj31401z4qa7.jpg" width="320"/>  

**前提：** 设置 LruCache 最大容量为 7MB，把图1、2、3放入了，此时占用容量为：1.87+0.38+2.47=4.47MB。

**执行操作**：
- **1.**然后点 get 图3一共16次（**证明访问次数和Lru没关系，只有访问顺序有关系**）。Recent visit显示了图3
- **2.**先 get 图2，再 get 图1，**制造最近访问顺序为：<1> <2> <3>**
- **3.** put 图4，预算容量需要4.47+2.47=7.19MB。会溢出。
- **4.**溢出了，删除最近没访问的图3。
- **5.**观察 `entryRemoved` 数据 图三被移除了（对照hashcode）

---

### 3.2 效果二（验证 entryRemoved 的 evicted=false，可以验证冲突）
<img src="http://ww3.sinaimg.cn/large/006lPEc9jw1f36oy2uii5g31401z44l6.gif" width="320x"/> <img src="http://ww2.sinaimg.cn/large/006lPEc9jw1f36p6e8t4jj31401z4gt0.jpg" width="320x"/>
  
**前提：**执行了效果一，put 了图4，删除了最近没访问的图3。  

**执行操作**：再一次 put 图4，发生冲突，拿到 key、冲突 value 以及 put 的 value，这里我放到是同一个 hashcode 的 bitmap，所以 hashcode 一样，但是无关紧要吧。

## 4. 源码分析	

LruCache 就是 **利用 LinkedHashMap 的一个特性再加上对 LinkedHashMap 的数据操作上锁实现的缓存策略**。

### 4.1 LruCache 的唯一构造方法
```java
/**
 * LruCache的构造方法：需要传入最大缓存个数
 */
public LruCache(int maxSize) {
    // 最大缓存个数小于0，会抛出IllegalArgumentException
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }
    this.maxSize = maxSize;
    /*
     * 初始化LinkedHashMap
     * 第一个参数：initialCapacity，初始大小
     * 第二个参数：loadFactor，负载因子=0.75f
     * 第三个参数：accessOrder=true，基于访问顺序；accessOrder=false，基于插入顺序
     */
    this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
}
```
第一个参数 `initialCapacity` 用于初始化该 LinkedHashMap 的大小。

先简单介绍一下 第二个参数 `loadFactor`，这个其实的 HashMap 里的构造参数，涉及到**扩容问题**，比如  HashMap 的最大容量是100，那么这里设置0.75f的话，到75容量的时候就会扩容。

主要是第三个参数 `accessOrder=true` ，**这样的话 LinkedHashMap 数据排序就会基于数据的访问顺序，从而实现了 LruCache 核心工作原理**。

### 4.2 LruCache.get(K key)  
```java
/**
 * 根据 key 查询缓存，如果存在于缓存或者被 create 方法创建了。
 * 如果值返回了，那么它将被移动到队列的头部。
 * 如果如果没有缓存的值，则返回 null。
 */
public final V get(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }

    V mapValue;
    synchronized (this) {
        // LinkHashMap 如果设置按照访问顺序的话，这里每次get都会重整数据顺序
        mapValue = map.get(key);
        // 计算 命中次数
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        // 计算 丢失次数
        missCount++;
    }

    /*
     * 官方解释：
     * 尝试创建一个值，这可能需要很长时间，并且Map可能在create()返回的值时有所不同。如果在create()执行的时
     * 候，一个冲突的值被添加到Map，我们在Map中删除这个值，释放被创造的值。
     */
    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }

    /***************************
     * 不覆写create方法走不到下面 *
     ***************************/

    /*
     * 正常情况走不到这里
     * 走到这里的话 说明 实现了自定义的 create(K key) 逻辑
     * 因为默认的 create(K key) 逻辑为null
     */
    synchronized (this) {
        // 记录 create 的次数
        createCount++;
        // 将自定义create创建的值，放入LinkedHashMap中，如果key已经存在，会返回 之前相同key 的值
        mapValue = map.put(key, createdValue);

        // 如果之前存在相同key的value，即有冲突。
        if (mapValue != null) {
            /*
             * 有冲突
             * 所以 撤销 刚才的 操作
             * 将 之前相同key 的值 重新放回去
             */
            map.put(key, mapValue);
        } else {
            // 拿到键值对，计算出在容量中的相对长度，然后加上
            size += safeSizeOf(key, createdValue);
        }
    }

    // 如果上面 判断出了 将要放入的值发生冲突
    if (mapValue != null) {
        /*
         * 刚才create的值被删除了，原来的 之前相同key 的值被重新添加回去了
         * 告诉 自定义 的 entryRemoved 方法
         */
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        // 上面 进行了 size += 操作 所以这里要重整长度
        trimToSize(maxSize);
        return createdValue;
    }
}
```
上述的 `get` 方法表面并没有看出哪里有实现了Lru的缓存策略。主要在 `mapValue = map.get(key)`;里，**调用了 LinkedHashMap 的 get 方法，再加上 LruCache 构造里默认设置 LinkedHashMap 的 accessOrder=true**。


### 4.3 LinkedHashMap.get(Object key)
```java
/**
 * Returns the value of the mapping with the specified key.
 *
 * @param key
 *            the key.
 * @return the value of the mapping with the specified key, or {@code null}
 *         if no mapping for the specified key is found.
 */
@Override public V get(Object key) {
    /*
     * This method is overridden to eliminate the need for a polymorphic
     * invocation in superclass at the expense of code duplication.
     */
    if (key == null) {
        HashMapEntry<K, V> e = entryForNullKey;
        if (e == null)
            return null;
        if (accessOrder)
            makeTail((LinkedEntry<K, V>) e);
        return e.value;
    }

    int hash = Collections.secondaryHash(key);
    HashMapEntry<K, V>[] tab = table;
    for (HashMapEntry<K, V> e = tab[hash & (tab.length - 1)];
         e != null; e = e.next) {
        K eKey = e.key;
        if (eKey == key || (e.hash == hash && key.equals(eKey))) {
            if (accessOrder)
                makeTail((LinkedEntry<K, V>) e);
            return e.value;
        }
    }
    return null;
}
```
其实仔细看 `if (accessOrder)` 的逻辑即可，如果  `accessOrder=true` 那么每次 `get` 都会执行 N 次  `makeTail(LinkedEntry<K, V> e)` 。

接下来看看

### 4.4 LinkedHashMap.makeTail(LinkedEntry<K, V> e)
```java
/**
 * Relinks the given entry to the tail of the list. Under access ordering,
 * this method is invoked whenever the value of a  pre-existing entry is
 * read by Map.get or modified by Map.put.
 */
private void makeTail(LinkedEntry<K, V> e) {
    // Unlink e
    e.prv.nxt = e.nxt;
    e.nxt.prv = e.prv;

    // Relink e as tail
    LinkedEntry<K, V> header = this.header;
    LinkedEntry<K, V> oldTail = header.prv;
    e.nxt = header;
    e.prv = oldTail;
    oldTail.nxt = header.prv = e;
    modCount++;
}
```

*// Unlink e*  
<img src="http://ww2.sinaimg.cn/large/006lPEc9jw1f36m59c4tgj31kw2c7tgn.jpg" width="500x"/>  

*// Relink e as tail*  
<img src="http://ww3.sinaimg.cn/large/006lPEc9jw1f36m68rkisj31kw1eswnd.jpg" width="500x"/>  

LinkedHashMap是双向循环链表，然后此次 **LruCache.get -> LinkedHashMap.get** 的数据就被放到最末尾了。

**以上就是 LruCache 核心工作原理(｡>﹏<｡)**

---

接下来介绍 **LruCache 的容量溢出策略**

上述展示场景中，7M的容量，我添加三张图后，不会溢出，put<4>后必然会超过7MB。

### 4.5 LruCache.put(K key, V value)
```java
public final V put(K key, V value) {
    ...
    synchronized (this) {
        ...
        // 拿到键值对，计算出在容量中的相对长度，然后加上
        size += safeSizeOf(key, value);
        ...
    }
	...
    trimToSize(maxSize);
    return previous;
}
```
记住几点：
- **1.**put 开始的时候确实是把值放入 LinkedHashMap 了，**不管超不超过你设定的缓存容量**。
- **2.**然后根据 `safeSizeOf` 方法计算 此次添加数据的容量是多少，并且加到 `size` 里 。
- **3.**说到 `safeSizeOf` 就要讲到 `sizeOf(K key, V value)` 会计算出此次添加数据的大小 （像上面的 Demo，我的容量是7MB，我每次添加进来的 Bitmap 要是不覆写 sizeOf 方法的话，会视为该 bitmap 的容量计算为默认的容量计算 return 1。如此一来，这样的话 7MB 的 LruCache 容量可以放7x1024x1024张图片？明显这样的逻辑是不对的！）
- **4.**直到 put 要结束时，进行了 `trimToSize` 才判断 `size` 是否 大于 `maxSize` 然后进行最近很少访问数据的移除

### 4.6 LruCache.trimToSize(int maxSize)
```java
public void trimToSize(int maxSize) {
    /*
     * 这是一个死循环，
     * 1.只有 扩容 的情况下能立即跳出
     * 2.非扩容的情况下，map的数据会一个一个删除，直到map里没有值了，就会跳出
     */
    while (true) {
        K key;
        V value;
        synchronized (this) {
            // 在重新调整容量大小前，本身容量就为空的话，会出异常的。
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(
                        getClass().getName() + ".sizeOf() is reporting inconsistent results!");
            }
            // 如果是 扩容 或者 map为空了，就会中断，因为扩容不会涉及到丢弃数据的情况
            if (size <= maxSize || map.isEmpty()) {
                break;
            }

            Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
            key = toEvict.getKey();
            value = toEvict.getValue();
            map.remove(key);
            // 拿到键值对，计算出在容量中的相对长度，然后减去。
            size -= safeSizeOf(key, value);
            // 添加一次收回次数
            evictionCount++;
        }
        /*
         * 将最后一次删除的最少访问数据回调出去
         */
        entryRemoved(true, key, value, null);
    }
}
```
简单描述：会判断之前 `size` 是否大于 `maxSize` 。是的话，直接跳出后什么也不做。不是的话，证明已经溢出容量了。由 `makeTail` 图已知，最近经常访问的数据在最末尾。拿到一个存放 key 的 Set，然后一直一直从头开始删除，删一个判断是否溢出，直到没有溢出。

---

最后看看 

### 4.7 覆写 entryRemoved 的作用

entryRemoved被LruCache调用的场景：
- **1.（put）** put 发生 key 冲突时被调用，**evicted=false，key=此次 put 的 key，oldValue=被覆盖的冲突 value，newValue=此次 put 的 value**
- **2.（trimToSize）** trimToSize 的时候，只会被调用一次，就是最后一次被删除的最少访问数据带回来。**evicted=true，key=最后一次被删除的 key，oldValue=最后一次被删除的 value，newValue=null（此次没有冲突，只是 remove）**
- **3.（remove）** remove的时候，存在对应 key，并且被成功删除后被调用。**evicted=false，key=此次 put的 key，oldValue=此次删除的 value，newValue=null（此次没有冲突，只是 remove）**
- **4.（get后半段，查询丢失后处理情景，不过建议忽略）** get 的时候，正常的话不实现自定义 `create` 的话，代码上看 get 方法只会走一半，如果你实现了自定义的 `create(K key)` 方法，并且在 你 create 后的值放入 LruCache 中发生 key 冲突时被调用，**evicted=false，key=此次 get 的 key，oldValue=被你自定义 create(key)后的 value，newValue=原本存在 map 里的 key-value**。

解释一下第四点吧：**<1>.**第四点是这样的，先 get(key)，然后没拿到，丢失。**<2>.**如果你提供了 自定义的 `create(key)` 方法，那么 LruCache 会根据你的逻辑自造一个 value，但是当放入的时候发现冲突了，但是已经放入了。**<3>.**此时，会将那个冲突的值再让回去覆盖，此时调用上述4.的 entryRemoved。

因为 HashMap 在数据量大情况下，拿数据可能造成丢失，导致前半段查不到，你自定义的 `create(key)` 放入的时候发现又查到了**（有冲突）**。然后又急忙把原来的值放回去，此时你就白白create一趟，无所作为，还要走一遍entryRemoved。


综上就如同注释写的一样：
```java
/**
 * 1.当被回收或者删掉时调用。该方法当value被回收释放存储空间时被remove调用
 * 或者替换条目值时put调用，默认实现什么都没做。
 * 2.该方法没用同步调用，如果其他线程访问缓存时，该方法也会执行。
 * 3.evicted=true：如果该条目被删除空间 （表示 进行了trimToSize or remove）  evicted=false：put冲突后 或 get里成功create后
 * 导致
 * 4.newValue!=null，那么则被put()或get()调用。
 */
protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {
}
```
可以参考我的 demo 里的 `entryRemoved` (｡>﹏<｡)

## 5. 开源项目中的使用

[square/picasso](https://github.com/square/picasso) 


## 6. 总结

LruCache重要的几点：

- **1.**LruCache 是通过 LinkedHashMap 构造方法的第三个参数的 `accessOrder=true` 实现了 Lru 算法缓存机制。

- **2.**LruCache 在内部的get、put、remove包括 trimToSize 都是安全的（因为都上锁了）。

- **3.**覆写 `entryRemoved` 方法能知道 LruCache 数据移除是是否发生了冲突。

- **4.**`maxSize` 和 `sizeOf(K key, V value)` 方法的覆写息息相关，必须相同单位。（ 比如 maxSize 是7MB，自定义的 sizeOf 计算每个数据大小的时候必须能算出与MB之间有联系的单位 ）

- **5.**LruCache 自身并没有释放内存，将 LinkedHashMap 的数据移除了，如果数据还在别的地方被引用了，还是有泄漏问题，还需要手动释放内存。





## 7. 资源

[LruCacheActivity](https://github.com/CaMnter/AndroidLife/blob/master/app/src/main/java/com/camnter/newlife/views/activity/lrucache/LruCacheActivity.java)    


[LruCache注释源码](https://github.com/CaMnter/AndroidLife/blob/master/app/src/main/java/com/camnter/newlife/utils/cache/LruCache.java)   

 

 

