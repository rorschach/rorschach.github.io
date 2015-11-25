---
layout:     post
title:      "Android中的多线程下载和断点续传下载"
subtitle:   " \"HttpURLConnection实现多线程及断点续传下载\""
date:       2015-11-25 17:54:30
author:     "Rorschach"
header-img: "img/post-multi-download.jpg"
tags:
    - android
    - http
---

>

首先我们来实现下载功能，这里只需要简单的使用HttpUrlConection，因为HttpUrlConection中包含一个可以用于获取文件长度的方法 ： `conn.getContentLength()`，相对于HttpClient而言更加方便。要下载一个文件，只需根据Url得到`HttpUrlConnection`对象conn，然后使用 `conn.getInputStream()`方法即可得到一个InputStreasm对象，然后将InputStream中的数据取出即可，并写入一个文件即可。

关于多线程下载，其核心思想为将要下载的文件分割为几个部分，每一个部分由一个线程单独完成下载，其重点在于如何将文件的字节流进行分割。我们首先要获得文件的大小，这就是为什么采用HttpURlConnection的原因。通过计算，得出每一个线程应负责下载的大小，即该线程对应下载文件数据s的起点位置和终点位置。然后在线程中设置`conn.setRequestProperty("Range", "bytes=" + start + "-" + end)`以指定连接的属性。
最后使用RandomAccessFile类进行数据的写入，该类可以方便的随机访问文件内的任何位置，是多线程下载的核心类。

关于断点续传，其核心思想为下载中断时保存下载的进度到数据库，在恢复下载的时候在数据库中读取进度，然后继续进行下载，以此达到断点续传的目的。

关于下载完成后，更新UI的问题，可以使用Handler和Message更新;也可以使用AsyncTask，在`doInBackground()`中进行耗时的任务，在`postExecute()`中更新UI;也可以使用广播，在任务完成后发送广播，接收到后更新UI，同时还有许多好用的第三方库，如`EventBus`和`AndroidEventBus`。在这里我使用的是AsyncTask。

#####大致流程如下：

1.根据url得到连接HttpURLConnection对象：

    HttpURLConnection conn = (HttpURLConnection) url.openConnection();

2.连接网络及进行必要的设置：

    conn.setReadTimeout(6000);
    conn.setConnectTimeout(8000);
    conn.setRequestMethod("GET");conn.connect();

3.获取要下载的文件大小：

    int length = conn.getContentLength();

4.根据url得到文件名：

    String fileName = url.substring(this.downloadUrl.lastIndexOf('/') + 1);  

5.根据得到的文件名创建RandomAccessFile文件：

    RandomAccessFile raf = new RandomAccessFile(fileName, "rw");

6.将新建文件的大小设置为获取到的文件大小：

    raf.setLength(length);

6.获取数据库中的关于下载信息的记录：

    List<DownInfo> lists = myDbHelper.queryToList(url);

7.若不存在下载记录，则新建线程进行下载，计算每个线程下载的大小：

    if( lists.isEmpty() ){
        int blocks = 4; //由输入决定
        int blockSize = length / blocks;
        int start;
        int end;
        for (int i = 1; i <= blocks; i++) {
            start = (i - 1) * blockSize;
            if (i == blocks) {
                end = length - 1;
            } else {
                end = i * blockSize - 1;
            }
            new DownLoadTask(i, start, end).execute();
        }
    }

8.若存在下载记录，则根据数据库中保存的记录进行断点下载：

    if (!lists.isEmpty()) {
        int blocks = lists.size();
        int blockSize = length / blocks;
        int id;
        int start;
        int end;
        for (DownInfo info : lists) {
            id = info.getThreadId();
            start = info.getStartPos();
            end =  info.getEndPos();
            new DownLoadTask(id, start, end).execute();
        }
    }

9.下载线程中新建网络连接：

    HttpURLConnection conn = (HttpURLConnection) url.openConnection();
    conn.setReadTimeout(10000);
    conn.setConnectTimeout(12000);
    conn.setRequestMethod("GET");
    conn.setRequestProperty("Range", "bytes=" + start + "-" + end);  //设定该线程下载的区域

10.从HttpURLConnection对象中获取输入流：

    InputStream is = conn.getInputStream();

11.在每个线程中都获取要写入数据的文件对象，避免出现线程间资源共享的问题：

    RandomAccessFile raf = new RandomAccessFile(fileName, "rw");

12.将文件要写入数据的位置设置为startPosition，以实现各个线程分别写入数据的功能：
    
    raf.seek(start);

13.将输入流中的数据写入文件：

    byte[] buff = new byte[1024];
    int nRead = 0;
    while ((nRead = is.read(buff)) != -1) {
        raf.write(buff, 0, nRead);
    }

14.数据写入后更新UI

    class DownLoadTask extends AsyncTask<Void, Void, Void> {
        private int id;
        private int start;
        private int end;
        public DownLoadTask(int id, int start, int end) {
            this.id = id;
            this.start = start;
            this.end = end;
        }
        @Override
        protected Void doInBackground(Void... params) {
            //在这里进行网络及文件操作，避免阻塞主线程
            return null;
        }
        @Override
        protected void onPostExecute(Void aVoid) {
            //在这里更新UI
        }
    }



未完待续.....