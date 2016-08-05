---
title: Fragment中启动Activity分析
date: 2016-06-15 14:16:17
tags: Android开发
---

Fragment中启动Activity有两种调用方式：

- 直接调用：`startActivity()`或`startActivityForResult()`
- 通过宿主Activity调用：`getActivity().startActivity()`或`getActivity().startActivityForResult()`

如果不关心启动的Activity的返回结果，那么通过这两种方式调用对于调用者来说没有什么区别。

如果调用者需要关心启动的Activity的返回结果，那就需要注意了。简单来说就是，如果直接调用Fragment的`startActivityForResult()`，那么就需要在对应的Fragment中重载`onActivityResult()`方法；如果通过宿主Activity调用`getActivity().startActivityForResult()`，那么就需要在宿主Activity中重载`onActivityResult()`方法。

通过Activity启动新的Activity是我们做Android开发的第一课，这个没什么好说的。Fragment并不是Activity的组件，它是如何通知Android Framework来启动新的Activity的呢？还是来看看源码吧。

``` java
public void startActivity(Intent intent) {
    if (mHost == null) {
        throw new IllegalStateException("Fragment " + this + " not attached to Activity");
    }
    mHost.onStartActivityFromFragment(this /*fragment*/, intent, -1);
}

public void startActivityForResult(Intent intent, int requestCode) {
    if (mHost == null) {
        throw new IllegalStateException("Fragment " + this + " not attached to Activity");
    }
    mHost.onStartActivityFromFragment(this /*fragment*/, intent, requestCode);
}
```

最终会调用`HostCallbacks.onStartActivityFromFragment()`：

``` java
@Override
public void onStartActivityFromFragment(Fragment fragment, Intent intent, int requestCode) {
    FragmentActivity.this.startActivityFromFragment(fragment, intent, requestCode);
}
```

看到这里应该可以大胆猜测了，通过Fragment启动Activity最终还是要借用宿主Activity来完成。我们继续跟踪，来验证我们的猜测：

``` java
/**
  * Called by Fragment.startActivityForResult() to implement its behavior.
  */
public void startActivityFromFragment(Fragment fragment, Intent intent, int requestCode) {
    if (requestCode == -1) {
        super.startActivityForResult(intent, -1);
        return;
    }
    if ((requestCode & 0xffff0000) != 0) {
        throw new IllegalArgumentException("Can only use lower 16 bits for requestCode");
    }
    super.startActivityForResult(intent, ((fragment.mIndex + 1) << 16) + (requestCode & 0xffff));
}
```

看到这里我们可以下定论了，Fragment自身没有和Android Framework交互的能力，需要借助于宿主Activity。但既然最终是通过宿主Activity来完成启动新的Activity的操作，那上面提到的Fragment中重载`onActivityResult()`方法还有什么意义呢？其实看上面的代码，宿主Activity先对我们的requestCode进行判断（高16位必须全为0），然后修改了最终传给Framework层的requestCode（Fragment的索引加1作为高16位，然后加上我们的requestCode）。为什么要这么做呢？requestCode的作用是标识要启动的新的Activity，以获取该Activity关闭时获取对应的结果。既然这样，我们就来看看结果：

``` java
/**
  * Dispatch incoming result to the correct fragment.
  */
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    mFragments.noteStateNotSaved();
    int index = requestCode >> 16;
    if (index != 0) {
        index--;
        final int activeFragmentsCount = mFragments.getActiveFragmentsCount();
        if (activeFragmentsCount == 0 || index < 0 || index >= activeFragmentsCount) {
            Log.w(TAG, "Activity result fragment index out of range: 0x"
                    + Integer.toHexString(requestCode));
            return;
        }
        final List<Fragment> activeFragments =
                mFragments.getActiveFragments(new ArrayList<Fragment>(activeFragmentsCount));
        Fragment frag = activeFragments.get(index);
        if (frag == null) {
            Log.w(TAG, "Activity result no fragment exists for index: 0x"
                    + Integer.toHexString(requestCode));
        } else {
            frag.onActivityResult(requestCode & 0xffff, resultCode, data);
        }
        return;
    }

    super.onActivityResult(requestCode, resultCode, data);
}
```

这里就跟上面对我们的requestCode的修改对应起来了，计算出Fragment的索引，如果索引不为0，根据索引找到对应的Fragment，调用Fragment的`onActivityResult()`；如果索引为0（requestCode的高16位全部为0），那就直接交给宿主Activity来处理。这里可能有疑问了，上面修改过的requestCode的高16位不可能全部为0啊，这里的判断有什么意义呢？的确，经过修改的requestCode的高16位不可能全部为0，但这里需要注意，我们上面分析的是直接调用Fragment的`startActivity()`或`startActivityForResult()`的情况，宿主Activity直接启动新的Activity就不会对requestCode进行修改了，看源码：

``` java
/**
  * Modifies the standard behavior to allow results to be delivered to fragments.
  * This imposes a restriction that requestCode be <= 0xffff.
  */
@Override
public void startActivityForResult(Intent intent, int requestCode) {
    if (requestCode != -1 && (requestCode & 0xffff0000) != 0) {
        throw new IllegalArgumentException("Can only use lower 16 bits for requestCode");
    }
    super.startActivityForResult(intent, requestCode);
}
```

传入的requestCode的高16位为什么必须全部为0？看到上面对index的计算，应该就明白了吧。

这里还有一点需要注意，如果你需要在Fragment中调用`startActivityForResult()`，并且该Fragment对应的宿主Activity中重载了`onActivityResult()`，那么在该重载函数中必须调用`super.onActivityResult()`。

### **直接调用Fragment的`startActivityForResult()`有一个坑**

在嵌套的Fragment中直接调用`startActivityForResult()`，但是在该Fragment的`onActivityResult()`收不到回调。这是由于上面宿主Activity的`onActivityResult()`中对于result的分发没有考虑Fragment嵌套的情况，如果在项目中用到了嵌套的Fragment需要自己处理分发。
