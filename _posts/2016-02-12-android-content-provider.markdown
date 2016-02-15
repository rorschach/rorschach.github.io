---
layout:     post
title:      "ContentProvider初探"
subtitle:   "读书笔记"
date:       2016-02-11 20:00:00
author:     "Rorschach"
header-img: "img/post-bg-content-provider.jpg"
tags:
    - android
---

## 概述

>Content providers are one of the primary building blocks of Android applications, providing content to applications. 

`ContentProvider`是Android 四大组件之一，用于给其他应用提供数据，适合进程间通信，其底层通过`Binder`实现。

`ContentProvider`主要以表格的形式来组织数据，并且可以包含多个表，对于每个表格都有行和列的层次，这一点类似于数据库中的`Table`。除了表格的形式，`ContentProvider`还支持文件数据，例如图片，视频等。文件数据和表格数据的形式不同，因此处理此类数据时可以在`ContentProvider`中返回句柄，从而让外界访问`ContentProvider`中的文件信息。Android提供的`MediaStore`就是文件类型的`ContentProvider`，具体参考`MediaStore`。另外，虽然`ContentProvider`的底层数据看起来像`SQLite`，但实际上其对底层的数据存储方式没有任何要求，可以是一个文件，甚至可以是一个内存中的对象。

## ContentProvider的访问

一个应用要想访问`ContentProvider`来获得数据，必须通过`ContentResolver`对象。在需要访问数据的应用中获得`ContentResolver`对象，提供数据的应用中负责提供`ContentProvider`对象，获得`ContentResolver`对象之后这两个应用所属的进程间便自动建立了连接。

要使得`ContentProvider`可以被访问，必须在提供数据的应用的`manifest`文件中声明权限，如

```
    <provider
        android:exported="true"
        android:authorities="me.rorschach.contentproviderdemo.provider"
        android:name="me.rorschach.contentproviderdemo.PersonProvider" />
```


`ContentProvider`为持久化存储提供了基本的"CRUD" (create, retrieve, update, and delete) 操作。通过以下代码在`ContextWrapper`的子类，如在`Activity`中得到`ContentResolver`对象


```
ContentResolver cr = getContentResolver();
```


获得`ContentResolver`对象后，可以进行`CRUD`操作：

- insert
```
public final @Nullable Uri insert(
    @NonNull Uri url, 
    @Nullable ContentValues values) {}
```

- delete

```
public final int delete(
    @NonNull Uri url, 
    @Nullable String where,
    @Nullable String[] selectionArgs) {}
```

- update

```
public final int update(
    @NonNull Uri uri, 
    @Nullable ContentValues values,
    @Nullable String where, 
    @Nullable String[] selectionArgs) {}
```

- query

```
public final @Nullable Cursor query(
    @NonNull Uri uri, 
    @Nullable String[] projection,
    @Nullable String selection, 
    @Nullable String[] selectionArgs,
    @Nullable String sortOrder) {}
```

可以注意到上述方法中都需要一个参数`Uri`，那么它是什么呢？

>A content URI is a URI that identifies data in a provider. Content URIs include the symbolic name of the entire provider (its authority) and a name that points to a table (a path). When you call a client method to access a table in a provider, the content URI for the table is one of the arguments.

形如
```
content://me.rorschach.contentproviderdemo.provider/insert
```

可以发现其形式和URL很相似,如
```
https://github.com/rorschach
```


对于上述URL，可分为三部分：

1. https:// : 协议部分，此部分固定
2. github.com : 域名部分，只要访问固定的网站，此部分总是固定的
3. rorschach : 资源部分，访问者需要访问不同的资源时，此部分改变

相应的，对于上述Uri，同样可以分为三个部分

1. content:// : 此部分为`ContentProvider`的协议，固定写法
2. me.rorschach.contentproviderdemo.provider : 此部分为`ContentProvider`的`authority`，系统通过这个部分来找到相应的`ContentProvider`
3. insert : 资源部分（数据部分），访问者需要访问不同的资源时，此部分改变

存在以下方法，将一个字符串构造成一个`Uri`对象

```
Uri uri = Uri.parse("content://me.rorschach.contentproviderdemo.provider/insert");
```

## ContentProvider的创建

### 基本步骤

创建`ContentProvider`分为两步

1. 创建一个`ContentProvider`的子类，实现`onCreate()`, `query()`,`insert()`,`update()`,`delete()`和`getType()`方法
2. 在`manifest`文件中注册该`ContentProvider`，声明其`authorities`

### 自定义ContentProvider使用实例

在此处我们创建一个App，用于提供数据，数据的格式定为`SQLite`。

1. 产生一个类，用于提供列名
2. 产生一个`SQLiteOpenHelper`的子类
3. 产生相应的`bean`类
4. 产生一个`ContentProvider`的子类
5. 在客户端中获得`ContentResolver`，执行相应的`CRUD`操作

服务端
```
public final class PersonContract {

    public PersonContract() {
    }

    public static abstract class PersonEntry implements BaseColumns {
        public static final String TABLE_NAME = "person";
        public static final String COLUMN_NAME_ID = "id";
        public static final String COLUMN_NAME_NAME = "name";
        public static final String COLUMN_NAME_AGE = "age";
    }
}

```

```
public class MyDbHelper extends SQLiteOpenHelper {

    public static final String DB_NAME = "person_provider.db";
    public static final int DB_VERSION = 1;
    public static final String TABLE_NAME = PersonContract.PersonEntry.TABLE_NAME;

    public static final String ID = PersonContract.PersonEntry.COLUMN_NAME_ID;
    public static final String NAME = PersonContract.PersonEntry.COLUMN_NAME_NAME;
    public static final String AGE = PersonContract.PersonEntry.COLUMN_NAME_AGE;

    private static final String CREATE_PERSON_TABLE = "create table if not exists " + TABLE_NAME
            + "( " + ID + " integer primary key, "
            + NAME + " string, "
            + AGE + " integer )";

    private MyDbHelper(Context context) {
        super(context, DB_NAME, null, DB_VERSION);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL(CREATE_PERSON_TABLE);
    }
}
```


```
public class Person {

    private int id;
    private String name;
    private int age;

    public Person(int id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

```
public class PersonProvider extends ContentProvider {

    public static final String AUTHORITY = "me.rorschach.contentproviderdemo.provider";

    private MyDbHelper mDbHelper;

    private SQLiteDatabase mDb;

    private Context mContext;

    private static final String TAG = "PersonProvider";

    private static UriMatcher sMatcher;

    private static final int INSERT = 1;
    private static final int DELETE = 2;
    private static final int UPDATE = 4;
    private static final int QUERY = 8;

    static {
        sMatcher = new UriMatcher(UriMatcher.NO_MATCH);
        sMatcher.addURI(AUTHORITY, "insert", INSERT);
        sMatcher.addURI(AUTHORITY, "delete", DELETE);
        sMatcher.addURI(AUTHORITY, "update", UPDATE);
        sMatcher.addURI(AUTHORITY, "query", QUERY);
    }

    @Override
    public boolean onCreate() {
        mContext = getContext();
        mDbHelper = MyDbHelper.getInstance(mContext);
        Log.d(TAG, "onCreate");
        return true;
    }

    @Nullable
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {

        Log.d(TAG, "query:" + uri.toString());
        if (sMatcher.match(uri) == QUERY) {
            mDb = mDbHelper.getReadableDatabase();
            return mDb.query(
                    MyDbHelper.TABLE_NAME,
                    projection,
                    selection,
                    selectionArgs,
                    null, null, null);
        } else {
            throw new IllegalArgumentException("uri not matched!");
        }
    }

    @Nullable
    @Override
    public String getType(Uri uri) {
        Log.d(TAG, "getType");
        return null;
    }

    @Nullable
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        Log.d(TAG, "insert:" + uri.toString());
        if (sMatcher.match(uri) == INSERT) {
            mDb = mDbHelper.getWritableDatabase();
            mDb.insert(
                    MyDbHelper.TABLE_NAME,
                    null,
                    values);
            mContext.getContentResolver().notifyChange(uri, null);
            return uri;
        } else {
            throw new IllegalArgumentException("uri not matched!");
        }
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        Log.d(TAG, "uri:" + uri.toString());
        if (sMatcher.match(uri) == DELETE) {
            mDb = mDbHelper.getWritableDatabase();
            int count = mDb.delete(
                    MyDbHelper.TABLE_NAME,
                    selection,
                    selectionArgs);
            if (count > 0) {
                mContext.getContentResolver().notifyChange(uri, null);
            }
            return count;
        } else {
            throw new IllegalArgumentException("uri not matched!");
        }
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        Log.d(TAG, "update:" + uri.toString());
        if (sMatcher.match(uri) == UPDATE) {
            mDb = mDbHelper.getWritableDatabase();
            int count = mDb.update(
                    MyDbHelper.TABLE_NAME,
                    values,
                    selection,
                    selectionArgs);
            if (count > 0) {
                mContext.getContentResolver().notifyChange(uri, null);
            }
            return count;
        } else {
            throw new IllegalArgumentException("uri not matched!");
        }
    }
}
```

客户端
```
    private ContentResolver mContentResolver =  getContentResolver();

    private static final String AUTHORITY = "me.rorschach.contentproviderdemo.provider";

    ......

    public void insert(View view) {
        Uri uri = Uri.parse("content://" + AUTHORITY + "/insert");
        ContentValues cv = new ContentValues();
        cv.put("id", 4);
        cv.put("name", "lol");
        cv.put("age", 25);
        mContentResolver.insert(uri, cv);

        mContentResolver.registerContentObserver(
            uri, 
            true, 
            new ContentObserver(new Handler()) {
            @Override
            public void onChange(boolean selfChange) {
                super.onChange(selfChange);
                Toast.makeText(MainActivity.this, "data insert", Toast.LENGTH_SHORT).show();
            }
        });
    }

    public void delete(View view) {
        Uri uri = Uri.parse("content://" + AUTHORITY + "/delete");
        mContentResolver.delete(
            uri, 
            "id=?", 
            new String[]{String.valueOf(2)});
    }

    public void update(View view) {
        Uri uri = Uri.parse("content://" + AUTHORITY + "/update");
        ContentValues cv = new ContentValues();
        cv.put("name", "heiheihei");
        cv.put("age", 33);
        mContentResolver.update(
            uri,
            cv,
            "id=?", 
            new String[]{String.valueOf(3)});
    }

    public void query(View view) {
        Uri uri = Uri.parse("content://" + AUTHORITY + "/query");

        Cursor cursor = mContentResolver.query(
            uri, 
            null, 
            null, 
            null, 
            null);

        StringBuilder sb = new StringBuilder();
        while (cursor.moveToNext()) {
            sb.append(cursor.getInt(cursor.getColumnIndex("id")) + ", "
            + cursor.getString(cursor.getColumnIndex("name")) + ", "
            + cursor.getInt(cursor.getColumnIndex("age")) + ", ");
        }

        cursor.close();
        tv.setText(sb.toString());
    }
```

## 使用ContentProvider读取和添加短信

```
    <uses-permission android:name="android.permission.READ_SMS"/>
    <uses-permission android:name="android.permission.WRITE_SMS"/>

    private void getSms() {
        ContentResolver cr = getContentResolver();
        Uri uri = Uri.parse("content://sms/");
        Cursor cursor = cr.query(uri,
                new String[]{"address", "date", "type", "body"},
                "address=?", new String[]{"12306"}, null);
        MySms sms;
        while (cursor.moveToNext()) {
            sms = new MySms(
                    cursor.getString(0),
                    cursor.getLong(1),
                    cursor.getInt(2),
                    cursor.getString(3));

            mSmsList.add(sms);
            Log.d("TAG", mSmsList.toString());
        }
    }

    private void writeSms() {
        ContentResolver cr = getContentResolver();
        Uri uri = Uri.parse("content://sms/");

        ContentValues cv = new ContentValues();
        cv.put("address", "12306");
        cv.put("date", SystemClock.currentThreadTimeMillis());
        cv.put("type", 1);
        cv.put("body", "您当前欠费1,000,000");
        cr.insert(uri, cv);

        Toast.makeText(this, "write success", Toast.LENGTH_SHORT).show();
    }
```

值得注意的是，上述增加短信的方式在Android4.4后失效，因为Android4.4之后只有系统默认的短信应用才能写短信到数据库中，具体信息见[官方博客](http://android-developers.blogspot.jp/2013/10/getting-your-sms-apps-ready-for-kitkat.html)。

## 使用ContentProvider读取和添加联系人

```
    <uses-permission android:name="android.permission.READ_CONTACTS" />
    <uses-permission android:name="android.permission.WRITE_CONTACTS" />

    public void readContact(View view) {
        contactTv.setText("");
        Cursor contactsCursor = mContentResolver.query(
                raw_contacts,
                new String[]{"contact_id"},
                null, null, null);

        Cursor dataCursor = null;
        StringBuilder sb = new StringBuilder();

        int contactId;

        while (contactsCursor.moveToNext()) {
            contactId = contactsCursor.getInt(0);

            dataCursor = mContentResolver.query(
                    data,
                    new String[]{"mimetype", "data1"},
                    "raw_contact_id=?",
                    new String[]{String.valueOf(contactId)},
                    null);

            while (dataCursor.moveToNext()) {
                sb.append(dataCursor.getString(0) + ", " + dataCursor.getString(1));
            }
        }
        contactsCursor.close();
        dataCursor.close();
        contactTv.setText(sb.toString());
    }

    public void writeContact(View view) {
        Cursor contactsCursor = mContentResolver.query(
                raw_contacts,
                new String[]{"contact_id"},
                null, null, null);
        contactsCursor.moveToLast();
        int maxId = contactsCursor.getInt(0);

        ContentValues cv = new ContentValues();
        cv.put("contact_id", maxId + 1);
        mContentResolver.insert(raw_contacts, cv);

        cv = new ContentValues();
        cv.put("mimetype", "vnd.android.cursor.item/phone_v2");
        cv.put("raw_contact_id", maxId + 1);
        cv.put("data1", "12345678910");
        mContentResolver.insert(data, cv);

        cv = new ContentValues();
        cv.put("mimetype", "vnd.android.cursor.item/name");
        cv.put("raw_contact_id", maxId + 1);
        cv.put("data1", "HuangYujie");
        mContentResolver.insert(data, cv);

        contactsCursor.close();
    }
```