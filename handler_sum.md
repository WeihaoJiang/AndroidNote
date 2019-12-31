---
title: handler学习总结
date: 2018-09-9
tags: [android]
categories: android
---


## 背景以及夙愿

- 1 练习博客排版
- 2 尝试阅读源码，从熟悉的handler开始
- 3 习惯养成

## handler是什么？

Handler主要用于异步消息的处理： 有点类似辅助类，封装了消息投递、消息处理等接口。当发出一个消息之后，首先进入一个消息队列，发送消息的函数即刻返回，而另外一个部分在消息队列中逐一将消息取出，然后对消息进行处理，也就是发送消息和接收消息不是同步的处理。 这种机制通常用来处理相对耗时比较长的操作。

## handler使用场景

在Android系统中出于性能优化考虑，Android的UI操作并不是线程安全的，这意味着如果有多个线程并发操作UI组件，可能导致线程安全问题。为了解决这个问题，Android制定了一条简单的原则，只允许UI线程（亦即主线程）修改Activity中的UI组件。但实际上，有部分UI需要在子线程中控制其修改逻辑，因此子线程需要通过handler通知主线程修改UI，实现线程间通信。


## handler的使用

###  1 post或postDelayed
区别在于一个立即执行，一个延时执行
下面以延时操作为例

直接上代码举个小例子
```
new Handler().postDelayed(new Runnable() {
    @Override
    public void run() {
        goHome();//可做UI操作等等
    }
}, 2000); // 延时2秒

```

###  2 sendMessage 
```
public class MainActivity extends AppCompatActivity {

    private TabLayout mTab;
    private ViewPager mVp;
    private ViewPagerAapter mvpAapter;
    public ArrayList<String> mlist = new ArrayList<>();

    public ArrayList<ImageView> mlistIv = new ArrayList<>();
    private ImageView v1, v2, v3;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mVp = findViewById(R.id.view1);
        mTab = findViewById(R.id.tab);
    }

    @Override
    protected void onResume() {
        super.onResume();

        mlist.add("daxifu");
        mlist.add("erxifu");
        mlist.add("sanxifu");

        //mTab.setTabMode(TabLayout.MODE_FIXED);

        for (int i = 0; i < mlist.size(); i++) {
            mTab.addTab(mTab.newTab().setText(mlist.get(i)));

        }
        v1 = new ImageView(this);
        v2 = new ImageView(this);
        v3 = new ImageView(this);

        v1.setImageResource(R.drawable.app_launch_01);
        v2.setImageResource(R.drawable.app_launch_02);
        v3.setImageResource(R.drawable.app_launch_03);

//        v1.setScaleType(ImageView.ScaleType.FIT_XY);
//        v2.setScaleType(ImageView.ScaleType.FIT_XY);
//        v3.setScaleType(ImageView.ScaleType.FIT_XY);


        mlistIv.add(v1);
        mlistIv.add(v2);
        mlistIv.add(v3);

        mTab.setupWithViewPager(mVp);

        mvpAapter = new ViewPagerAapter(mlist, mlistIv);
        mVp.setAdapter(mvpAapter);

        new Thread(){
            @Override
            public void run() {
                super.run();

                for (int i  = 1; i < 4; i++){
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    Message message = new Message();
                    message.what = i;

                    mhandler.sendMessage(message);
                }


                }


        }.start();


    }

    @SuppressLint("HandlerLeak")
    private Handler mhandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);

            switch (msg.what) {
                case 1:
                    v1.setImageResource(R.drawable.app_fuli_01);

                    break;
                case 2:
                    v2.setImageResource(R.drawable.app_fuli_02);

                    break;
                case 3:
                    v3.setImageResource(R.drawable.app_fuli_03);

                    break;
            }
        }
    };




}

```
### 运行效果
![图片](https://s1.ax1x.com/2018/09/15/iV8gAA.gif)

### handler的内存泄漏

上面的代码有没有发现一些问题或风险，没错就是内存泄漏

#### 什么是内存泄漏
Java使用有向图机制，通过GC自动检查内存中的对象（什么时候检查由虚拟机决定），如果GC发现一个或一组对象为不可到达状态，则将该对象从内存中回收。也就是说，一个对象不被任何引用所指向，则该对象会在被GC发现的时候被回收；另外，如果一组对象中只包含互相的引用，而没有来自它们外部的引用（例如有两个对象A和B互相持有引用，但没有任何外部对象持有指向A或B的引用），这仍然属于不可到达，同样会被GC回收。而一旦GC因为某些对象持有外部引用而无法回收资源时就会发生内存泄露，这部分内存即没有被使用又无法被回收，积少成多就会造成oom，以现在的手机性能造成oom的情况极少，但内存泄露处理仍是android代码质量高低的重要标准之一

#### 为什么handler会造成内存泄露
当使用内部类（包括匿名类）来创建Handler的时候，Handler对象会隐式地持有一个外部类对象（通常是一个Activity）的引用（不然你怎么可能通过Handler来操作Activity中的View？）。而Handler通常会伴随着一个耗时的后台线程（例如从网络拉取图片）一起出现，这个后台线程在任务执行完毕（例如图片下载完毕）之后，通过消息机制通知Handler，然后Handler把图片更新到界面。然而，如果用户在网络请求过程中关闭了Activity，正常情况下，Activity不再被使用，它就有可能在GC检查时被回收掉，但由于这时线程尚未执行完，而该线程持有Handler的引用（不然它怎么发消息给Handler？），这个Handler又持有Activity的引用，就导致该Activity无法被回收（即内存泄露），直到网络请求结束（例如图片下载完毕）。另外，如果你执行了Handler的postDelayed()方法，该方法会将你的Handler装入一个Message，并把这条Message推到MessageQueue中，那么在你设定的delay到达之前，会有一条MessageQueue -> Message -> Handler -> Activity的链，导致你的Activity被持有引用而无法被回收。

#### 如何来处理handler内存泄露
目前方式有如下几种
##### 方法一：通过程序逻辑来进行保护。

1.在关闭Activity的时候停掉你的后台线程。线程停掉了，就相当于切断了Handler和外部连接的线，Activity自然会在合适的时候被回收。

2.如果你的Handler是被delay的Message持有了引用，那么使用相应的Handler的removeCallbacks()方法，把消息对象从消息队列移除就行了。

##### 方法二：将Handler声明为静态类和加入弱引用

有很多文章说设为静态类就可以了，但是实际操作事会发现，由于Handler不再持有外部类对象的引用，导致程序不允许你在Handler中操作Activity中的对象了。所以你需要在Handler中增加一个对Activity的弱引用（WeakReference）：

###### 小扩展：什么是弱引用？
WeakReference弱引用，与强引用（即我们常说的引用）相对，它的特点是，GC在回收时会忽略掉弱引用，即就算有弱引用指向某对象，但只要该对象没有被强引用指向（实际上多数时候还要求没有软引用，但此处软引用的概念可以忽略），该对象就会在被GC检查到时回收掉。对于上面的代码，用户在关闭Activity之后，就算后台线程还没结束，但由于仅有一条来自Handler的弱引用指向Activity，所以GC仍然会在检查的时候把Activity回收掉。这样，内存泄露的问题就不会出现了。

###### 上面的杨幂demo修正版本：

```
//仅给出部分代码，主要看handler的变动
  @Override
    protected void onResume() {
        super.onResume();
        final MyHandler myHandler = new MyHandler(this);

        new Thread(){
            @Override
            public void run() {
                super.run();

                for (int i  = 1; i < 4; i++){
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    Message message = new Message();
                    message.what = i;

                    myHandler.sendMessage(message);
                }


                }


        }.start();


    }

   class MyHandler extends Handler{

        WeakReference<SwipeViewPager> mWeak;

        public MyHandler(SwipeViewPager swipeViewPager) {
            mWeak = new WeakReference<SwipeViewPager>(swipeViewPager);
        }


        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);

            SwipeViewPager swipeViewPager = (SwipeViewPager) mWeak.get();

            if (swipeViewPager != null) {
                switch (msg.what) {
                    case AUTO_SCORLL:
                        if (isStart) {
                            if (vpAd.getChildCount() > 1) {
                                vpAd.setCurrentItem(vpAd.getCurrentItem() + 1, true);
                            }
                            mHandler.sendEmptyMessageDelayed(AUTO_SCORLL, 5000);
                        }
                        break;
                }
            }
        }
    }
  

```

## handler源码分析

### handler机制概括
首先用一张图来概括handler以及它的四大“组件”的工作过程

![图片](https://s1.ax1x.com/2018/09/27/iMbeEj.png)
 handler具有两个功能，1.发送消息 2.处理消息
 handler将消息发送到消息队列，由looper通过轮询从消息队列中将消息取出并派发(通过dispatchMessage()方法)给相应的hanndler，再通过重写handleMessage()方法处理。
### handler
找到构造方法
```
public Handler() {
        this(null, false);
    }
```
 this跟进去
```
 public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper(); //指定了looper对象
        if (mLooper == null) {
            //如果是在子线程中，没有首先调用Looper.prepare()的话就会抛出该异常，具体原因稍后分析looper时再分析
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;   //绑定消息队列
        mCallback = callback;
        mAsynchronous = async;
    }
```
可以看到主要做了两件事
- 1 指定了looper对象
- 2 绑定消息队列
那么问题来了，我们需要看一下looper的源码，本菜鸡看的时候着实被吓一跳，没有头绪苦思冥想一番想到看一眼官方文档是怎么介绍它的，以下是官网给的使用范例

### looper
```
  class LooperThread extends Thread {
      public Handler mHandler;

      public void run() {
          Looper.prepare();

          mHandler = new Handler() {
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();
      }
  }
```
可以看到调用顺序
- 1 Looper.prepare()
- 2 Looper.loop()
以下分别贴出源码
```
   public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) { //看这里
            //ThreadLocal相当于一个容器，是线程内部的数据存储类，通过它可以在指定线程中存储数据，
            只有在指定线程中才能获取到存储的数据，对于其他线程是无法获取到的，通过对sThreadLocal.get()的判空，保证了一个线程只能set一个looper
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
sThreadLocal若为null则new一个looper对象，若不为null抛出异常。
说明
- 1 Looper.prepare()只能调用一次
- 2 一个线程中仅有一个looper

```
  public static void loop() { 

        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //获取looper实例中的消息队列
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        //无限遍历
        for (;;) { 
            //从消息队列中取出消息
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.（若取出消息为null则线程阻塞 --翻译一下怕后期自己看不懂）
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            final long start = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            final long end;
            try {
                //把消息分发给hangler，稍后分析源码
                msg.target.dispatchMessage(msg);
                end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (slowDispatchThresholdMs > 0) {
                final long time = end - start;
                if (time > slowDispatchThresholdMs) {
                    Slog.w(TAG, "Dispatch took " + time + "ms on "
                            + Thread.currentThread().getName() + ", h=" +
                            msg.target + " cb=" + msg.callback + " msg=" + msg.what);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
            //释放
            msg.recycleUnchecked();
        }
    }
```

#### 曾经的疑问之一：用handler用了好多次，哪次也没写looper.prepare()和looper.loop()也跑的好好的。
读完源码发现，早在UI线程创建之初，就绑定了一个looper，凡是在主线程中显示什么什么，都不需要我们自己初始化looper，默认使用主线程的looper，那么代码中是什么时候创建主线程的looper的呢，再上一段代码
```
//在Android应用进程启动时，会默认创建1个主线程
// 创建时，会自动调用ActivityThread的1个静态的main（）方法 = 应用程序的入口
// main（）内则会调用Looper.prepareMainLooper()为主线程生成1个Looper对象
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

### dispatchMessage(msg)分析
下面我们看dispatchMessage(msg)是如何把消息分发给handler
```
 /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg); 

        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            //若msg.callback属性为空，则代表使用了sendMessage
            handleMessage(msg);
        }
    }

```
* 1 检查message的callback是否为null，不为空就交给handleCallback(msg)来处理消息，
message的callback是一个runnable对象，实际就是handler.post()所传递的runnable参数
```
 private static void handleCallback(Message message) {
        message.callback.run();
    }
```
* 2 检查mCallback是否为空，不为空就调用mCallback.handleMessage(msg)来处理消息，Callback是一个接口
```
   /**
     * Callback interface you can use when instantiating a Handler to avoid
     * having to implement your own subclass of Handler.
     *
     * @param msg A {@link android.os.Message Message} object
     * @return True if no further handling is desired
     */
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
    
```
   通过callback可以使用Handler mhandler = new Handler(Callback)来创建handler对象，那么callback是什么呢？
   上文注释里给出了说明：当不想派生handler的子类时可以使用callback创建handler对象，下面是本例使用callback创建的代码（本人不建议使用）
```
  public class MainActivity extends AppCompatActivity implements Callback{

    /*
    中间代码
     
    */
    @Override
      protected void onResume() {
        super.onResume();
        final Handler mhandler = new Handler(this);
        new Thread(){
            @Override
            public void run() {
                super.run();

                for (int i  = 1; i < 4; i++){
                    try {
                        Thread.sleep(3000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    Message message = new Message();
                    message.what = i;
                    mhandler.sendMessage(message);
                 }
                }
        }.start();
    }
      @Override
       public boolean handleMessage(Message msg) {
        switch (msg.what) {
            case 1:
                v1.setImageResource(R.drawable.app_fuli_01);

                break;
            case 2:
                v2.setImageResource(R.drawable.app_fuli_02);

                break;
            case 3:
                v3.setImageResource(R.drawable.app_fuli_03);

                break;
         }
         return true;
     }
```
* 3 最后通过handleMessage来处理消息
```
   /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
        //这是一个空方法，需要我们自己在代码业务中重写
    }
```
### MessageQueen
我们一直称呼它为消息队列，但实际它是一个单链表结构，在handler机制中我们设计MessageQueen的两个方法，取出和插入

#### next()

```
Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }

```
简单的说，这个函数就是从从头Head取出下一个Message，如果队列中没有message了，那么则可以取处理IdleHandler接口。当队列中没有消息或者消息指定了等待时间，那么线程会进入等待状态。
函数中有两个变量
int pendingIdleHandlerCount = -1;空闲的IdleHandler个数，只有在第一次迭代的时候为-1。
int nextPollTimeoutMillis = 0;下一轮等待时间，如果当前消息队列中没哟消息，需要等待的时间。
如果等待时间不为零，执行flush pending command，刷新等待时间。然后执行nativePollOnce(ptr, nextPollTimeoutMillis);其功能是查询消息队列中有没有消息。如果消息为空，那么nextPollTimeoutMillis=-1，接着等待消息，如果不为空，那么就处理这个消息。当我们设置的等待时间到了，将msg从Message中取出来并返回msg，如果没有返回，则说明没有消息需要处理，既然没有消息需要处理，检查以下是否要退出队列，如果退出返回null，否则那么就可以处理IdleHandler，处理完IdleHandler后将nextPollTimeoutMillis设为0（因为在处理IdleHandler的时候可能来消息），重新检测消息。

#### enqueueMessage()

```
   boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;

            //判断，如果mMessages对象为空，或者when为0也就是立刻执行，或者新消息的when时间比mMessages队列的when时间还要早
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                //官方注释，当前消息作为新的队列头部，如果阻塞则唤醒
                //就把新的msg插到mMessages的前面 并把next指向它，也就是队列的最前面，等待loop的轮询。
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                官方注释：插入队列中间。 通常我们不必醒来
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                //when是新消息的执行时间，p.when的是队列中message消息的执行时间，
                如果找到比新的message还要晚执行的消息，
                就执行msg.next = p;prev.next = msg;
                也就是把插到该消息的前面，优先执行新的消息。
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }

```
