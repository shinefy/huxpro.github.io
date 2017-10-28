---
layout:     post
title:      "Android中AIDL简单学习"
subtitle:   ""
date:       2017-10-28 12:00:00
author:     "Shinefy"
header-img: ""

---

### Android中的多进程

在Android中，正常情况下，一个应用只会运行在一个单独的进程中。但是如果需要将某些组件（如Service、Activity等）运行在单独的进程中，就需要在AndroidManifest中将该组件设置为android:process=":remote"即可。

- 使用多进程有什么好处

	1. 我们知道Android系统对每个应用进程的内存占用是有限制的，即Android对内存的限制是针对于进程的，而且占用内存越大的进程，通常被系统杀死的可能性越大。让一个组件运行在单独的进程中，可以减少主进程所占用的内存，降低被系统杀死的概率.
	2. 如果子进程因为某种原因崩溃了，不会直接导致主程序的崩溃，可以降低我们程序的崩溃率。
	3. 即使主进程退出了，我们的子进程仍然可以继续工作，假设子进程是推送服务，在主进程退出的情况下，仍然能够保证用户可以收到推送消息


- 同时会带来的问题

	1. Application多次实例化：解决办法是在Application onCreate()时判断当前进程名称。
	2. 每个进程间的内存空间是隔离的，因此任何class、变量、常量等等都是无法共享的，需要格外注意


---

### Android 进程间通信的几种方式


1. 使用Bundle ： Bundle实现了Parcelable接口，四大组件间可以通过intent传递Bundle给远程进程。
2. 使用文件共享
3. 使用AIDL通过Binder进行通信
4. 使用Messenger，Messenger封装了AIDL，易用，但是只能以串行的方式处理CLient的消息。
5. 使用ContentProvider，ContentProvider是Android中专门用于不同应用间进行数据共享的方式，因此天生就适合进程间通信，其底层实现也是Binder
6. 使用Socket

	(参见《Android开发艺术探索》P65）

---

### AIDL

Android Interface Definition Language : 简称AIDL，是Android进程通信接口的描述语言，用于生成可以在两个进程之间进行进程间通信(IPC)的代码。

```java
//IDownloadManager.aidl
//举例为一个单独的下载进程，有开始下载和获取下载进度的功能
interface IDownloadManager {
    int getDownloadProgress();
    void startDownload();
}
```
build一下，我们就看到Android帮我们生成了相应的Binder类文件。我们直接在Server端和Client端直接使用即可。
接下来通过部分源码+自己实现Binder(不通过AIDL自动生成)的形式，把Binder上层结构整理了一下下。
 

```java
//源码
public interface IBinder {
	int FIRST_CALL_TRANSACTION  = 0x00000001;
	public boolean transact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException;
}
```



```java
//源码
public class Binder implements IBinder {
	 
	 
	 //我们在构造Stub(即下面的DownloadManagerImpl)时，调用了native的init方法，将该Binder注册到系统中。
	 //貌似BinderProxy的代理相关代码好像也是在这里...猜测...因为java层我找不到
	 public Binder() {
        init();  
        ...
    }
	
	private native final void init();
	
	public final boolean transact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
        if (false) Log.v("Binder", "Transact: " + code + " to " + this);

        if (data != null) {
            data.setDataPosition(0);
        }
        boolean r = onTransact(code, data, reply, flags);
        if (reply != null) {
            reply.setDataPosition(0);
        }
        return r;
    }
    
    
        
    
	//Default implementation is a stub that returns false.
	protected boolean onTransact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
        if (code == INTERFACE_TRANSACTION) {
            reply.writeString(getInterfaceDescriptor());
            return true;
        } else if (code == DUMP_TRANSACTION) {
            ...
            return true;
        } else if (code == SHELL_COMMAND_TRANSACTION) {
        	  ...
            return true;
        }
        return false;
    }
    
}
```

```java
//源码
final class BinderProxy implements IBinder {

	//Client端proxy调用了transact方法，实际上调用了BinderProxy的transact方法
    public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        ...
        try {
            return transactNative(code, data, reply, flags);
        } finally {
            if (tracingEnabled) {
                Trace.traceEnd(Trace.TRACE_TAG_ALWAYS);
            }
        }
    }
    
    //transactNative方法会通过NDK和底层通信
        public native boolean transactNative(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException;


}
```



```java
//源码
public interface IInterface
{
    public IBinder asBinder();
}
```

```java
public interface IDownloadManager implements IInterface{
	public void startDownload();
	public int getDownloadProgress();
	
}
```

```java
//就是AIDL中的Stub，Binder的具体实现
public class DownloadManagerImpl extends Binder implements IDownloadManager{

	static final TRANSACTION_startDownload = IBinder.FIRST_CALL_TRANSACTION+1;
	static final TRANSACTION_ getDownloadProgress = IBinder.FIRST_CALL_TRANSACTION+2;

	public void startDownload(){
		//开始下载
		Log.d("Tag","开始下载")
	}
	
	public int getDownloadProgress(){
		//获取下载进度
		return 100;
	}
	
	
	@Override 
	public IBinder asBinder(){
		return this;
	}
	
	//Client端通过该静态方法获取Proxy对象后，调用相应的服务
	public static IDownloadManager asInterface(IBinder obj){
		if ((obj==null)) {
			return null;
			}
		...
		renturn new DownloadManagerProxy(obj)
	}

	//Client调transactNative方法后，将找到底层的相应的IBinder，底层的IBinder驱动会回调这个方法，并将结果返回至应用层
	protected boolean onTransact(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException {
        switch(code){
        	case(INTERFACE_TRANSACTION):{
        		reply.writeString(getInterfaceDescriptor());
             return true;
        	}
        	case(TRANSACTION_startDownload):{
        		data.enforceInterface(DESCRIPTOR);
				this.getDownloadProgress();
				reply.writeNoException();
				return true;
        	}
        	case(TRANSACTION_ getDownloadProgress):{
        		data.enforceInterface(DESCRIPTOR);
				int _result = this.getDownloadProgress();
				reply.writeNoException();
				reply.writeString(_result);
				return true;
        	}
        }  
    }
    
    
	
}

```

```java
//Client端获得的Server服务的代理
public class DownloadManagerProxy implements IDownloadManager{

	private IBinder mRemote;
		
	Proxy(IBinder remote){
		mRemote = remote;
   }
       
	@Override 
	public IBinder asBinder(){
		return mRemote;
	}
	
	@Override
    public void startDownload() throws RemoteException{
        Parcel _data = Parcel.obtain();
        Parcel _reply = Parcel.obtain();
        try {
            _data.writeInterfaceToken(DESCRIPTOR);
            mRemote.transact(IBinderTest.Stub.TRANSACTION_startDownload, _data, _reply, 0);
            _reply.readException();
        }
        finally {
            _reply.recycle();
            _data.recycle();
        }
    }
	
	@Override
    public int getDownloadProgress() throws RemoteException {
        Parcel _data = Parcel.obtain();
        Parcel _reply = Parcel.obtain();
        int _result;
        try {
            _data.writeInterfaceToken(DESCRIPTOR);
            mRemote.transact(TRANSACTION_getDownloadProgress, _data, _reply, 0);
            _reply.readException();
            _result = _reply.readInt();
        }
        finally {
            _reply.recycle();
            _data.recycle();
        }
        return _result;
    }
		
}
```

使用该Binder：

```java
//Server端
public class SeverService extends Service {

	private DownloadManagerImpl binder = new DownloadManagerImpl();
	
	@Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
	
}


//Client端
Intent intent = new Intent(this,SeverService.class);
bindService(intent,serviceConnection, Context.BIND_AUTO_CREATE);

private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            try {
                IDownloadManager manager = DownloadManagerImpl.asInterface(service);
                manager.startDownload();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

```

一个我瞎画的类似UML的关系图（忽略了BinderProxy类）：
![Binder](http://pics-markdown.oss-cn-hangzhou.aliyuncs.com/Binder.png)



整个过程大概是这样的：
我们在Client端通过
IDownloadManager manager = DownloadManagerImpl.asInterface(service);
这段代码获取了一个Proxy对象，通过该客户端Proxy调用AIDL接口方法时，会调用
mRemote.transact(Stub.TRANSACTION_XXXX, _data, _reply, 0);
客户端Proxy会去底层系统中找到相应的IBinder,IBinder再将请求回调给服务端的Stub，即回调了服务端Stub中的onTransact方法。
那么这个时候，服务端Stub就会调用具体实现的方法，并将结果写入reply中，此时客户端proxy同步获取到这个reply。

- 如果要在使用AIDL时实现观察者模式，使用常规的Listener监听模式即可，唯一不同的时需要为Listener接口新建一个单独的AIDL。（参见《Android开发艺术探索》P77）



