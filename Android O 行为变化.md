### 后台执行限制

#### 1. 后台Service限制

> 处于空闲状态时，应用可以使用的后台 Service 存在限制。 这些限制不适用于前台 Service，因为前台 Service 更容易引起用户注意。

这里的”后台“和”前台“的区分，看官网文档的中文资料，还是有点模棱两可的，看了英文资料才明白。

系统可以区分 foreground(前台) app 和 background(后台) app。

#### 1.1 前台 & 后台

如果满足以下任意一个条件，我们就认为他是 前台app

- It has a visible activity, whether the activity is started or paused.即app有一个可见的activity，started和paused状态的都可以。( 我记得如果activity没有走onStop，那么他就还是可见的，就正好对应visible activity的意思吧)
- It has a foreground service.具有前台Service
- Another foreground app is connected to the app, either by binding to one of its services or by making use of one of its content providers. 也就是说，另一个前台的应用关联到这个app，无论是通过service还是content provider，都可以。

如果不满足以上的条件，app将被视为处于后台。

(前台后台是可以彼此转换的，一个app不是永远都是前台，或者永远都是后台)

#### 1.2  前台service

当app处于foreground状态，可以自由的 创建和运行，前台和后台Service。

进入后台时，在一个持续数分钟的时间窗内，应用仍可以创建和使用 Service。 在该时间窗结束后，应用将被视为处于*空闲*状态。 此时，系统将停止应用的后台 Service，就像应用已经调用 Service 的 `Service.stopSelf()` 方法一样。

#### 1.2.1 创建前台服务

在 Android 8.0 之前，创建前台 Service 的方式通常是先创建一个后台 Service，然后将该 Service 推到前台。

**<u>Android 8.0 有一项复杂功能：系统不允许后台应用创建后台 Service。</u>** 

因此，Android 8.0 引入了一种全新的方法，即 `startForegroundService()`，以在前台启动新 Service。 

在系统创建 Service 后，应用有五秒的时间来调用该 Service 的 `startForeground()` 方法以显示新 Service 的用户可见通知。

 如果应用在此时间限制内*未*调用 `startForeground()`，则系统将停止此 Service 并声明此应用为 [ANR](https://developer.android.com/training/articles/perf-anr.html)。

#### 1.2.2 startForeground与startForgroundService的区别

> 在Android 9.0 的设备上试了 开启前台服务，关于Android 8.0 以上使用前台服务的坑，下一篇再说。

就像 1.2.1 中说的，我们一般都是先用 *startForegroundService* 开启后台服务。

之后在Service中，使用 *startForground* 将服务推到前台。

但是，跟先用 *startService*，再*startForeground*，有什么区别呢？

这个答案在，*startForegroundService*的方法注释里有

```
Similar to {@link #startService(Intent)}, but with an implicit promise that the
Service will call {@link android.app.Service#startForeground(int, android.app.Notification) startForeground(int, android.app.Notification)} once it begins running.  The service is given an amount of time comparable to the ANR interval to do this, otherwise the system will automatically stop the service and declare the app ANR.
     
Unlike the ordinary {@link #startService(Intent)}, this method can be used at any time, regardless of whether the app hosting the service is in a foreground state.
```



> 与startService类似。
>
> 但是如果使用它来开启后台服务，那它之后要使用 startForeground方法，将服务推到前台。
>
> service会被赋予一个ANR的时间(5s)来将自己推到前台，否则系统会自动停止service，并声明ANR。



> 与startService的不同点在于，我们可以在任何时候是用这个方法，不用考虑启动service的app是否处于，foreground(前台)状态。而startService，不能在app处于后台状态时是用。



#### 2. 广播限制

>除了有限的例外情况，应用无法使用清单注册隐式广播。 它们仍然可以在运行时注册这些广播，并且可以使用清单注册专门针对它们的显式广播。



### 3. JobScheduler

在大多数情况下，应用都可以使用 `JobScheduler` 作业克服这些限制。 这种方法允许应用安排其在未活跃运行时执行工作，不过仍能够使系统可以在不影响用户体验的情况下安排这些作业。 Android 8.0 提供针对 `JobScheduler` 的多项改进，让您可以更轻松地使用计划作业取代 Service 和广播接收器。



### 4. JobIntentService

`IntentService` 是一项 Service，因此其遵守针对后台 Service 的新限制。 因此，许多依赖 `IntentService` 的应用在适配 Android 8.0 或更高版本时无法正常工作。

 出于这一原因，[Android 支持库 26.0.0](https://developer.android.com/topic/libraries/support-library/revisions.html#26-0-0) 引入了一个新的`JobIntentService`类，该类提供与 `IntentService` 相同的功能，但在 Android 8.0 或更高版本上运行时使用Job而非 Service。

以上的区别，可以在JobIntentService的enqueueWork_getWorkEnqueuer方法中看到



[后台执行限制](https://developer.android.com/about/versions/oreo/background?hl=zh-cn)

[Android秘技之JobService的使用详解](https://www.catbro.cn/detail/5c394d0584063683a872a591.html)

[JobIntentService详解及使用](https://blog.csdn.net/houson_c/article/details/78461751)