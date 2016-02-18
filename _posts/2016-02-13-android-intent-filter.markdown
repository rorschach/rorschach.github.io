---
layout:     post
title:      "IntentFilter的匹配规则"
subtitle:   "读书笔记"
date:       2016-02-13 20:00:00
author:     "Rorschach"
header-img: "img/post-bg-intent-filter.jpg"
tags:
    - android
---

## Intent的启动方式

Intent是Android系统中用于组件间通讯的桥梁，我们可以通过Intent进行组件间的信息传递，也可以用于启动其他组件，例如

- 启动Activity

```
public abstract void startActivity (Intent intent, Bundle options)      //Context

//Or

public void startActivityForResult (Intent intent, int requestCode, Bundle options)     //Activity
```

- 发送广播

```
public abstract void sendBroadcast (Intent intent)      //Context
```

- 启动服务

```
public abstract ComponentName startService (Intent service)     //Context
```


Intent分为两种，显式和隐式，官方描述如下

- 显式Intent

>specify the component to start by name (the fully-qualified class name). You'll typically use an explicit intent to start a component in your own app, because you know the class name of the activity or service you want to start. For example, start a new activity in response to a user action or start a service to download a file in the background.

- 隐式Intent

>do not name a specific component, but instead declare a general action to perform, which allows a component from another app to handle it. For example, if you want to show the user a location on a map, you can use an implicit intent to request that another capable app show a specified location on a map.

这里我们只关注隐式Intent，因为只有它需要IntentFilter来进行匹配。原则上一个Intent不应该既是显式调用又是隐式调用，如真的共存，以显式的为主。本文以启动一个`Activity`为例。

## IntentFilter的组成

隐式调用Intent要求匹配组件的IntentFilter，否则无法启动组件。
IntentFilter的过滤信息有:

- action:声明这个Intent的行为，必须为指向一个具体的行为的字符串

- category:声明这个Intent的类别，必须为指向一个具体的行为的字符串

**为了让组件接受隐式调用，必须在`IntentFilter`中添加`android.intent.category.DEFAULT`，否则系统将找不到任何`Activity`响应你的隐式Intent调用**

- data:声明这个Inten启动的`Activity`接受的数据类型


实例一

```
<activity
    android:name=".SampleActivity">

    <intent-filter>

        <action android:name="me.rorschach.action.a"/>
        <action android:name="me.rorschach.action.b"/>

        <category android:name="me.rorschach.category.c"/>
        <category android:name="me.rorschach.category.d"/>
        <category android:name="android.intent.category.DEFAULT" />   //必须添加

        <data
            android:scheme="myScheme"
            android:host="rorschach.github.io"
            android:port="8888"
            android:path="/activityDemo"
            android:mimeType="image/*"/>

    </intent-filter>

</activity>
```

另外，一个组件可以存在多个IntentFilter，只要能匹配其中任何一个就能够启动该`Activity`，例如：

实例二

```
<activity android:name="ShareActivity">
    <!-- This activity handles "SEND" actions with text data -->
    <intent-filter>
        <action android:name="android.intent.action.SEND"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="text/plain"/>
    </intent-filter>
    <!-- This activity also handles "SEND" and "SEND_MULTIPLE" with media data -->
    <intent-filter>
        <action android:name="android.intent.action.SEND"/>
        <action android:name="android.intent.action.SEND_MULTIPLE"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data android:mimeType="application/vnd.google.panorama360+jpg"/>
        <data android:mimeType="image/*"/>
        <data android:mimeType="video/*"/>
    </intent-filter>
</activity>
```

### Action的匹配规则

- Intent中如果不存在action，匹配失败
- action必须和指定的action的字符串完全一样。
- IntentFilter中可以存在多个action，匹配其中一个即可
- action区分大小写

对于实例一，若action的值为"me.rorschach.action.a"或者"me.rorschach.action.b"，都可以匹配成功

### Category的匹配规则

- Intent中可以存在多个category，但是每个都必须存在于目标的IntentFilter中
- Intent中可以没有category，因为系统在调用`startActivity()`或者`startActivityForResult()`时，会默认为Intent加上`android.intent.category.DEFAULT`

### Data的匹配规则

#### data的语法

```
<data 
    android:scheme="string"
    android:host="string"
    android:port="string"
    android:path="string"
    android:pathPattern="string"
    android:pathPrefix="string"
    android:mimetype="string"
/>
```

data由两部分组成,`URI`和`mimeType`,后者指的是媒体类型，如text/plain，video/*等，前者的结构如下：

```
<scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]
```

关于URI的知识，在上一篇文章中已经有过讲述，这里不在赘述，[点这里查看](http://rorschach.github.io/2016/02/11/android-content-provider/)。

下面分别介绍一下上述属性的意义：

- scheme:URI的模式，未指定此属性则整个URI无效
- host:主机名，未指定此属性则整个URI无效
- port:端口号，只有前两个属性指定后才有意义
- path/pathPattern/pathPrefix:<br>
    path表示完整的路径信息<br>
    pathPattern也表示完整的路径信息，但可以包含通配符`*`<br>
    pathPrefix表示路径的前缀信息

#### data的匹配规则

data的匹配分为以下两种情况

- data中包括URI和mimetype

    data的匹配规则类似于action，也要求完全一致。针对实例一，可以给出以下Intent以完全匹配其`IntentFilter`：

    ```
       Intent intent = new Intent("me.rorschach.action.a");
       intent.addCategory("me.rorschach.category.c");
       intent.addCategory("me.rorschach.category.d");
       intent.setDataAndType(
               Uri.parse("myScheme://rorschach.github.io:8888/activityDemo"),
               "image/*");
        startActivity(intent);
    ```

    值得注意的是上文代码中必须调用`setDataAndType()`方法，不能先调用`setData()`再调用`setType()`，因为后两个方法会清除彼此的值。

- data中仅包括mimetype

对以下实例

```
<intent-filter>
    ...
    <data android:mimeType="image/*" />
</intent-filter>
```

此时data中虽然没有指定URI，但是却有默认值，只有`Intent`中`URI`部分的`scheme`为`content`或者`file`才能匹配，如

```
intent.setDataAndType(Uri.parse("content://XXXXX"), "image/*");

//Or

intent.setDataAndType(Uri.parse("file://XXXXX"), "image/*");
```

另外，以下两种写法表示同样的意思，这是data的特殊之处

```
<intent-filter>
    ...
    <data android:scheme="myScheme" android:mimeType="text/plain" />
</intent-filter>

//Or

<intent-filter>
    ...
    <data android:scheme="myScheme" />
    <data android:mimeType="text/plain" />
</intent-filter>

```

## And More

- IntentFilter的匹配规则对于`BroadcastReceiver`和`Service`的是相同的，系统建议尽量显式调用启动服务，在`lollipop`及其之后的版本，禁止使用隐式调用的方式启动Service，官方描述如下：

>The Context.bindService() method now requires an explicit Intent, and throws an exception if given an implicit intent. To ensure your app is secure, use an explicit intent when starting or binding your Service, and do not declare intent filters for the service.

- 使用隐式Intent调用的方式启动一个Activity时，需要先进行一次判断，看系统中是否存在匹配的Activity，有以下方法

```
Intent intent = new Intent();
...     // 设置Intent



if (intent.resolveActivity(getPackageManager()) != null) {      
    try {  
        startActivity(intent);  
    } catch (ActivityNotFoundException e) { 
          e.printStackTrace();
    }  
}else {
    ...                               
}

//Or

if (getPackageManager().resolveActivity(intent,  PackageManager.MATCH_DEFAULT_ONLY) != null) {  
    try {  
        startActivity(intent);  
    } catch (ActivityNotFoundException e) { 
          e.printStackTrace();
    }  
}else {
    ...                               
}
```

**值得注意的是，在上述第二种方式中，我们要使用`PackageManager.MATCH_DEFAULT_ONLY`这个标记位，该标记位的意思为仅匹配在`intent-filter`中声明了`<category android:name="android.intent.category.DEFAULT"/>`的`Activity`。**

以上两种方法在一般情况下均可行，下面介绍一种情况

在应用A中，有

```
<activity
    android:name=".AimActivity"
    android:exported="false">

    <intent-filter>

        <action android:name="me.rorschach.action.a"/>
        <action android:name="me.rorschach.action.b"/>

        <category android:name="me.rorschach.category.c"/>
        <category android:name="me.rorschach.category.d"/>
        <category android:name="android.intent.category.DEFAULT" />

        <data
            android:scheme="myScheme"
            android:host="rorschach.github.io"
            android:port="8888"
            android:path="/activityDemo"
            android:mimeType="image/*"/>
    </intent-filter>

</activity>
```

在应用B中，启动一个隐式Intent

```
Intent intent = new Intent("me.rorschach.action.a");
intent.addCategory("me.rorschach.category.c");
intent.addCategory("me.rorschach.category.d");
intent.setDataAndType(
        Uri.parse("myScheme://rorschach.github.io:8888/activityDemo"),
        "image/*");
if (intent.resolveActivity(getPackageManager()) != null) {
    startActivity(intent);
}
```

结果

```
AndroidRuntime: FATAL EXCEPTION: main
   Process: me.rorschach.activitytest, PID: 27542
   java.lang.SecurityException: Permission Denial: 
        starting Intent { 
            act=me.rorschach.action.a 
            cat=[me.rorschach.category.c,me.rorschach.category.d] 
            dat=myScheme://rorschach.github.io:8888/activityDemo 
            typ=image/* 
            cmp=me.rorschach.activitydemo/.AimActivity } 
        from ProcessRecord{3207dd03 27542:me.rorschach.activitytest/u0a76} (pid=27542, uid=10076) not exported from uid 10013
```

因为设置了`exported="false"`之后，该Activity无法被其他应用访问，仅在本应用中可见，因此无法被启动。
为保证启动的是可以被其他应用访问的Activity，将上述代码做以下修改:

```
Intent intent = new Intent();
...     //set intent
ActivityInfo activityInfo = intent.resolveActivityInfo(getPackageManager(), intent.getFlags());

if (activityInfo.exported) {
    startActivity(intent);
}
```

