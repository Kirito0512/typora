从开始学Android，就一直听到 “不能在子线程更新ui” 这种说法，一直没深究，大部分时候确实是这样的。

如果在子线程更新ui，一般会有异常提示 "Only the original thread that created a view hierarchy can touch its views."

但是如果在onCreate中，用线程更新ui，其实是不会报错的。

带着疑问，我们往下看：

上面的异常提示，出现在 `ViewRootImpl`的 checkThread方法 中

```
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

#### 概述

简单来说，我们更新ui，比如TextView#setText方法，最终会调用 ViewRootImpl.requestLayout方法，这个方法中会调用checkThread。

onCreate时，ViewRootImpl还没有创建，因此 requestLayout 方法没有走，系统把你想要设置的文字保存下来，在绘制时直接绘制上去。

而通常情况下，ViewRootImpl创建好了的话，在子线程更新ui就会报错。

想要深入这个问题，我们就得大致了解 Android view绘制的流程

下面是ViewRootImpl的创建过程

1. ActivityThread.handleResumeActivity

> 内部调用 performResumeActivity 方法，内部调用 activity.performResume-> （Activity）mInstrumentation.callActivityOnResume -> activity.onResume() 回调到onResume

2. WindowManagerImpl.addView(DecorView) （底层使用）
3. WindowManagerGlobal.addView 中 创建 ViewRootImpl
4. ViewRootImpl.setView （方法里使用Binder mWindowSession.addToDisplay 跨进程操作）
5. DecorView.requestLayout
6. requestLayout->scheduleTraversals->doTraversal->performTraversals（performMeasure， performLayout，performDraw）绘制View的流程
7. View.dispatchAttachedToWindow，View会保存AttachInfo，里面有ViewRootImpl信息



