---
layout:     post
title:      "Android中的多线程下载和断点续传下载"
subtitle:   " \"HttpURLConnection实现多线程及断点续传下载\""
date:       2015-11-25 17:54:30
author:     "Rorschach"
header-img: "img/post-bg-2015.jpg"
tags:
    - android
    - http
---

> “Yeah It's on. ”


## 前言

1.根据url得到连接对象：

    HttpURLConnection conn = (HttpURLConnection) url.openConnection();

2.连接网络：

    conn.setReadTimeout(6000);
    conn.setConnectTimeout(8000);
    conn.setRequestMethod("GET");conn.connect();

3.获取文件大小：

    int length = conn.getContentLength();

4.根据url得到文件名：

    String fileName = url.substring(this.downloadUrl .lastIndexOf('/') + 1);  

5.根据文件名创建文件：

    RandomAccessFile raf = new RandomAccessFile(fileName, "rw");

6.将新建文件的大小设置为获取到的文件大小

    raf.setLength(length);

6.获取下载记录

    List<DownInfo> lists = myDbHelper.queryToList(url);

7.若不存在下载记录，则新建线程进行下载，计算每个线程负责的区域

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

8.如存在下载记录，则根据数据库中保存的记录进行断点下载

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

10.获取输入流

    InputStream is = conn.getInputStream();

11.在每个线程中都获取要写入数据的文件对象，防止线程间资源共享的问题

    RandomAccessFile raf = new RandomAccessFile(fileName, "rw");

12.将文件写入的位置移到startPosition
    
    raf.seek(start);

13.将输入流中的数据写入文件

    byte[] buff = new byte[1024];
    int nRead = 0;
    while ((nRead = is.read(buff)) != -1) {
        raf.write(buff, 0, nRead);
    }


未完待续.....