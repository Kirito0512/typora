主要总结下，startService, startForeground 和bindService，以及Service和IntentService的区别，免得忘了

### 1. 启动方式



> 虽然这里分开讨论 启动服务 和 绑定服务，但服务可同时以这两种方式运行，换言之，它既可以是启动服务（以无限期运行），亦支持绑定。唯一的问题在于您是否实现一组回调方法：`onStartCommand()`（让组件启动服务）和 `onBind()`（实现服务绑定） 



- **startService** ---- 启动服务

  > 启动服务由另一个组件通过调用startService启动，会回调 **onStartCommand()** 。
  >
  > 服务会一直运行，生命周期独立于启动它的组件。
  >
  > 即使系统已经销毁启动服务的组件，该服务然可以后台无限期运行。除非必须回收内存资源，否则系统不会停止或销毁服务。

  - 停止

    多个服务启动请求，会多次回调 onStartCommand() 。要停止服务，只需要要一个服务停止请求即可。

    stopSelf() 或者 stopService() 都可以。

- **bindService** ------ 绑定服务

  和启动服务的在应用场景上的区别

  > 如需与 Activity 和 其他应用组件中的服务进行交互， 或需要通过进程间通信(IPC)，向其他应用公开某些应用功能，则应该使用绑定服务。

  

  > 绑定服务允许应用组件 调用 bindService() 与其绑定，从而创建长期连接。
  >
  > 服务只会在组件与之绑定的时期运行。
  >
  > 多个组件可以同时绑定到该服务，组件全部取消绑定时，系统会将Service销毁。

-  **前台服务**

  > 前台服务是用户主动意识到的一种服务，因此在内存不足时，系统也不会考虑将其终止。
  >
  > 前台服务必须在状态栏展示通知。除非将服务停止，或从前台移除，否则不能清除通知。
  
  - 一般我们是先创建服务，再推到前台。
  
  > Android 8.0 之后这个过程还有变化，可以看另一篇笔记<u>Android O 之后是用前台服务</u>。
  
  - 停止前台服务
  
    在 onDestroy() 中使用 stopForeground()
  
  
  
  ### 2. 何时被系统销毁
  
  > 只有在内存过低且必须回收系统资源以供拥有用户焦点的 Activity 使用时，Android 系统才会停止服务。
  
  
  
  > 如果将服务绑定到拥有用户焦点的 Activity，则它其不太可能会终止.
  
  
  
  > 如果将服务声明为[在前台运行](https://developer.android.com/guide/components/services?hl=zh-cn#Foreground)，则其几乎永远不会终止。
  
  
  
  ### 3. IntentService
  
  #### 3.1 IntentService概述
  
  > Service默认情况下，会在应用的主线程中运行。如果服务要执行密集型或阻止性操作，需要再服务内创建新线程。
  
  IntentService 内部是用 HandlerThread + Handler ，实现了将请求放在子线程处理。
  
  先看下Service的onStart方法
  
  ```java
  public void onStart(@Nullable Intent intent, int startId) {
      Message msg = mServiceHandler.obtainMessage();
      msg.arg1 = startId;
      msg.obj = intent;
      mServiceHandler.sendMessage(msg);
  }
  ```
  
  再来看Handler的实现
  
  ```
  private final class ServiceHandler extends Handler {
      public ServiceHandler(Looper looper) {
          super(looper);
      }
  
      @Override
      public void handleMessage(Message msg) {
          onHandleIntent((Intent)msg.obj);
          stopSelf(msg.arg1);
      }
  }
  ```
  
  如果不要求服务同时处理多个请求，此类为最佳选择。
  
  默认实现会把Intent依次发送到onHandleIntent() 实现。处理完后会自动停止服务。
  
  
  
  #### 3.2 onStartCommand()方法
  
  **onStartCommand()** 的返回值，用于描述系统终止服务时，如何继续服务。
  
  下面是默认实现，默认情况下是**START_NOT_STICKY**
  
  ```
  public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
      onStart(intent, startId);
      return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
  }
  ```
  
  - **START_NOT_STICKY**
  
    如果系统在 `onStartCommand()` 返回后终止服务，则除非有待传递的挂起 Intent，否则系统*不会*重建服务。这是最安全的选项，可以避免在不必要时以及应用能够轻松重启所有未完成的作业时运行服务。
  
  - **START_STICKY**

    如果系统在 `onStartCommand()` 返回后终止服务，则其会重建服务并调用 `onStartCommand()`，但*不会*重新传递最后一个 Intent。相反，除非有挂起 Intent 要启动服务，否则系统会调用包含空 Intent 的 `onStartCommand()`。在此情况下，系统会传递这些 Intent。此常量适用于不执行命令、但无限期运行并等待作业的媒体播放器（或类似服务）。
  
  - **START_REDELIVER_INTENT**
  
    如果系统在 `onStartCommand()` 返回后终止服务，则其会重建服务，并通过传递给服务的最后一个 Intent 调用 `onStartCommand()`。所有挂起 Intent 均依次传递。此常量适用于主动执行应立即恢复的作业（例如下载文件）的服务。
  
  #### 4. reference
  
  [服务概览](https://developer.android.com/guide/components/services?hl=zh-cn)
  
  [Android Service和IntentService知识点详细总结](https://juejin.im/post/5914431944d904006c3fae59#heading-0)

