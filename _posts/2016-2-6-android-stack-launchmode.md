---
layout:     post
title:      "Android中Activity栈及Activity的启动方式"
subtitle:   "读书笔记"
date:       2015-11-25 17:54:30
author:     "Rorschach"
header-img: "img/post-bg-data-type.jpg"
tags:
    - Android
---

#### 任务及回退栈

一个应用程序通常包括多个Activity，每个Activity都应该对应用户的某种特殊的行为或者需求。

`Task`是一系列能与用户交互的Activity的集合，这些Activity按照它们被启动的顺序，被系统以栈(`Back Stack`)结构管理。栈底元素为整个任务栈的发起者，栈定元素为当前获得焦点的Activity。

当一个App被启动时，如果当前环境中不存在该App的任务栈，则会为其创建一个，此后所有该App启动的`Activity`都将在这个任务栈中被管理，这个栈也被称之为`Task`。

**值的注意的是，一个`Task`中的`Activity`可以来自不同的App，同一个App中的`Activity`也可以出现在不同的`Task`中**。<br>例如在你自己的App中跳转系统Activity，被启动的系统的Activity也会被添加到你自己的App的栈中。

#### 启动模式

默认情况下，当在同一个`Task`中启动新的Activity时，将一个新的Activity的实例入栈，使其处于栈顶，获取焦点，前一个Activity仍然留在该任务栈中；当用户按下返回键时，将栈顶Activity出栈，前一个Activity处于栈顶并获取焦点。我们有以下两种方式修改Activity的启动模式：

- [在 Android Manifest 文件中指定](#manifest) 
- [使用 Intent Flag 指定](#flag) 

##### Manifest 中指定<p id = "manifest"></p>

- standard

    默认启动模式，未指定或指定为该模式时，每次启动Activity都会创建该Activity的实例，并覆盖与原有的Activity上。

    典型使用场景：撰写Email或者IM发布消息的Activity。

- singleTop

    指定为该模式时，系统在启动Activity时会判断该Activity是否已经位于栈顶，如果是的话则调用其`newIntent()`方法，否则创建一个它的实例入栈。

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
    
    与`singleTop`类似，但`singleTop`检测的是栈顶元素是否为需要启动的Activity，而`singleTask`检测的是整个栈中是否存在需要启动的`Activity`，如果存在，则将其置顶，并将其上的所有`Activity`销毁。

    假设存在下图所示任务栈，栈中存在A、B、C三个`Activity`，A为栈底
    
    |Task|
    |:--:|
    | C  |
    | B  |
    | A  |

    启动模式为`singleTask`的`Activity`A,则

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

        **该模式可用于退出app：将主Activity设置为`singleTask`，在要退出的Activity中跳转主Activity，从而清除主Activity上的所有Activity，再重写主Activity的`newIntent()`方法，在其中增加 `finish()`，从而结束整个app。**

    典型使用场景：邮件app中的收件箱Activity

- singleInstance

    声明为`singleInstance`的Activity会出现在一个单独的栈里，且该栈只会存在这么一个Activity。
    所有应用共享该Activity的实例

    典型使用场景：桌面启动器launcher

##### Intent Flag 指定<p id = "flag"></p>