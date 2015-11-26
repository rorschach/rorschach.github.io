---
layout:     post
title:      "Android中的多线程下载和断点续传下载"
subtitle:   "HttpURLConnection实现"
date:       2015-11-25 17:54:30
author:     "Rorschach"
header-img: "img/post-multi-download.jpg"
tags:
    - java
    - android
    - http
---


####本文参考[http://blog.csdn.net/zhaokaiqiang1992](http://blog.csdn.net/zhaokaiqiang1992)
####转载请注明
[http://rorschach.github.io/2015/11/25/android-multi-breakpoint-download/](http://rorschach.github.io/2015/11/25/android-multi-breakpoint-download/)


要在Android中实现多线程下载和断点续传下载，不需要其他框架，只需要使用Java和Android的API即可完成。可以简单的分为3个步骤：下载、多线程下载、断点续传。

首先我们来实现下载功能，这里只需要简单的使用HttpUrlConection，因为HttpUrlConection中包含一个可以用于获取文件长度的方法 ： `conn.getContentLength()`，相对于HttpClient而言更加方便。要下载一个文件，只需根据Url得到`HttpUrlConnection`对象conn，然后使用 `conn.getInputStream()`方法即可得到一个InputStreasm对象，然后将InputStream中的数据取出即可，并写入一个文件即可。

关于多线程下载，其核心思想为将要下载的文件分割为几个部分，每一个部分由一个线程单独完成下载，其重点在于如何将文件的字节流进行分割。我们首先要获得文件的大小，这就是为什么采用HttpURlConnection的原因。通过计算，得出每一个线程应负责下载的大小，即该线程对应下载文件数据s的起点位置和终点位置。然后在线程中设置`conn.setRequestProperty("Range", "bytes=" + start + "-" + end)`以指定连接的属性。
最后使用RandomAccessFile类进行数据的写入，该类可以方便的随机访问文件内的任何位置，是多线程下载的核心类。

关于断点续传，其核心思想为下载中断时保存下载的进度到数据库，在恢复下载的时候在数据库中读取进度，然后继续进行下载，以此达到断点续传的目的。

关于下载完成后，更新UI的问题，可以使用Handler和Message更新;也可以使用AsyncTask，在`doInBackground()`中进行耗时的任务，在`postExecute()`中更新UI;也可以使用广播，在任务完成后发送广播，接收到后更新UI，同时还有许多好用的第三方库，如`EventBus`和`AndroidEventBus`。在这里我使用的是AsyncTask。



#####大致流程如下：####


[跳过流程，直接看代码 ](#build) 


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

<p id = "build"></p>
---
代码如下，最终代码放在[github](https://github.com/rorschach/MultiDownloader)

抽取下载信息的实体类：

    public class DownLoadInfo {

        int threadId;
        int startPos;
        int endPos;
        int completedSize;
        String url;
        String fileName;

        public DownLoadInfo() { }

        public DownLoadInfo(int threadId, int startPos, int endPos, int completedSize, String url, String fileName) {
            this.threadId = threadId;
            this.startPos = startPos;
            this.endPos = endPos;
            this.completedSize = completedSize;
            this.url = url;
            this.fileName = fileName;
    }

        @Override
        public String toString() {
            return "DownLoadInfo [" + "threadId : " + threadId
                    + " startPos : " + startPos
                    + " endPos : " + endPos
                    + " completedSize : " + completedSize
                    + " url : " + url
                    + " filename : " + fileName + " ]";
        }

        public int getThreadId() {
            return threadId;
        }
        public void setThreadId(int threadId) {
            this.threadId = threadId;
        }
        public int getStartPos() {
            return startPos;
        }
        public void setStartPos(int startPos) {
            this.startPos = startPos;
        }
        public int getEndPos() {
            return endPos;
        }
        public void setEndPos(int endPos) {
            this.endPos = endPos;
        }
        public int getCompletedSize() {
            return completedSize;
        }
        public void setCompletedSize(int completedSize) {
            this.completedSize = completedSize;
        }
        public String getUrl() {
            return url;
        }
        public void setUrl(String url) {
            this.url = url;
        }
        public String getFileName() {
            return fileName;
        }
        public void setFileName(String fileName) {
            this.fileName = fileName;
        }
    }

封装数据库操作

    public class MyDbHelper extends SQLiteOpenHelper {

        private static final String DB_NAME = "DOWN_LOAD";
        private static final int VERSION = 1;
        public static final String TABLE_NAME = "down_info";

        private static final String ID = "_id";
        private static final String THREAD_ID = "thread_id";
        private static final String START_POS = "start_pos";
        private static final String END_POS = "end_pos";
        private static final String COMPLETED_SIZE = "completed_size";
        private static final String URL = "url";
        private static final String FILE_NAME = "file_name";

        private static MyDbHelper sInstance;

        public MyDbHelper(Context context) {
            super(context, DB_NAME, null, VERSION);
        }


        //设置为单例,防止多次创建Helper实例产生的资源消耗，
        // 加锁的目的为避免多个数据库操作同时进行，产生冲突
        public static synchronized MyDbHelper getInstance(Context context) {
            if (sInstance == null) {
                sInstance = new MyDbHelper(context.getApplicationContext());
            }
            return sInstance;
        }

        private static final String CREATE_DOWN_INFO =
                "create table if not exists " + TABLE_NAME
                        * "( " + ID + " integer primary key autoincrement, "
                        * THREAD_ID + " integer not null, "
                        * START_POS + " integer not null, "
                        * END_POS + " integer not null, "
                        * COMPLETED_SIZE + " integer not null, "
                        * URL + " string not null, "
                        * FILE_NAME + " string not null )";

        private static final String DROP_TABLE = "drop table " + TABLE_NAME;

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL(CREATE_DOWN_INFO);
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
            db.execSQL(DROP_TABLE);
            db.execSQL(CREATE_DOWN_INFO);
        }

        public void insertInfo(DownLoadInfo info) {
            SQLiteDatabase db = this.getWritableDatabase();
            ContentValues cv = new ContentValues();
            cv.put(THREAD_ID, info.getThreadId());
            cv.put(START_POS, info.getStartPos());
            cv.put(END_POS, info.getEndPos());
            cv.put(COMPLETED_SIZE, info.getCompletedSize());
            cv.put(URL, info.getUrl());
            cv.put(FILE_NAME, info.getFileName());
            db.insert(TABLE_NAME, null, cv);
        }

        public void deleteInfo(DownLoadInfo info) {
            SQLiteDatabase db = this.getReadableDatabase();
            db.delete(TABLE_NAME,
                    THREAD_ID + "=? and " + URL + "=?",
                    new String[]{String.valueOf(info.getThreadId()), info.getUrl()});
        }

        public void updateInfo(DownLoadInfo info) {
            SQLiteDatabase db = this.getWritableDatabase();
            ContentValues cv = new ContentValues();
            cv.put(THREAD_ID, info.getThreadId());
            cv.put(START_POS, info.getStartPos());
            cv.put(END_POS, info.getEndPos());
            cv.put(COMPLETED_SIZE, info.getCompletedSize());
            cv.put(URL, info.getUrl());
            cv.put(FILE_NAME, info.getFileName());
            db.update(TABLE_NAME, cv,
                    THREAD_ID + "=? and " + URL + "=?",
                    new String[]{String.valueOf(info.getThreadId()), info.getUrl()});
        }

        public DownLoadInfo queryInfo(int threadId, String url) {
            SQLiteDatabase db = this.getReadableDatabase();
            Cursor cursor = db.rawQuery(
                    "select * from " + TABLE_NAME + " where "
                            + THREAD_ID + "=? and " + URL + "=?",
                    new String[]{String.valueOf(threadId), url});
            cursor.moveToFirst();
            DownLoadInfo info = new DownLoadInfo();
            info.setThreadId(threadId);
            info.setStartPos(cursor.getInt(cursor.getColumnIndex(START_POS)));
            info.setEndPos(cursor.getInt(cursor.getColumnIndex(END_POS)));
            info.setCompletedSize(cursor.getInt(cursor.getColumnIndex(COMPLETED_SIZE)));
            info.setUrl(url);
            info.setFileName(cursor.getString(cursor.getColumnIndex(FILE_NAME)));
            cursor.close();
            return info;
        }

        public List<DownLoadInfo> getListByUrl(String url) {
            List<DownLoadInfo> list = new ArrayList<>();
            DownLoadInfo info;
            SQLiteDatabase db = this.getReadableDatabase();
            Cursor cursor = db.rawQuery(
                    "select * from " + TABLE_NAME + " where " + URL + "=?",
                    new String[]{ url});
            if(cursor.moveToFirst()){
                do {
                    info = new DownLoadInfo();
                    info.setThreadId(cursor.getInt(cursor.getColumnIndex(THREAD_ID)));
                    info.setStartPos(cursor.getInt(cursor.getColumnIndex(START_POS)));
                    info.setEndPos(cursor.getInt(cursor.getColumnIndex(END_POS)));
                    info.setCompletedSize(cursor.getInt(cursor.getColumnIndex(COMPLETED_SIZE)));
                    info.setUrl(url);
                    info.setFileName(cursor.getString(cursor.getColumnIndex(FILE_NAME)));
                    list.add(info);
                }while (cursor.moveToNext());
            }
            cursor.close();
            return list;
        }

        public void closeDb() {
            sInstance.close();
        }
    }

网络操作工具类

    public class DownLoadUtils {

    private String url;
    private String path;
    private String fileName;
    private int threadCount;
    private Handler mHandler;

    private MyDbHelper mMyDbHelper;
    private List<DownLoadInfo> mInfoList = new ArrayList<>();

    private int fileSize = 0;
    private int allCompletedSize = 0;

    private int downloadState = STATE_READY;

    private static final int STATE_READY = 0;
    private static final int STATE_DOWNLOADING = 1;
    private static final int STATE_PAUSE = 2;
    private static final int STATE_COMPLETED = 3;
    private static final int STATE_FAILED = 4;
    private static final int STATE_DELETED = 5;

    private ExecutorService es;

    public DownLoadUtils(Context context, String url, String path, String fileName, int threadCount, Handler handler) {
        this.url = url;
        this.path = path;
        this.fileName = fileName;
        this.threadCount = threadCount;
        this.mHandler = handler;
        mMyDbHelper = MyDbHelper.getInstance(context);
    }

    public void ready() {
        Log.d("TAG", "downloadState is ready");

        es = Executors.newFixedThreadPool(threadCount);
        downloadState = STATE_READY;

        mInfoList = mMyDbHelper.getListByUrl(url);
        Log.d("TAG", "size of list : " + mInfoList.size());

        if (mInfoList.isEmpty() || mInfoList == null) {
            Log.d("TAG", "case 1");

            init();
        } else {
            File file = new File(path + fileName);
            if (!file.exists()) {
                Log.d("TAG", "case 2");
                mMyDbHelper.deleteUrl(url);
                init();
            } else {
                Log.d("TAG", "case 3");
                fileSize = mInfoList.get(mInfoList.size() - 1).getEndPos();
                for (DownLoadInfo info : mInfoList) {
                    allCompletedSize += info.getCompletedSize() + info.getState();
                    Log.d("TAG", "allCompletedSize : " + allCompletedSize);
                }
            }
        }
    }

    public void start() {
        Log.d("TAG", "DownLoadUtils is start");
        Log.d("TAG", "STATE_DOWNLOADING : " + downloadState);
        if (mInfoList != null) {
            if (downloadState != STATE_DOWNLOADING) {
                downloadState = STATE_DOWNLOADING;
            }
            Log.d("TAG", "STATE_DOWNLOADING : " + downloadState);
            for (DownLoadInfo info : mInfoList) {
                es.isShutdown();
                es.submit(new DownLoadRunnable(info));
            }
        }
    }

    public void pause() {
        Log.d("TAG", "DownLoadUtils is pause");
        downloadState = STATE_PAUSE;
        mMyDbHelper.closeDb();
        es.shutdown();
    }

    public void completed() {
        Log.d("TAG", "DownLoadUtils is completed");
        mMyDbHelper.deleteUrl(url);
        mMyDbHelper.closeDb();
        es.shutdown();
    }

    public void failed() {
        Log.d("TAG", "DownLoadUtils is failed");
        downloadState = STATE_FAILED;
        mMyDbHelper.closeDb();
        es.shutdown();
    }

    public void delete() {
        Log.d("TAG", "DownLoadUtils is delete");
        downloadState = STATE_DELETED;
        File file = new File(path + fileName);
        if (file.exists()) {
            file.delete();
        }
    }

    public int getFileSize() {
        return fileSize;
    }

    public int getAllCompletedSize() {
        return allCompletedSize;
    }

    private void init() {
        mMyDbHelper.deleteUrl(url);
        Log.d("TAG", "init start");
        HttpURLConnection conn = null;
        RandomAccessFile raf = null;
        InputStream is = null;

        try {
            File filePath = new File(path);
            if (!filePath.exists()) {
                filePath.mkdir();
            }

            raf = new RandomAccessFile(path + fileName, "rwd");

            URL urlToConnection = new URL(url);
            conn = (HttpURLConnection) urlToConnection.openConnection();
            conn.setConnectTimeout(10000);
            conn.setReadTimeout(8000);
            conn.setRequestMethod("GET");

            fileSize = conn.getContentLength();
            Log.d("TAG", "fileSize : " + fileSize);
            raf.setLength(fileSize);

            Log.d("TAG", "init half");

            int blockSize = fileSize / threadCount;
            int start;
            int end;
            DownLoadInfo info;

            for (int i = 1; i <= threadCount; i++) {

                start = (i - 1) * blockSize;
                if (i == threadCount) {
                    end = fileSize - 1;
                } else {
                    end = i * blockSize - 1;
                }

                info = new DownLoadInfo();
                info.setThreadId(i);
                info.setStartPos(start);
                info.setEndPos(end);
                info.setCompletedSize(0);
                info.setUrl(url);
                info.setPath(path);
                info.setFileName(fileName);
                info.setState(STATE_READY);
                mInfoList.add(info);
                Log.d("TAG", "size of list : " + mInfoList.size());
            }
            mMyDbHelper.insertList(mInfoList);

        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                Log.d("TAG", "init end");
                if (is != null) {
                    is.close();
                }
                if (raf != null) {
                    raf.close();
                }
                if (conn != null) {
                    conn.disconnect();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private class DownLoadRunnable implements Runnable {

        private DownLoadInfo mInfo;
        private int currentCompletedSize;

        public DownLoadRunnable(DownLoadInfo mInfo) {
            this.mInfo = mInfo;
        }

        @Override
        public void run() {
            Log.d("TAG", mInfo.toString() + "is doing...");
            HttpURLConnection conn = null;
            RandomAccessFile raf = null;
            InputStream is = null;
            currentCompletedSize = mInfo.getCompletedSize();
            int start = mInfo.getStartPos();
            int end = mInfo.getEndPos();

            try {
                File filePath = new File(mInfo.getPath());
                if (!filePath.exists()) {
                    filePath.mkdir();
                }

                raf = new RandomAccessFile(
                        mInfo.getPath() + mInfo.getFileName(), "rwd");
                raf.seek(start);

                URL url = new URL(mInfo.getUrl());
                conn = (HttpURLConnection) url.openConnection();
                conn.setConnectTimeout(10000);
                conn.setReadTimeout(8000);
                conn.setRequestMethod("GET");
                conn.setRequestProperty("Range",
                        "bytes=" + start + "-" + end);

                is = conn.getInputStream();

                byte[] buff = new byte[1024];
                int nRead;

                while ((nRead = is.read(buff)) != -1) {
                    raf.write(buff, 0, nRead);
                    currentCompletedSize += nRead;

                    Message message = new Message();
                    message.what = mInfo.getThreadId();
                    message.arg1 = nRead;
                    message.obj = mInfo.getUrl();
                    mHandler.sendMessage(message);

                    Log.d("TAG", "downloadState : " + downloadState);
                    if (downloadState != STATE_DOWNLOADING
                            || currentCompletedSize >= end) {
                        Log.d("TAG", "threadId : " + mInfo.getThreadId()
                                + ", currentCompletedSize : " + currentCompletedSize);
                        mInfo.setCompletedSize(currentCompletedSize);
                        mInfo.setStartPos(start + currentCompletedSize + 1);
                        mInfo.setState(STATE_COMPLETED);
                        mMyDbHelper.updateInfo(mInfo);
                        break;
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
                mInfo.setState(STATE_PAUSE);
                mMyDbHelper.updateInfo(mInfo);

            } finally {
                try {
                    if (is != null) {
                        is.close();
                    }
                    if (raf != null) {
                        raf.close();
                    }
                    if (conn != null) {
                        conn.disconnect();
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}


封装工具类

    public class MultiDownloader {

    private DownLoadUtils mDownLoadUtils;
    private OnMultiDownLoadListener mListener;

    private int fileSize = 0;
    private int completedSize = 0;

    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            synchronized (this) {
                completedSize += msg.arg1;
            }

            if (mListener != null) {
                mListener.updateProgress(completedSize);
            }

            if (completedSize >= fileSize) {
                mDownLoadUtils.completed();
                if (mListener != null) {
                    mListener.DownLoadCompleted();
                } else {
                    return;
                }
            } else {
                return;
            }
        }
    };

    public MultiDownloader(Context context, String url, String path, String fileName, int threadCount) {
        mDownLoadUtils = new DownLoadUtils(context, url, path, fileName, threadCount, mHandler);
    }

    public void start() {
        Log.d("TAG", "MultiDownloader is start");
        new AsyncTask<Void, Void, Void>() {

            @Override
            protected Void doInBackground(Void... params) {
                Log.d("TAG", "MultiDownloader is doInBackground");
                mDownLoadUtils.ready();

                return null;
            }

            @Override
            protected void onPostExecute(Void aVoid) {
                Log.d("TAG", "MultiDownloader is onPostExecute");
                fileSize = mDownLoadUtils.getFileSize();
                completedSize = mDownLoadUtils.getAllCompletedSize();
                Log.d("TAG", "fileSize : " + fileSize + ", completedSize : "  +completedSize);
                if (mListener != null) {
                    mListener.startDownLoad(fileSize);
                }
                mDownLoadUtils.start();
            }
        }.execute();
    }

    public void pause() {
        mDownLoadUtils.pause();
    }

    public void delete() {
        mDownLoadUtils.delete();
    }

    public void restart() {
        delete();
        start();
    }

    public void setOnMultiDownLoadListener(OnMultiDownLoadListener mListener) {
        this.mListener = mListener;
    }

    public interface OnMultiDownLoadListener {
        void startDownLoad(int totalSize);

        void updateProgress(int progress);

        void DownLoadCompleted();
    }
}

例子

     public class MainActivity extends AppCompatActivity {

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.activity_main);

            final Button start = (Button) findViewById(R.id.start);
            final Button pause = (Button) findViewById(R.id.pause);
            final Button delete = (Button) findViewById(R.id.delete);
            final Button restart = (Button) findViewById(R.id.restart);
            final ProgressBar progressBar = (ProgressBar) findViewById(R.id.progress);

            final String url = "http://guo.lu/wp-content/uploads/2015/11/wallhaven-48626.jpg";
            final String path = "/storage/emulated/0/";
            final String filename = "multi_download.jpg";
            final int threadCount = 4;

            final MultiDownloader downloader = new MultiDownloader(this, url, path, filename, threadCount);

            downloader.setOnMultiDownLoadListener(new MultiDownloader.OnMultiDownLoadListener() {
                @Override
                public void startDownLoad(int totalSize) {
                    Log.d("TAG", "totalSize : " + totalSize);
                    progressBar.setMax(totalSize);
                }

                @Override
                public void updateProgress(int progress) {
                    progressBar.setProgress(progress);
                }

                @Override
                public void DownLoadCompleted() {
                    Toast.makeText(MainActivity.this, "download completed", Toast.LENGTH_SHORT).show();
                }
            });

            start.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    downloader.start();
                }
            });

            pause.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    downloader.pause();
                }
            });

            delete.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    downloader.delete();
                    progressBar.setProgress(0);
                }
            });

            restart.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    downloader.restart();
                }
            });
        }
    }


未完待续.....