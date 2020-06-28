之前复习了Android事件分发的机制，回忆起了Activity->DecorView->ViewGroup->View整个分发的流程。

但是比较好奇事件是怎么传递到Activity的，直到看了下面的博客，[原来Android触控机制竟是这样的？](https://www.jianshu.com/p/b7cef3b3e703)

记录一下这个流程吧，防止面试官杠精。。



大致来说是

1. 首先是底层C++封装点击事件，包装成Event
2. ViewRootImpl 的 **WindowInputEventReceiver.onInputEvent** 收到 Event
3. 收到后按照 enqueueInputEvent - doProcessInputEvents - deliverInputEvent 最后调用到 **ViewPostImeInputStage.onProcess** 方法
4. onProcess中processPointerEvent方法，会调用 mView.dispatchPointerEvent(event) ，mView 为 DecorView 对象，这个方法最终调用到 **DecorView.dispatchTouchEvent** 方法

```
    public boolean dispatchTouchEvent(MotionEvent ev) {
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
    }
```

5. mWindow.getCallBack 获取的是 Activity对象，点击事件传递到了**Activity.dispatchTouchEvent**方法

 