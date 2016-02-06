---
layout:     post
title:      "Android中Activity的生命周期"
subtitle:   "读书笔记"
date:       2016-02-05 12:00:00
author:     "Rorschach"
header-img: "img/post-bg-circle.jpg"
tags:
    - Android
---

### Activity的生命周期

#### 概述
>An activity is a single, focused thing that the user can do. Almost all activities interact with the user, so the Activity class takes care of creating a window for you in which you can place your UI with setContentView(View).

Activity是Application和用户交互的核心组件， Android系统采用栈的方式管理Activity。

#### 状态

- Active/Running
    
    此时Activity处于前台，并可以与用户交互的状态，位于Activity栈的顶端。

- Paused
    
    当Activity部分被挡住或者被一个透明的Activity盖住时，处于该状态。此时系统仍保存其所有的状态信息及成员变量，但此时该Activity失去与用户交互的能力。

- Stopped
    
    当Activity完全不可见的时候进入该状态，此时该Activity处于后台，系统仍保存其所有的状态信息及成员变量。

- Killed
    
    当Activity被系统回收或者未被创建时处于该状态。

#### 生命周期

![生命周期1](http://developer.android.com/images/activity_lifecycle.png "最经典的图")

![生命周期2](http://developer.android.com/images/training/basics/basic-lifecycle.png "讲解图")

作为开发者，我们不需要实现所有的生命周期的回调方法，只需要选择性的实现部分即可。
Activity只有三个状态是稳定的，其余状态均为过渡状态，很快就会结束掉。

- Resumed 
    对应上文中的Running状态

- Paused
    对应上文中的Paused状态

- Stopped
    对应上文中的Stopped状态
