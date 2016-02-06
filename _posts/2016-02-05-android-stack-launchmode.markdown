---
layout:     post
title:      "Android中Activity栈及Activity的启动方式"
subtitle:   "读书笔记"
date:       2016-02-05 21:00:00
author:     "Rorschach"
header-img: "img/post-bg-stack.jpg"
tags:
    - Android
---

### 任务及回退栈

一个应用程序通常包括多个Activity，每个Activity都应该对应用户的某种特殊的行为或者需求。

`Task`是一系列能与用户交互的Activity的集合，这些Activity按照它们被启动的顺序，被系统以栈(`Back Stack`)结构管理。栈底元素为整个任务栈的发起者，栈定元素为当前获得焦点的Activity。

当一个App被启动时，如果当前环境中不存在该App的任务栈，则会为其创建一个，此后所有该App启动的`Activity`都将在这个任务栈中被管理，这个栈也被称之为`Task`。

**值的注意的是，一个`Task`中的`Activity`可以来自不同的App，同一个App中的`Activity`也可以出现在不同的`Task`中**。<br>例如在你自己的App中跳转系统Activity，被启动的系统的Activity也会被添加到你自己的App的栈中。

### 启动模式

默认情况下，当在同一个`Task`中启动新的Activity时，将一个新的Activity的实例入栈，使其处于栈顶，获取焦点，前一个Activity仍然留在该任务栈中；当用户按下返回键时，将栈顶Activity出栈，前一个Activity处于栈顶并获取焦点。我们有以下两种方式修改Activity的启动模式：

- [在 Android Manifest 文件中指定](#manifest) 
- [使用 Intent Flag 指定](#flag) 

<p id = "manifest"></p>
---
Manifest 中指定

```
<activity
    android:name="me.rorschach.DemoActivity"
    android:launchMode="standard | singleTop | singleTask | singleInstance"
    lable="@string/app_name"/>
```

- standard

    默认启动模式，未指定或指定为该模式时，每次启动Activity都会创建该Activity的实例，并覆盖与原有的Activity上。

    典型使用场景：撰写Email或者IM发布消息的Activity。

- singleTop

    指定为该模式时，系统在启动Activity时会判断该Activity是否已经位于栈顶，如果是的话则调用其`onNewIntent()`方法，否则创建一个它的实例入栈。

    假设存在下图所示任务栈，栈中存在A、B、C三个Activity，A为栈底，现在要启动`Activity` C

    | Task |
    |:----:|
    |  C   |
    |  B   |
    |  A   |

    + 若C的启动模式为`standard`
    
        |before|after |
        |:----:|:----:|
        |  C   |  C   |
        |  B   |  C   |
        |  A   |  B   |
        |      |  A   |  

    + 若C的启动模式为`singleTop`

        |before| after|
        |:----:|:----:|
        |  C   |  C   |
        |  B   |  B   |
        |  A   |  A   |


    典型使用场景：搜索用Activity。

- singleTask

    + 在同一个App中启动
    
    与`singleTop`类似，但`singleTop`检测的是栈顶元素是否为需要启动的Activity，而`singleTask`检测的是整个栈中是否存在需要启动的`Activity`，如果存在，则将其置顶，并将其上的所有`Activity`销毁，同时其`onNewIntent()`方法被调用。

    假设存在下图所示任务栈，栈中存在A、B、C三个`Activity`，A为栈底
    
    |Task|
    |:--:|
    | C  |
    | B  |
    | A  |

    启动模式为`singleTask`的`Activity`A，则

    |before|after|
    |:----:|:---:|
    |  C   |  A  |
    |  B   |
    |  A   |

    + 在不同的App中启动

        * 若该实例不存在,则会创建一个新的任务
     
        例如在Task1中启动`singleTask`模式的`Activity`A

        |Task1|
        |:---:|
        |  Y  |
        |  X  |

        启动后

        |Task1|Task2|
        |:---:|:---:|
        |  Y  |  A  |
        |  X  |

        * 若该实例存在，则会将该实例所属的任务栈调转到前台，并将该实例上的所有Activity销毁
      
        例如：

        |foreground|
        |:---:|
        |  Y  |
        |  X  |

        |background|
        |:---:|
        |  C  |
        |  B  |
        |  A  |

        前台app启动后台app中的`Activity`B，B的启动模式为`singleTask`，则

        |After|press back| after back |press back| after back |
        |:---:|:--------:|:----------:|:--------:|:----------:|
        |  B  |          |  A         |          |  Y         |
        |  A  |          |  Y         |          |  X         |
        |  Y  |          |  X         |          |            |
        |  X  |          |            |          |            |

        **小技巧**：该模式可用于退出app。<br>
        将主Activity设置为`singleTask`，在要退出的Activity中跳转主Activity，从而清除主Activity上的所有Activity，再重写主Activity的`newIntent()`方法，在其中增加 `finish()`，从而结束整个app。

    典型使用场景：邮件app中的收件箱Activity

- singleInstance

    声明为`singleInstance`的Activity会出现在一个单独的栈里，且该栈只会存在这么一个Activity。
    所有应用共享该Activity的实例

    典型使用场景：桌面启动器launcher

- 特殊说明

    对于singleTask和singleInstance两种启动模式，如果有一个处于其中一种模式的`Activity`A，希望通过`startActivityForResult()`的方式来启动一个`Activity`B，则系统将直接返回`Activity.RESULT_CANCELED`，而不会去等待数据的返回。因为不同`Task`直接默认是不能传递数据的，故不会有数据返回。


    startActivityForResult()
    
    >Note that this method should only be used with Intent protocols that are defined to return a result. In other protocols (such as Intent.ACTION_MAIN or Intent.ACTION_VIEW), you may not get the result when you expect. For example, if the activity you are launching uses the singleTask launch mode, it will not run in your task and thus you will immediately receive a cancel result. 

<p id = "flag"></p>
---
Intent Flag 指定

```
Intent intent = new Intent(FirstActivity.this, SecondActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_XXX);
startActivity(intent);
```

- Intent.FLAG_ACTIVITY_NEW_TASK
    
    使用一个新的`Task`来启动一个`Activity`，但是启动的每一个`Activity`都在一个新的`Task`中。通常适用于从`Service`或者使用`getApplicationContext()`启动一个新的`Activity`,因为非`Activity`的`Context`并不存在所谓的任务栈，因此需要为其创建一个任务栈，否则会报错。

    **使用此方式启动不完全等同于上文的`singleTask`模式**

- Intent.FLAG_ACTIVITY_SINGLE_TOP
    
    使用`singleTop`模式启动一个`Activity`

- Intent.FLAG_ACTIVITY_CLEAR_TOP

    具有此标记位`Activity`，若以`standard`模式启动，则同一个`Task`中所有位于其上的`Activity`都要出栈。`singleTask`模式默认具备此功能。

- Intent.FLAG_ACTIVITY_NO_HISTORY

    使用此标记位启动的`Activity`，当其启动其他`Activity`后，该`Activity`就消失了。

     例如：

    |task |
    |:---:|
    |  B  |
    |  A  |
    
        Intent intent1 = new Intent(ActivityB.this, ActivityC.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_NO_HISTORY);
        startActivity(intent1);  
        //then
        Intent intent2 = new Intent(ActivityC.this, ActivityD.class);
        startActivity(intent2);

    |task |
    |:---:|
    |  D  |
    |  B  |
    |  A  |


- Intent.FLAG_ACTIVITY_EXECULDE_FROM_RECENTS
    
    具有此标记的`Activity`不会出现在历史`Activity`的列表中，<br>
    等同于在`manifest`文件中指定`android:excludeFromRecents=true`


#### 两种方式的区别

- `flag`的优先级高于`manifest`文件中指定
- `flag`方式无法指定`singleInstance`
- `manifest`无法指定`Intent.FLAG_ACTIVITY_CLEAR_TOP`

### Task＆taskAffinity

上文中多次提到`Task`，即`任务栈`，那什么是一个`Task`呢？这就涉及到一个`manifest`文件中的参数`taskAffinity`，翻译为　任务相关性。该参数标识了一个`Activity`需要的任务栈的名字，默认为包名。该参数主要与`singleTask`及`allowTaskReparenting`想配对使用，否则无意义。

- 与`singleTask`一起使用

    标识具有该模式的`Activity`的任务栈的名字，启动的`Activity`将会运行于名字为该参数指定的`Task`中。

- 与`allowTaskReparenting`一起使用
    
    当一个App A 启动　App B中的一个`Activity`时，若该`Activity`的属性为`andoid:allowTaskReparenting=true`的话，那么App B被启动后，该`Activity`会从App A的任务栈转移到App B的任务栈中。

### 清空任务栈

系统提供了以下几种方式清空任务栈,均设置于`manifest`文件中

- clearTaskOnLaunch

    每次返回该`Activity`时，清理其上的所有`Activity`，通过该属性，使得这个`Task`每次初始化时只有这么一个`Activity`

- finishOnTaskLaunch

    当离开这个`Activity`所在的`Task`后，用户再返回时`finish`掉这个`Activity`

- alwaysRetainState

    若一个`Activity`设置为`adnroid:alwaysRetainState=true`，该`Activity`所在的`Task`不接受任何清理指令，将一直保持状态
