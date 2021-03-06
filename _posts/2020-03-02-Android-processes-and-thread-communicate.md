---
layout: post
title:  "Android进程和线程间通信方式"
author: "陈宇瀚"
date:   2020-03-02 11:46:20 +0800
categories: article
tags:
  - Android 
  - 进程与线程
---
# Android进程和线程间通信方式
进程：是具有一定独立功能的程序关于某个数据集合上的一次运行活动，进程是系统进行资源分配和调度的一个独立单位。

线程：是进程的一个实体,是CPU调度和分派的基本单位,它是比进程更小的能独立运行的基本单位。线程自己基本上不拥有系统资源,只拥有一些在运行中必不可少的资源(如程序计数器,一组寄存器和栈)，但是它可与同属一个进程的其他的线程共享进程所拥有的全部资源。

区别：

- （1）、一个程序至少有一个进程，一个进程至少有一个线程；

- （2）、线程的划分尺度小于进程，使得多线程程序的并发性高；

- （3）、进程在执行过程中拥有独立的内存单元，而多个线程共享内存，但线程之间没有单独的地址空间，一个线程死掉就等于整个进程死掉。

## Android进程间通信方式
可分为七种
### 1、Bundle
由于Activity,Service,Receiver都是可以通过Intent来携带Bundle传输数据的，所以我们可以在一个进程中通过**Intent**将携带数据的**Bundle**发送到另一个进程的组件。
由于**Bundle**实现了**Parcelable**接口，所以他可以方便地在不同进程间传递。
缺点：无法传输Bundle不支持的类型，即传输的数据必须能够序列化（如**基本类型**，实现了**Parcelable**或者**Serializable**接口的对象以及一些Aandroid支持的对象）；

### 2、文件共享
两个进程通过读/写同一个文件来交换数据，由于Android系统基于Linux，所以其并发读/写文件可以没有限制的进行，除了可以通过文件交换一些文本信息外，还可以序列化一个对象到文件系统中的同时从另一个进程恢复这个对象。
文件共享适合在对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读/写问题
缺点：不适合**并发读/写**；

当然 **SharedPreference**是个特例，**SharedPreference**是Android中提供的轻量级存储方案，通过键值对的方式存储数据，在底层上实现上采用**XML**文件来存储键值对，每个应用的**SharePreference**文件都在当前包所在的data目录下查看到，**/data/package name/shared_prefs**目录下。
**SharedPreference**也属于文件的一种，但是由于系统对他的读/写有一定的缓存策略，即在内存中会有一份**SharedPreference**文件的缓存你，因此在多进程模式下，系统对它的的读/写就变的不可靠，当面对高并发的读/写访问，SharedPreferences有很大几率会丢失数据，因此，不建议在进程间通信中使用**SharedPreference**。

### 3、Messenger
**Messenger**可翻译为心事，通过它可以在不同进程中传递**Message**对象，**Messenger**是一种轻量级的IPC方案，基于AIDL实现，服务端（被动方）提供一个Service来处理客户端（主动方）连接，维护一个**Handler**来创建**Messenger**，在**onBind**时返回**Messenger**的**binder**。双方用**Messenger**来发送数据，用**Handler**来处理数据。**Messenger**处理数据依靠**Handler**，所以是**串行**的，也就是说，**Handler**接到多个**message**时，就要**排队依次处理**。如果有大量的并发请求，使用**Messenger**就不太合适了。
使用方式如下：
- 在服务端进程创建一个**Service**来处理客户端的连接请求，同时创建一个**Handler**并通过创建一个**Messenger**对象，然后再Service的**onBind**中返回这个**Messenger**对象给底层的Binder即可；
- 在客户端进程绑定服务端的**Service**，绑定成功后用服务端返回的**IBinder**对象创建一个**Messenger**，通过这个**Messenger**就可以向服务端发送消息了，发消息类型为**Message**对象，如果需要服务端能够回应客户端，还需创建一个**Handler**并创建一个新的**Messenger**。并把这个**Messenger**对象通过**Message**的**replyTo**参数传递给服务端。服务端通过这个**replyTo**参数就可以回应客户端。
- 在**Messenger**中进行数据传递必须将数据放入**Message**中，而**Messenger**和**Message**都实现了**Parcelable**接口，因此可以跨进程传输。同时，**Message**所支持的数据类型就是**Messenger**所支持的传输类型。

### 4、ContentProvider
**ContentProvider**是Android四大组件之一，换门用于不同应用间进行数据共享，以表格的方式存储数据，底层实现基于**Binder**，由于系统的封装，**ContentProvider**使用比AIDL简单。系统预置了许多**ContentProvider**（例如通讯录信息，日程表，信息等）。
**ContentProvider**的使用方法：
- 继承**ContentProvider**，并实现六个抽象方法：
- - 1、onCreate：代表**ContentProvider**的创建，一般做些初始化工作；
- - 2、insert：增；
- - 3、delete：删
- - 4、update：改；
- - 5、query：查；
- - 6、getType：用来返回一个Uri请求所对应的**MIME**类型（如媒体类型），比如图片，视频等，可直接返回null或"*/*"（若不关注这个选项的话）。
- onCreate由系统回调运行在主线程，其他五个方法由外界回调并运行在**Binder**线程池中。

**ContentProvider**主要以表格的形式来组织数据，并且可以包含多个表，对于每个表来说都具有行和列的层次性，行往往对应一条记录，列对应一条记录中的一个字段，这点和数据库类似。除了表哥形式，**ContentProvider**还支持文件数据，比如图片、视频等。

### 5、Broadcast
**Broadcast**可以向Android系统中所有应用程序发送广播，而需要跨进程通讯的应用程序可以监听这些广播。

### 6、Socket
**Socket**是网络通信的概念，分为**流式Socket**和**用户数据报Socket**两种，分别对应网络的传输控制层种的TCP和UDP协议。**Socket**本身可以支持传输任意字节流。**Socket**通过网络进行数据交换，需要在子线程请求，不然会阻塞主线程，客户端和服务端建立连接之后即可不断传输数据，比较适合实时的数据传输。使用时在远程Service建立一个TCP服务，在主界面连接TCP服务，连接成功后向服务端发送消息。

### 7、AIDL
**AIDL**进行进程间通信的流程，分为服务端和客户端两个方面
- 1、服务端  
     创建一个**Service**用来监听客户端的连接请求，之后创建一个AIDL文件（通过AIDL文件会生成一个java文件，内部包含Stub和Stub.Proxy）将暴露给客户端的接口在AIDL声明，最后在**Service**中实现**Stub**即实现**AIDL**接口即可；
- 2、客户端  
     首先绑定服务端的Service，绑定成功后，将服务端返回的**Binder**对象转成AIDL接口所属的类型，接着就可以调用**AIDL**方法了
  
AIDL文件支持的数据类型：
- 1、基本数据类型（int、long、char、boolean、double等）；
- 2、String和CharSequence；
- 3、List：只支持ArrayList，里面每个元素都必须能够被**AIDL**支持；
- 4、Map：只支持HashMap，里面的每个元素都必须被**AIDL**支持，包括Key和Value；
- 5、Parcelable：所有实现了Parcelable接口的对象；
- 6、AIDL：所有的ALDI接口本身也可以在AIDL文件中使用，如果AIDL文件中用到了自定义的**Parcelable**对象，必须新创建一个和它同名的AIDL文件，并在其中声明它为**Parcelable**类型。  
AIDL除了基本数据类型，其他类型的参数必须标上方向；
- in:输入型参数；
- out：输出型参数；
- inout：输入输出型参数；
例： void addBook(int Book book);
不能一致使用inout，这在底层是有开销的。  
AIDL接口中支持方法，不支持声明的静态常量，这一点有别于传统接口。  
建议将AIDL相关的类和文件全部放在同一个包中，好处是当客户端是另一个应用是，可以将整个包复制到客户端工程中。AIDL的包结构在服务端和客户端要保持一致。  
如果要在服务端和客户端建立监听，需使用**RemoteCallbackList**,因为正常传入的接口对象会在服务端重新转化成一个新的对象，解注册的时候对应不上。  
**RemoteCallbackList**是系统专门提供的用于删除跨进程listener的接口，它的内部有一个Map结构，专门用来保存所有的AIDL回调，Map的key是**IBinder**类型，value是**Callback**类型。

使用示例：
```java
private RemouteCallbackList<ITestListener> = mListenerList = 
    new RemoteCallbackList<ItestListener>();
    
//注册
mListenerList.register(listener);
//解注册
mListenerList.unregister(listener);
//遍历
final int N = mListenerList.beginBroadcast();
for(int i=0;i<N;i++){
    mListenerList.getBroadcastItem(i).xxx();
}
mListenerList.finishBroadcast();
```
- 客户端调用远程服务的方法，被调用的方法运行在服务端的**Binder线程池**中，同时客户端会被挂起，这时如果服务端执行比较耗时，就会导致客户端线程阻塞。不可再UI线程调用服务端耗时方法。需要把调用放在非UI线程。同样，当服务端调用客户端的listener中的方法是，被调用的方法运行在客户端的**Binder线程池**，所以同样不可以在服务端中调用客户端耗时方法。若要调用，保证运行在非UI线程，否则将导致服务端无法响应。
- 由于客户端的listener运行在客户端的**Binder线程池**，所以不能在它里面访问UI相关的内容，如果要访问UI，需要使用Handler切换到UI线程。

Binder可能会意外停止，往往是由于服务端进程意外停止了，这时需要重连服务，有两种方式：
- 1、给Binder设置**DeathRecipient**监听，当Binder死亡时，会收到BinderDied方法的回调
```java
private IBinder.DeathRecipient mDeathRecipient = 
        new IBinder.DeatgRecipient(){
            @Overwide
            public void binderDied(){
                if(mTestManager == null){
                    return;
                }
                mTestManager.asBinder().unlinkToDeath(mDeathRecipient);
                mTestMnager = null;
                //这次重新绑定远程Service
                ...
            }
            
        }
```
在客户端绑定远程服务后，给Binder设置死亡代理
```java
mService = ITestManager.Stub.asInterface(binder);
binder.linkToDeath(mDeathRecipient,0);
```
- 2、在onServiceDisconnected中重连远程服务  
区别在于onServiceDisconnected在客户端的UI线程被回调，而binderDied在客户端的Binder线程池中回调；
  
在AIDL中加入**权限验证**功能  
有两种常用方法：  
- 1、在onBind中进行验证，验证不通过直接返回null,这样验证失败的客户端无法绑定服务。验证方法有很多种，比如使用**premission**验证：首先在AndroidManifest中声明所需权限：
```xml
<permission
    android:name = "com.cyh.test.ACCESS_SERVICE"
    android:procetionLevel="normal"/>
```
内部的应用绑定到我们的服务同样在AndroidManifest中加入
```xml
<user-permission 
    android:name = "com.cyh.test.ACCESS_SERVICE"/>
```
之后在Service类中
```java
TestManagerService.onBind:
public IBinder onBind(Intent intent){
    int check = checkCallingOrSelfPermission("com.cyh.test.ACCESS_SERVICE");
    if(check == PackageManager.PERMISSION_DENIED){
        return null;
    }
    return mBinder;
}
```  
- 2、可以在服务端的onTransact方法中进行验证，验证失败则直接返回false，这样服务端会中指进行AIDL中的方法，从而达到保护服务器的效果。验证方法同样有很多，可以采用与1类型的permission验证。还可以采用Uid和Pid验证，通过**getCallingUid**和**getCallingPid**可以拿到客户端所属应用的**Pid**和**Uid**，通过这两个参数可以做一些验证工作,比如验证包名：
```java
//包名验证如下：
String packageName = null;
String[] packages = getPackageManger().getPackagesForUid(getCallingUid());
if(packages!=null&& packages.length>0){
    packageName = packages[0];
}
if(!packageName.startWith(com.cyh)){
    return false;
}
return super.onTransact(code,data,reply,flags);
```

