### 1. 检查启动时长

#### 1.1 logcat

在logcat里过滤 `Displayed` 即可（计算从 “启动进程” 到 “LAUNCH activity绘制完毕”之间的时间）

如果存在延迟加载的情况，比如有图片需要从网络加载，这种情况下，我们可以手动调用`reportFullyDrawn()`方法，通知系统我们的activity延迟加载完毕。

#### 1.2 adb shell 命令

`adb shell am start -S -W packagename/LAUNCH activity的文件路径`



### 2. 启动页优化

APP启动时会创建一个空白的启动Window，表现就是先白屏，再进入Launch Activity。

为了在视觉上看着变快一些，我们可以在AndroidManifest中，给Launch Activity设置自定义Theme，其中`android:windowBackground` 设置为，和Launch Activity的layout内容一样的，drawable文件（一般会用到layer-list）。

这样就在点击app图标后，立即展示启动页的内容。

需要注意，LaunchActivity启动后，需要手动setTheme为正常的主题。

### 3. 有意思的点

Android方法数超过65536，需要设置 `multiDexEnabled true`，这个在Android 5.0 之后默认是开启的。

5.0之前需要`multiDexEnabled true` + 使用MultiDexApplication or 在自己的application的attachBaseContext回调中 ，手动调用 `MultiDex.install(this)`

5.0以前的手机，因为不支持multi dex，所以这样兼容一下。

这个`MultiDex.install(this)` 方法非常耗时，在Android 5.0以下的手机上，这里可以有优化，我准备在面试的时候以这个作为自己做过的亮点，哈哈哈哈解决了一个面试头疼的问题。



#### 3.1 MultiDex#install 源码分析

![9F09B084-C8CB-4C53-A6D9-8C1B7DDECFF3](https://tva1.sinaimg.cn/large/00831rSTly1gdm6fui9cgj30u01e6ans.jpg)





#### 3.2 MultiDex原理

![MultiDex原理](https://tva1.sinaimg.cn/large/00831rSTly1gdm6nf89u1j30xc0fl0u0.jpg)

#### 3.3 优化的具体方式

如果按照Google的方法，在`application#attachBaseContext`中执行`MultiDex.install`，因为attachBaseContext方法的执行，在application和launch activity onCreate之前。我们的启动页必须等待install执行完毕才能展示。install里的逻辑，有相应的缓存，只要不是第一次打开，速度也很快，但是第一次打开还是比较慢。

- 不成熟的想法

为了优化这一点，首先我们能想到，在`attachBaseContext()`方法中开一个子线程执行install，这个想法本身是可以的。但是有个问题，在install方法处于子线程运行时，主线程里的逻辑会继续往后走，有可能此时出现不在主dex中的类被调用，那么就会报错了。

因此，想要使用这种方式，我们需要把启动时会用到的各个类都加入到multiDexKeepFile中。这样做，可能会导致后续维护成本变高。[声明主要 DEX 文件中必需的类](https://developer.android.com/studio/build/multidex#keep)

- 头条使用的方法

另一种方式是，在`Application#attachBaseContext`时，启动一个位于子进程的MultiDexActivity，并将其ui风格设置为与主进程的LaunchActivity一致。

然后我们的application需要等待，直到MultiDexActivity中完成 `MultiDex.install()`并 finish 自身。之后在主进程再调用一遍install方法。因为已经执行过一遍，所以这一遍会很快。之后，会按照正常流程启动LaunchActivity，因为ui一致，所以用户是感知不到这一变化的。



因此，用户首次安装并打开app时，会立刻看到同LaunchActivity一样ui的界面，不需要等待install结束。

[面试官：今日头条启动很快，你觉得可能是做了哪些优化？](https://www.jianshu.com/p/d0fe74f4e9c4)

[为方法数超过 64K 的应用启用多 dex 文件](https://developer.android.google.cn/studio/build/multidex#java)


