![](https://tva1.sinaimg.cn/large/006tNbRwly1ga24cp10afj31080pg40i.jpg)

### 1. 定义启动模式

我们可以通过启动模式来定义一个 activity 实例如何与当前 task(任务栈) 交互。有以下两种方式

- 使用manifest文件

在manifest文件中定义activity时， [通过launchMode定义](https://developer.android.com/guide/components/tasks-and-back-stack.html?hl=zh-cn#ManifestForTasks)

- 使用intent flag

调用 startActivity() 方法时，给 intent 添加对应的 flag

> 注意如果同时使用两种方式，那么以intent flag 中的定义为优先

#### 1.1 使用Intent flag

- FLAG_ACTIVITY_NEW_TASK

会产生和 **singleTask** 一样的行为。如果activity已经运行在对应栈中，那么把这个栈移到前台，同时回调 **onNewIntent()** 方法。

- FLAG_ACTIVITY_SINGLE_TOP

和 **singleTop** 一样。如果activity自己启动自己，自己位于栈顶，那么回调 **onNewIntent()** 方法，不会重新创建。

- FLAG_ACTIVITY_CLEAR_TOP

如果要启动的activity A 已经在当前task栈中运行，那么不会创建新实例，系统会清理掉 A 上面的所有activity，将 A 置于栈顶

#### 1.2 CLEAR_TOP的各种组合使用

- **FLAG_ACTIVITY_CLEAR_TOP**

  如果要启动的activity A 已经在 **当前task栈** 中运行，那么系统会清理掉 A 上面的所有activity，将 A 置于栈顶， A 会**销毁后重建**

- **FLAG_ACTIVITY_CLEAR_TOP** + **FLAG_ACTIVITY_SINGLE_TOP**

  与上一条类似，但是设置了SINGLE_TOP 的 flag，A 不会被销毁，而是回调 **onNewIntent**

- **FLAG_ACTIVITY_CLEAR_TOP** + **FLAG_ACTIVITY_NEW_TASK**

  一起使用时，通过这些标志，可以找到 **其他task栈** 中正在运行的 A ，将其**销毁并重建**

- **FLAG_ACTIVITY_CLEAR_TOP** + **FLAG_ACTIVITY_SINGLE_TOP** + **FLAG_ACTIVITY_NEW_TASK**

  一起使用，可以找到任意一个task栈中正在运行的 A ，回调 **onNewIntent()**

> 为了好记，我猜测，SINGLE_TOP 负责管理是否重建，NEW_TASK 负责管理是否找遍所有task栈



### 2. taskAffinity属性

只在两种情况下有效：

- activity 的launchMode 为 singleTask ， 或者设置了 `intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK)`

- activity设置了 `android:allowTaskReparenting = true` （默认为false）



### 3. allowTaskReparent 的作用

当把已启动 Activity ，想要的任务栈 转至前台时，Activity 是否能从当前所在的任务栈，转移至其 想要的任务栈

 —“`true`”表示可以转移，“`false`”表示仍须留在启动它的任务处。

如果未设置该属性，则对 Activity 应用，由`<application>   ` 元素的 `allowTaskReparenting` 属性，所设置的值。默认值为“`false`”。



> 《Android开发艺术探索》中有个例子：

当一个应用A启动了应用B的某个Activity后，如果这个Activity的allowTaskReparenting属性设置为true，那么当应用B被启动，此Activity会直接从应用A的任务栈转移到应用B的任务栈中。

具体来说，现在有两个应用A和B，A启动了B的一个Activity C，然后按Home键回到桌面。

再单击B的桌面图标，这个时候不是启动了B的主Activity，而是重新显示了已经被应用A启动的Activity C。

我们也可以理解为，C从A的任务栈转移到了B的任务栈中。



> 例如，如果电子邮件消息包含网页链接，则点击该链接会调出可显示该网页的 Activity。该 Activity 由浏览器应用定义，但作为电子邮件任务的一部分启动。如果将该 Activity 的父项更改为浏览器任务，则它会在浏览器下一次转至前台时显示，在电子邮件任务再次转至前台时消失。



可以这么理解，由于A启动了C，这个时候C只能运行在A的任务栈中，但是C属于B应用，正常情况下，它的TaskAffinity值肯定不可能和A的任务栈相同，所以当B启动后，B会创建自己的任务栈，这个时候系统发现C原本想要的任务栈已经创建了，所以就把C从A的任务栈中转移过来了。



### 4. finishOnTaskLaunch

每当用户再次启动 Activity 的任务（在主屏幕上选择任务）时，是否应关闭现有的 Activity 实例

 —“`true`”表示应关闭，“`false`”表示不应关闭。默认值为“`false`”。



#### 4.1 + allowTaskReparenting

如果此属性和 `allowTaskReparenting` 均为“`true`”，则优先使用此属性。

系统会忽略 Activity 的affinity属性，activity不会被re-parent，而是被销毁。



### 5. alwaysRetainTaskState

系统是否始终保持 Activity 所在任务的状态 

“`true`”表示是，“`false`”表示允许系统在特定情况下将任务重置到其初始状态。

默认值为“`false`”。该属性只对任务的根 Activity 有意义；所有其他 Activity 均可忽略该属性。

正常情况下，当用户从主屏幕重新选择某个任务时，系统会在特定情况下清除该任务（从根 Activity 上的堆栈中移除所有 Activity）。通常，如果用户在一段时间（如 30 分钟）内未访问任务，系统会执行此操作。

不过，如果该属性的值是“`true`”，则无论用户如何返回任务，该任务始终会显示最后一次的状态。例如，该属性非常适用于网络浏览器这类应用，因为其中存在大量用户不愿丢失的状态（如多个打开的标签）。

### 6. clearTaskOnLaunch

每当从主屏幕重新启动任务时，是否都从该任务中移除根 Activity 之外的所有 Activity 

`true`表示始终将任务清除至只剩其根 Activity；`false`表示不清除。默认值为`false`。

该属性只对启动新任务的 Activity（根 Activity）有意义；任务中的所有其他 Activity 均可忽略该属性。

若值为“`true`”，则每次当用户再次启动任务时，无论用户最后在任务中正在执行哪个 Activity，也无论用户是使用*返回*还是*主屏幕*按钮离开，系统都会将用户转至任务的根 Activity。当值为“`false`”时，可在某些情况下清除任务中的 Activity（请参阅 `alwaysRetainTaskState` 属性），但也有例外。



#### 6.1 +allowTaskReparenting

如果把该属性和 `allowTaskReparenting` 的值均为“`true`”，则 **所有可 re-parent (更改父项) 的 Activity** ，都将转移至，与他们的affinity属性一致的task中。而其余 Activity 随即会被移除。





[<activity> | Android 开发者](https://developer.android.google.cn/guide/topics/manifest/activity-element#finish)

[任务和返回栈 | Android Developers](https://developer.android.com/guide/components/tasks-and-back-stack.html?hl=zh-cn)