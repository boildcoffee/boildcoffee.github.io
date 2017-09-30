---
title: 消息队列实现
date: 2017-09-30 14:34:49
categories: android
tags: 
    android
    java
---
最近看了许多消息队列的资料,也就试着自己实现了下,有问题欢迎一起探讨
### 设计说明
![QQ截图20160929165355.png](http://upload-images.jianshu.io/upload_images/2898841-491ac8e2d5d77c6f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
大体上的设计是由一条线程1执行从等待列表中获取任务插入任务队列再由线程池中的线程从任务队列中取出任务去执行.   
添加一条线程1主要是防止在执行耗时的任务时阻塞主线程.当执行耗时任务时,添加的任务的操作快于取出任务的操作,   
当任务队列长度达到最大值时,线程1将被阻塞,等待线程2,3...从任务队列取出任务执行。

### 实现
#### 1.编写任务模型
```java
public abstract class TaskBase implements Serializable,Comparable{
    public long taskId;
    public int priority; //任务优先级,约大优先级越高

    public TaskBase(int priority){
        this.priority = priority;
    }

    //任务被执行时调用
    public abstract void taskExc();

    @Override
    public int compareTo(Object o) {
        TaskBase taskBase = (TaskBase) o;
        if (priority > taskBase.priority){
            return -1;
        }else if (priority < taskBase.priority){
            return 1;
        }
        return 0;
    }
}
```

#### 2.编写任务队列
```java
public class TaskQueue {
    private final int QUEUE_SIZE = 20; //任务队列大小
    private final List<TaskBase> mWaitList = new ArrayList<TaskBase>();
    private final PriorityBlockingQueue<TaskBase> mTaskQueue =  new PriorityBlockingQueue(QUEUE_SIZE);

    private ExecutorService mThreadPool;
    private ExecutorService mAddThread;
    private final int mThreadSize;

    public TaskQueue(int threadSize){
        mThreadPool = Executors.newFixedThreadPool(threadSize);
        mAddThread = Executors.newSingleThreadExecutor();
        mThreadSize = threadSize;
    }

    public void start(){
        for (int i=0; i<mThreadSize; i++){
            mThreadPool.execute(new TaskDispatcher(mTaskQueue));
        }
        mAddThread.execute(new TaskAddDispatcher(mWaitList,mTaskQueue));
    }

    public void stop(){
        if (mThreadPool != null && !mThreadPool.isShutdown()){
            mThreadPool.shutdown();
        }
    }


    public boolean addTask(TaskBase taskBase){
        synchronized (mWaitList){
            return mWaitList.add(taskBase);
        }
    }

    public boolean addTask(List<TaskBase> taskBases){
        synchronized (mWaitList){
            return mWaitList.addAll(taskBases);
        }
    }

    public boolean retry(TaskBase taskBase){
        synchronized (mWaitList){
            if (mWaitList.contains(taskBase)){
                return false;
            }
            return mWaitList.add(taskBase);
        }
    }

    public boolean remove(TaskBase taskBase){
        synchronized (mWaitList){
            return mWaitList.remove(taskBase);
        }
    }

}
```

#### 3.编写添加任务到等待列表线程
```java
public class TaskAddDispatcher extends Thread {
    private List<TaskBase> mWaitList;
    private BlockingQueue<TaskBase> mTaskQueue;

    public TaskAddDispatcher(List<TaskBase> waitList, BlockingQueue<TaskBase> taskQueue) {
        mWaitList = waitList;
        mTaskQueue = taskQueue;
    }

    @Override
    public void run() {
        if (mWaitList == null) return;
        while (true) {
            if (!mWaitList.isEmpty() && mTaskQueue != null) {
                synchronized (mWaitList) {
                    mTaskQueue.add(mWaitList.remove(0));
                }
            } else {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }

        }
    }
}

```

#### 4.编写任务工作线程
```java
public class TaskDispatcher extends Thread{
    private BlockingQueue<TaskBase> mTaskQueue;

    public TaskDispatcher(BlockingQueue<TaskBase> taskQueue){
        mTaskQueue = taskQueue;
    }


    @Override
    public void run() {
        while (true){
            try {
                if (mTaskQueue != null){
                    TaskBase task = mTaskQueue.take();
                    task.taskExc();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                continue;
            }
        }
    }
}
```

#### 5.编写管理类
```java
public class TaskManager {
    public final int THREAD_SIZE = 3;

    private static TaskManager mTaskManager;
    private TaskQueue mTaskQueue;

    private TaskManager(){
        mTaskQueue = new TaskQueue(THREAD_SIZE);
    }

    public synchronized static TaskManager getInstance(){
        if (mTaskManager == null){
            mTaskManager = new TaskManager();
        }
        return mTaskManager;
    }

    public boolean addTask(TaskBase taskBase){
        return mTaskQueue.addTask(taskBase);
    }

    public boolean addTask(List<TaskBase> taskBases){
        return mTaskQueue.addTask(taskBases);
    }

    public boolean retryTask(TaskBase taskBase){
        return mTaskQueue.retry(taskBase);
    }

    public boolean cancelTask(TaskBase taskBase){
        return mTaskQueue.remove(taskBase);
    }

    public void start(){
        mTaskQueue.start();
    }

    public void stop(){
        mTaskQueue.stop();
    }

}

```

### 使用

#### 1.继承TaskBase实现taskExc()方法
```java
public class TestBean extends TaskBase{
    public TestBean(int priority) {
        super(priority);
    }

    public TestBean(){
        super(0);
    }

    @Override
    public void taskExc() {
        Log.d(TestBean.class.getName(), "tasksuccess,priority==>" + priority);
        excDelayTask();
    }

    private void excDelayTask(){
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

#### 2.启动所有工作线程
```java
    TaskManager.getInstance().start()
```

#### 3.添加任务
```java
    TaskManager.getInstance().add(new TestBean());
```

#### github Demo地址:https://github.com/aii1991/QueueDemo.git