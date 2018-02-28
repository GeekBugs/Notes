

## Handler的由来

 　　当程序第一次启动的时候，Android会同时启动一条主线程（ Main Thread）来负责处理与UI相关的事件，我们叫做UI线程。

　　Android的UI操作并不是线程安全的（出于性能优化考虑），意味着如果多个线程并发操作UI线程，可能导致线程安全问题。 

　　为了解决Android应用多线程问题—Android平台只允许UI线程修改Activity里的UI组建，就会导致新启动的线程无法改变界面组建的属性值。

　　**简单的说：**当主线程队列处理一个消息超过5秒,android 就会抛出一个 ANP(无响应)的异常,所以,我们需要把一些要处理比较长的消息,放在一个单独线程里面处理,把处理以后的结果,返回给主线程运行,就需要用的Handler来进行线程建的通信。

## Handler的作用

1. 让线程延时执行

主要用到的两个方法：

- final boolean	postAtTime(Runnable r, long uptimeMillis)
	 final boolean	postDelayed(Runnable r, long delayMillis)

2. 让任务在其他线程中执行并返回结果

分为两个步骤：

 - 在新启动的线程中发送消息

　　使用Handler对象的sendMessage()方法或者SendEmptyMessage()方法发送消息。

 - 在主线程中获取处理消息

　　重写Handler类中处理消息的方法（void handleMessage(Message msg)），当新启动的线程发送消息时，消息发送到与之关联的MessageQueue。而Hanlder不断地从MessageQueue中获取并处理消息。

## Handler更新UI线程一般使用

 - 首先要进行Handler 申明，复写handleMessage方法( 放在主线程中)

```java
private Handler handler = new Handler() {

		@Override
		public void handleMessage(Message msg) {
			// TODO 接收消息并且去更新UI线程上的控件内容
			if (msg.what == UPDATE) {
				// 更新界面上的textview
				tv.setText(String.valueOf(msg.obj));
			}
			super.handleMessage(msg);
		}
	};
```

 - 子线程发送Message给ui线程表示自己任务已经执行完成，主线程可以做相应的操作了。

```java
new Thread() {
			@Override
			public void run() {
				// TODO 子线程中通过handler发送消息给handler接收，由handler去更新TextView的值
				try {
					   //do something
						
						Message msg = new Message();
						msg.what = UPDATE;					
						msg.obj = "更新后的值" ;
						handler.sendMessage(msg);
					}
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		}.start();
```



## Handler原理分析

### Handler的构造函数

> public　Handler()
public　Handler(Callbackcallback)
public　Handler(Looperlooper)
 public　Handler(Looperlooper, Callbackcallback) 　


 - 第①个和第②个构造函数都没有传递Looper，这两个构造函数都将通过调用Looper.myLooper()获取当前线程绑定的Looper对象，然后将该Looper对象保存到名为mLooper的成员字段中。 　
　　下面来看①②个函数源码：


```java
public Handler() {
    this(null, false);
}

public Handler(Callback callback) {
    this(callback, false);
}

//他们会调用Handler的内部构造方法
public Handler(Callback callback, boolean async) {
  if (FIND_POTENTIAL_LEAKS) {
  final Class<? extends Handler> klass = getClass();
  if ((klass.isAnonymousClass() ||klass.isMemberClass()
  || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
}
```

  　　我们看到暗红色的重点部分：

　　通过Looper.myLooper()获取了当前线程保存的Looper实例，又通过这个Looper实例获取了其中保存的MessageQueue（消息队列）。**每个Handler 对应一个Looper对象，产生一个MessageQueue**
　　

 - 第③个和第④个构造函数传递了Looper对象，这两个构造函数会将该Looper保存到名为mLooper的成员字段中。 
　　下面来看③④个函数源码：

```Java
public Handler(Looper looper) {
    this(looper, null, false);
}　

public Handler(Looper looper, Callback callback) {
    this(looper, callback, false);
}
//他们会调用Handler的内部构造方法
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

 - 第②个和第④个构造函数还传递了Callback对象，Callback是Handler中的内部接口，需要实现其内部的handleMessage方法，Callback代码如下:

```java
public interface Callback {
    public boolean More ...handleMessage(Message msg);
}
```

　　Handler.Callback是用来处理Message的一种手段，如果没有传递该参数，那么就应该重写Handler的handleMessage方法，也就是说为了使得Handler能够处理Message，我们有两种办法： 
　　
　1. 向Hanlder的构造函数传入一个Handler.Callback对象，并实现Handler.Callback的handleMessage方法 　

　2. 无需向Hanlder的构造函数传入Handler.Callback对象，但是需要重写Handler本身的handleMessage方法 　

　　　也就是说无论哪种方式，我们都得通过某种方式实现handleMessage方法，这点与Java中对Thread的设计有异曲同工之处。 

## Handle发送消息的几个方法源码

```java
public final boolean sendMessage(Message msg)
 {
     return sendMessageDelayed(msg, 0);
 }
```

```java
public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
     Message msg = Message.obtain();
     msg.what = what;
     return sendMessageDelayed(msg, delayMillis);
}
```

```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
       MessageQueue queue = mQueue;
       if (queue == null) {
           RuntimeException e = new RuntimeException(
                   this + " sendMessageAtTime() called with no mQueue");
           Log.w("Looper", e.getMessage(), e);
           return false;
       }
       return enqueueMessage(queue, msg, uptimeMillis);
}
```

我们可以看出他们最后都调用了sendMessageAtTime（），然后返回了enqueueMessage方法，下面看一下此方法源码：

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
//把当前的handler作为msg的target属性
     msg.target = this;
     if (mAsynchronous) {
         msg.setAsynchronous(true);
     }
     return queue.enqueueMessage(msg, uptimeMillis);
}
```

在该方法中有两件事需要注意：  

1. msg.target = this 

 　　该代码将Message的target绑定为当前的Handler
2. queue.enqueueMessage

　　变量queue表示的是Handler所绑定的消息队列MessageQueue，通过调用queue.enqueueMessage(msg, uptimeMillis)我们将Message放入到消息队列中。

过下图可以看到完整的方法调用顺序： 

![这里写图片描述](http://img.blog.csdn.net/20160516125245477)



## Looper原理分析

　　我们一般在主线程申明Handler，有时我们需要继承Thread类实现自己的线程功能，当我们在里面申明Handler的时候会报错。其原因是主线程中已经实现了两个重要的Looper方法，下面看一看ActivityThread.java中main方法的源码：

```java
public static void main(String[] args) {
            //......省略
        Looper.prepareMainLooper();//>

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        AsyncTask.init();

        if (false) {
            Looper.myLooper().setMessageLogging(new
   LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();//>

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
}
```

### 首先看prepare()方法

```java
public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
//证了一个线程中只有一个Looper实例
      if (sThreadLocal.get() != null) {
          throw new RuntimeException("Only one Looper may be created per thread");
      }
      sThreadLocal.set(new Looper(quitAllowed));
}
```
该方法会调用Looper构造函数同时实例化出MessageQueue和当前thread.

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
} 

public static MessageQueue myQueue() {
    return myLooper().mQueue;
}
```


　　prepare()方法中通过ThreadLocal对象实现Looper实例与线程的绑定。（不清楚的可以查看　[ThreadLocal的使用规则和源码分析](http://blog.csdn.net/amazing7/article/details/51313851)）　

### loop()方法

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
}
final MessageQueue queue = me.mQueue;

Binder.clearCallingIdentity();
final long ident = Binder.clearCallingIdentity();

for (;;) {
    Message msg = queue.next(); // might block
    if (msg == null) {
       
        return;
    }

    Printer logging = me.mLogging;
    if (logging != null) {
        logging.println(">>>>> Dispatching to " + msg.target + " " +
                msg.callback + ": " + msg.what);
    }
//重点****
         msg.target.dispatchMessage(msg);

         if (logging != null) {
             logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }

        msg.recycleUnchecked();
    }
}
```
 　　首先looper对象不能为空，就是说loop()方法调用必须在prepare()方法的后面。

　Looper一直在不断的从消息队列中通过MessageQueue的next方法获取Message，然后通过代码msg.target.dispatchMessage(msg)让该msg所绑定的Handler（Message.target）执行dispatchMessage方法以实现对Message的处理。 

Handler的dispatchMessage的源码如下：

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```
 　　我们可以看到Handler提供了三种途径处理Message，而且处理有前后优先级之分：首先尝试让postXXX中传递的Runnable执行，其次尝试让Handler构造函数中传入的Callback的handleMessage方法处理，最后才是让Handler自身的handleMessage方法处理Message。



## 如何在子线程中使用Handler

  　　Handler本质是从当前的线程中获取到Looper来监听和操作MessageQueue，当其他线程执行完成后回调当前线程。

　　子线程需要先prepare（）才能获取到Looper的，是因为在子线程只是一个普通的线程，其ThreadLoacl中没有设置过Looper，所以会抛出异常，而在Looper的prepare（）方法中sThreadLocal.set(new Looper())是设置了Looper的。

### 实例代码

　定义一个类实现Runnable接口或继承Thread类（一般不继承）。

```java
class Rub implements Runnable {  

    public Handler myHandler;  
    // 实现Runnable接口的线程体 
    @Override  
    public void run() {  
    	
     /*①、调用Looper的prepare()方法为当前线程创建Looper对象并，
      创建Looper对象时，它的构造器会自动的创建相对应的MessageQueue*/
        Looper.prepare();  
        
        /*.②、创建Handler子类的实例，重写HandleMessage()方法，该方法处理除当前线程以外线程的消息*/
         myHandler = new Handler() {  
            @Override  
            public void handleMessage(Message msg) {  
                String ms = "";  
                if (msg.what == 0x777) {  
                 
                }  
            }  

        };  
        //③、调用Looper的loop()方法来启动Looper让消息队列转动起来
        Looper.loop();  
    }
}
```

注意分成三步：　

１．调用Looper的prepare()方法为当前线程创建Looper对象，创建Looper对象时，它的构造器会创建与之配套的MessageQueue。 　

２．有了Looper之后，创建Handler子类实例，重写HanderMessage()方法，该方法负责处理来自于其他线程的消息。 　

３．调用Looper的looper()方法启动Looper。

　　然后使用这个handler实例在任何其他线程中发送消息，最终处理消息的代码都会在你创建Handler实例的线程中运行。

## 总结

1. **Handler**： 

　　　　　　发送消息，它能把消息发送给Looper管理的MessageQueue。 
　　　　　　处理消息，并负责处理Looper分给它的消息。 

2. **Message**：

　　　　　　Handler接收和处理的消息对象。 

3. **Looper**： 

　　　　　　每个线程只有一个Looper，它负责管理对应的MessageQueue，会不断地从MessageQueue取出消息，并将消息分给对应的Hanlder处理。 　
　　　　　　
　　　　　　主线程中，系统已经初始化了一个Looper对象，因此可以直接创建Handler即可，就可以通过Handler来发送消息、处理消息。 程序自己启动的子线程，程序必须自己创建一个Looper对象，并启动它，调用Looper.prepare()方法。 

prapare()方法：保证每个线程最多只有一个Looper对象。 　

looper()方法：启动Looper，使用一个死循环不断取出MessageQueue中的消息，并将取出的消息分给对应的Handler进行处理。 　

MessageQueue：由Looper负责管理，它采用先进先出的方式来管理Message。　

　　Handler的构造方法，会首先得到当前线程中保存的Looper实例，进而与Looper实例中的MessageQueue想关联。　
　　
　　Handler的sendMessage方法，会给msg的target赋值为handler自身，然后加入MessageQueue中。

