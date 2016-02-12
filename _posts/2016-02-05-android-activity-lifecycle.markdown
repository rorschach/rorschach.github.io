---
layout:     post
title:      "Activity的生命周期"
subtitle:   "读书笔记"
date:       2016-02-05 12:00:00
author:     "Rorschach"
header-img: "img/post-bg-circle.jpg"
tags:
    - android
---

## 概述

>An activity is a single, focused thing that the user can do. Almost all activities interact with the user, so the Activity class takes care of creating a window for you in which you can place your UI with setContentView(View).

Activity是Application和用户交互的核心组件， Android系统采用栈的方式管理Activity。

## 状态

- Active/Running
    
    此时Activity处于前台，并可以与用户交互的状态，位于Activity栈的顶端。

- Paused
    
    当Activity部分被挡住或者被一个透明的Activity盖住时，处于该状态。此时系统仍保存其所有的状态信息及成员变量，但此时该Activity失去与用户交互的能力。

- Stopped
    
    当Activity完全不可见的时候进入该状态，此时该Activity处于后台，系统仍保存其所有的状态信息及成员变量。

- Killed
    
    当Activity被系统回收或者未被创建时处于该状态。

## 生命周期

最经典的生命周期图

![lifecycle1](http://developer.android.com/images/activity_lifecycle.png )

常用的讲解图

![lifecycle2](http://developer.android.com/images/training/basics/basic-lifecycle.png)

作为开发者，我们不需要实现所有的生命周期的回调方法，只需要选择性的实现部分即可。
Activity只有三个状态是稳定的，其余状态均为过渡状态，很快就会结束掉。

- Resumed 
    对应上文中的Running状态

- Paused
    对应上文中的Paused状态

- Stopped
    对应上文中的Stopped状态

### 典型情况下的生命周期分析

#### Activity的启动和销毁

![start&destory](http://developer.android.com/images/training/basics/basic-lifecycle-create.png)

Activity的启动过程为上图中的①②③，其中

- onCreate()

做一些初始化操作，例如调用`setContentView()`加载布局资源，创建需要的UI元素，初始化`Activity`需要的数据等等。<br>注意此处不能执行耗时任务，否则会影响Activitiy的进入时间。

```
@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState); 

    // 加载布局资源
    setContentView(R.layout.main_activity);

    //创建UI
    
    ...
}
```

- onStart()

表示`Activity`正在启动，即将开始。此时`Activity`已经处于可见状态，只是还未位于前台，无法和用户交互。

- onResume()

此时`Activity`已经位于前台，可以和用户交互。

- onDestory()

和`onCreate()`对应的方法有`onDestory()`，其对应的是`Activity`的销毁过程，表示`Activity`即将被销毁，一般在此处做最后的资源的回收和释放。

```
@Override
public void onDestroy() {
    super.onDestroy();  // 总是调用父类的同名方法
    
    // 关闭Activity中的方法，清除引用
    android.os.Debug.stopMethodTracing();

    //线程开启后，不会在Activity销毁后自动的销毁，因此需要清除开启的线程
}
```


#### Activity暂停和恢复

![pause&resume](http://developer.android.com/images/training/basics/basic-lifecycle-paused.png)

栈顶的`Acrivity`部分不可见后，会导致其进入`Pause`状态，此时会调用`onPasue()`，恢复完全可见后，会调用`onResume()`方法来恢复到`Resume`状态。

Activity的暂停和恢复分别对应上图的①和②，其中

- onPasue()

表示Activity正在停止，正常情况下紧接着就会调用`onStop()`方法，特殊情况下会再次调用`onRsume`，例如快速的回到当前应用。在`onPause()`可以做一些释放系统资源的操作，例如`Camera`、`Sensor`、`Receiver`等等，也可以做一些存储数据、停止动画的操作。
注意此处的操作同样不能太过耗时，否则会影响到新的 `Acitivity`的显示。

```
@Override
public void onPause() {
    super.onPause();  // 总是先调用父类同名方法

    // 释放不需要的相机资源
    if (mCamera != null) {
        mCamera.release();
        mCamera = null;
    }
    
    ...
}
```

- onResume()

重新初始化在被`onPasue()`中释放的系统资源、取出之前存储的数据。

```
@Override
public void onResume() {
    super.onResume();  // 总是先调用父类同名方法

    // 重新获得相机资源
    if (mCamera == null) {
        initializeCamera(); // 用于获取相机的本地方法
    }
}
```

#### Activity的停止和恢复

![stop&restart](http://developer.android.com/images/training/basics/basic-lifecycle-stopped.png)

当栈顶的Activity完全不可见之后，会调用`onStop()`方法进入Stop状态，在调用`onStop()`之前，总会调用`onPause()`方法。一个应用从Stop状态恢复到Resume状态时，会调用`onRestart()`方法，而`onRestart()`会调用`onStart()`方法，最终调用`onResume`方法，如此完成了一个从停止到恢复的过程。该过程对应上图的①②③④。

- onStop()

表示Activity即将停止，此时可以做一些重量级的回收工作，但同样注意不能耗时。

```
@Override
protected void onStop() {
    super.onStop();  // 总是先调用父类同名方法

    // 保存数据
    ContentValues values = new ContentValues();
    values.put("TITLE", title);
    values.put("CONTENT", content);

    getContentResolver().update(
            mUri,    // 目标　URI 
            values,  // 要保存的数据
            null,    
            null     
            );
}
```

- onRestart()

表示Activity正在重新启动，之后会调用`onStart()`方法。

```

@Override
protected void onRestart() {
    super.onRestart();  // 总是先调用父类同名方法
}

@Override
protected void onStart() {
    super.onStart();  // 总是先调用父类同名方法
    
    // 例如此Activity需要GPS数据，可以在此处初始化LocationManager获取位置信息
    LocationManager locationManager = 
            (LocationManager) getSystemService(Context.LOCATION_SERVICE);
    boolean gpsEnabled = locationManager.isProviderEnabled(LocationManager.GPS_PROVIDER);
    
    if (!gpsEnabled) {
       // do something
    }
}
```

#### 两个问题

1. onStart()和onResume()以及onStop()和onPause()，从描述上看差不多，那有什么区别呢？
    
       `onStart()`和`onStop()`是从一个`Activity`是否可见的角度来回调的，<br>
       而`onResume()`和`onPause()`是从一个`Activity`是否位于前台这一个角度来回调的，
       <br>在实际使用中没有明显的区别。

2. 若`Activiti`Ａ启动`Activiyt`B，那么A的`onPause()`和B的`onResume()`哪个先执行呢？

    查看源码可以发现，是A的`onPause()`先调用，然后B的`onResume()`再调用。

### 异常情况下的生命周期分析

![recreate](http://developer.android.com/images/training/basics/basic-lifecycle-savestate.png)

#### 资源相关系统配置发生更改，导致Activity被杀死并重建

最典型的情况就是旋转屏幕，在启动一个`Activity`后，旋转屏幕，通过添加打印日志会发现：<br>`onPause`、`onStop`、`onDestory`、`onCreate`、`onStart`、`onResume`方法均会有打印输出，说明该`Activity`被销毁重建。
由于`Activity`是在异常情况下终止的，因此会调用`onSaveInstanceSate()`来保存当前`Activity`的状态，具体流程如上图。从时序上来说，`onRestoreInstanceState()`在`onStart()`之后被调用。

在`onSaveInstanceSate()`和`onRestoreInstanceState()`中，系统自动的为我们做了保存视图结构和恢复其状态的工作。和`Activity`一样，`View`中也有`onSaveInstanceSate()`和`onRestoreInstanceState()`两个方法，只有实现了这两个方法的`View`能够自动的保存好恢复状态。

**系统只有在`Activity`被异常终止是才会调用`onSaveInstanceSate()`和`onRestoreInstanceState()`方法来存储和恢复数据，其他情况不会触发。但是按下`Home`键或者启动新的`Activity`仍然会单独触发`onSaveInstanceSate()`方法。**

#### 资源内存不足，导致低优先级的Activity被杀死

Activity的优先级从高到低，分为：

1. 前台Activity　－－　正在和用户交互的Activity
2. 可见但非前台　－－　例如被对话框盖住的Activity
3. 后台Activity　－－　被暂停的Activity

当系统内存不足时，会按照从低到高的顺序杀死目标Activity，并通过`onSaveInstanceSate()`和`onRestoreInstanceState()`存储和恢复数据。**如果一个进程中没有四大组件在运行，那么这个进程将很快被杀死。**因此，后台任务应该放在一个`Service`中运行，以防止被系统杀死。

#### configChanges

我们已经知道Activity在系统配置发生改变之后会重新创建，是否存在不让其重建的办法？答案就是`configChanges`。
例如我们不希望一个`Activity`在旋转屏幕后重建，在`manifest`文件中对应的<activity>标签中增加一行：
```
android:configChanges="oritation"
```

可以通过"|"来连接多个选项。
一般来说，我们会写成如下格式
```
android:configChanges="oritation｜keyboardHidden|screensize"
```

增加`screensize`是api13后的要求，和编译选项有关，和运行环境无关。

在`manifest`文件中配置后，当屏幕旋转后，将不会重建`Activity`，而是调用`onConfigurationChanged()`方法实现自定义的逻辑，例如：

```
@Override
public void onConfigurationChanged(Configuration newConfig) {
    super.onConfigurationChanged(newConfig);    

    Toast.makeText(this, newConfig.orientation, Toast.LENGTH_SHORT).show();
}
```