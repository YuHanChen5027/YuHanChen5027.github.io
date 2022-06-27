---
layout: post
title:  "Android蓝牙短信功能开发"
author: "陈宇瀚"
date:   2022-03-03 12:05:40 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android
  - Bluetooth
---
## Android蓝牙短信功能开发
---
由于我司是做车机中控的，目前需要在车机上实现与蓝牙手机相连，并通过蓝牙进行短信发送和接收的功能，针对这一功能的实现方式做个简单的记录。

### 文档目录说明
---
-   [**一、蓝牙短信协议规范**](#*一蓝牙短信协议规范)
-   [**二、协议SDK文件接口说明**](#二协议SDK文件接口说明)
-   [**三、蓝牙MapClient协议支持**](#三蓝牙MapClient协议支持)
-   [**四、蓝牙短信接收功能开发**](#四蓝牙短信接收功能开发)  
-   [**五、蓝牙短信发送功能开发**](#五蓝牙短信发送功能开发)  

*以下开发基于Android 9.0 车机版本，其他版本或存在不同*
### 一、蓝牙短信协议规范
--- 
相关蓝牙协议：**MAP**（MESSAGE ACCESS PROFILE）：蓝牙短信访问协议规范。

借助**MAP**协议规范，可在车机上通过连接的远程设备**收发短信**。目前不会将短信内容存储在 IVI 本地存储空间，而是每当连接的远程设备收到短信时，IVI 会接收相应短信并对其进行解析，然后在 intent 中**广播**消息内容，应用便可收到相应内容。

要连接到移动设备以收发短信，IVI 必须启动 MAP 连接。 MapClientService 中的 **MAXIMUM_CONNECTED_DEVICES** 指定了 IVI 允许同时连接的 **MAP** 设备数量上限。同时获得 IVI 和移动设备的授权时每个连接才能传输消息。

协议SDK代码在源码内的路径：**frameworks/base/core/java/android/bluetooth/BluetoothMapClient.java**

协议服务代码在源码内的路径：**packages/apps/Bluetooth/src/com/android/bluetooth/mapclient/MapClientService.java**

### 二、协议SDK文件接口说明

接口名 | 描述
---|---
connect | 连接指定设备
disconnect | 断开指定设备
isConnected | 判断指定设备是否连接，连接则返回true，否则false
getConnectedDevices | 获有已连接设备列表
getDevicesMatchingConnectionStates | 获得与指定状态匹配的设备
getConnectionState | 获得指定设备的连接状态
setPriority | 设置设备MAP协议的优先级
getPriority | 获得设备MAP协议的优先级
sendMessage | 使用指定设备发送消息至指定的联系人
getUnreadMessages | 获得未读消息

其中我们需要重点关注的接口如下：
```java
    /**
     * 向指定的电话号码发送SMS消息
     * 
     * @param device 蓝牙设备
     * @param contacts 联系人的Uri[]列表
     * @param message y要发送的消息
     * @param sentIntent 发送消息时发出的意图 SMS消息发送成功将发送{@link #ACTION_MESSAGE_SENT_SUCCESSFULLY} 广播
     * @param deliveredIntent 消息传递时发出的意图 SMS消息传递成功将发送{@link #ACTION_MESSAGE_DELIVERED_SUCCESSFULLY} 广播
     * @return 如果消息入队则返回 true，错误则返回 false
     */
    public boolean sendMessage(BluetoothDevice device, Uri[] contacts, String message,
            PendingIntent sentIntent, PendingIntent deliveredIntent) 
            
    /**
     * 获取未读消息.  未读消息将发送 {@link #ACTION_MESSAGE_RECEIVED} 广播。
     *
     * @param device 蓝牙设备
     * @return 如果消息入队则返回 true，错误则返回 false
     */
    public boolean getUnreadMessages(BluetoothDevice device)
    
```
### 三、蓝牙MapClient协议支持
若需要车机设备与支持**MAP**的蓝牙设备连接后，可进行短信收发，那么我们车机系统就需要在蓝牙配对时支持**MapClient**协议规范，那么我们需要修改或overlay **frameworks/base/core/res/res/values/config.xml**内的**enable_pbap_pce_profile**配置值为**true**(**AutoMotive内该值默认overlay为true**)，这样我们在进行蓝牙设备连接时才会将**MapClientProfile**协议加入到可连接的蓝牙协议规范列表内，具体的代码如下：
**framework/base/packages/SettingsLib/src/com/android/settingslib/bluetooth/LocalBluetoothProfileManager.java**
```java
public class LocalBluetoothProfileManager {
    ....
    private MapClientProfile mMapClientProfile;
    ....
    LocalBluetoothProfileManager(Context context,
            LocalBluetoothAdapter adapter,
            CachedBluetoothDeviceManager deviceManager,
            BluetoothEventManager eventManager) {
            ....
            
            // pbap为电话簿访问协议规范
            mUsePbapPce = mContext.getResources().getBoolean(R.bool.enable_pbap_pce_profile);
            // MAP Client 通常用于与 PBAP Client 相同的情况
            mUseMapClient = mContext.getResources().getBoolean(R.bool.enable_pbap_pce_profile);
            
            ....
            if (mUseMapClient) {
                if(mMapClientProfile==null){
                    mMapClientProfile = new MapClientProfile(mContext, mLocalAdapter,
                                        mDeviceManager, this);
                    addProfile(mMapClientProfile, MapClientProfile.NAME,
                        BluetoothMapClient.ACTION_CONNECTION_STATE_CHANGED);
                }
            } else {
                if(mMapProfile==null){
                    mMapProfile = new MapProfile(mContext, mLocalAdapter, mDeviceManager, this);
                    addProfile(mMapProfile, MapProfile.NAME,
                        BluetoothMap.ACTION_CONNECTION_STATE_CHANGED);
                }
            }
            ....
                
    }
    ....        
}
```
之后进行连接就可以在车机的蓝牙设备详情页面看到对应的协议开关选项，以U918A为例，存在如下选项：

!](https://upload-images.jianshu.io/upload_images/4273129-aa631c686297780b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

勾选之后就会进行蓝牙短信协议的连接。

#### MapClient连接设备状态监听
当**MapClient**设备连接状态变化时，会发送**BluetoothMapClient.ACTION_CONNECTION_STATE_CHANGED**广播，我们需要监听该广播，并记录当前连接的蓝牙设备，以备后续的**未读短信获取**和**短信发送**功能的开发，具体的代码如下:
```java
    // 记录当前连接的蓝牙设备
    private BluetoothDevice mBluetoothDevice;

    // MapClient设备连接状态变化广播注册
    IntentFilter filter = new IntentFilter();
    filter.addAction(BluetoothMapClient.ACTION_CONNECTION_STATE_CHANGED);
            BtServiceApplication.getApplication().registerReceiver(mBroadcastReceiver, filter);
            
    // MapClient设备连接状态变化广播监听
    private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            if (action.equals(BluetoothMapClient.ACTION_CONNECTION_STATE_CHANGED)) {
                if (intent.getIntExtra(BluetoothProfile.EXTRA_STATE, 0) == BluetoothProfile.STATE_CONNECTED) {
                    // 设备连接
                    mBluetoothDevice = (BluetoothDevice) intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);
                } else if(intent.getIntExtra(BluetoothProfile.EXTRA_STATE, 0) == BluetoothProfile.STATE_DISCONNECTED){
                    // 设备断开
                    BluetoothDevice bluetoothDevice = (BluetoothDevice) intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);
                    if(bluetoothDevice!=null && mBluetoothDevice.getAddress().equals(bluetoothDevice.getAddress())){
                    mBluetoothDevice = null;
                }
            }
            }
        }
    };
            
```

### 四、蓝牙短信接收功能开发
根据官方协议来看，目前可支持的蓝牙短信接收功能有如下两个：
1. 获取所有未读短信；
2. 实时短信接收；
*针对已读短信仅通过现有的协议还无法实现，后续有机会再继续研究。*
根据协议规范说明可知，蓝牙短信收实际上是通过广播的形式进行接收的，所以我们只需要在代码监听对应的广播信息，解析内部的短信内容即可，实现代码如下：
```java
    // 短信接收广播注册
    IntentFilter filter = new IntentFilter();
    filter.addAction(BluetoothMapClient.ACTION_MESSAGE_RECEIVED);
    BtServiceApplication.getApplication().registerReceiver(mBroadcastReceiver, filter);
     
    // 短信接收广播监听
    private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            if (action.equals(BluetoothMapClient.ACTION_MESSAGE_RECEIVED)) {
                // 收到短信
                // 获得发送者的Uri
                String senderUri = intent.getStringExtra(BluetoothMapClient.EXTRA_SENDER_CONTACT_URI);
                if (senderUri == null) {
                    senderUri = "<null>";
                }
                // 获得发送者名称
                String senderName = intent.getStringExtra(BluetoothMapClient.EXTRA_SENDER_CONTACT_NAME);
                if (senderName == null) {
                    senderName = "<null>";
                }
                
                // 获得短息内容
                String message = intent.getStringExtra(android.content.Intent.EXTRA_TEXT);
                // 获得短信接受的蓝牙设备
                String bluetoothDevice = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);
            }
        }
    };
```
通过以上代码即可实现实时短信的接收，若要获得未读短信，则需要调用**BluetoothMapClient**的**getUnreadMessages**接口触发对应的蓝牙设备将未读短信发送至我们的车机上，接收处理的上面一致，都是通过广播来进行接收，就不再阐述。

针对**BluetoothMapClient**的对象创建和**getUnreadMessages**的调用可参考如下代码:
```java
public class Test {
    private BluetoothMapClient mMapProfile;
    private BluetoothAdapter mAdapter;
    private final int MAP_CLIENT = 18;
    
    class MapServiceListener implements BluetoothProfile.ServiceListener {
        @Override
        public void onServiceConnected(int profile, BluetoothProfile proxy) {
            synchronized (mLock) {
                mMapProfile = (BluetoothMapClient) proxy;
            }
        }

        @Override
        public void onServiceDisconnected(int profile) {
            synchronized (mLock) {
                mMapProfile = null;
            }
        }
    }
    
    public Test(Contexst context){
        mAdapter = BluetoothAdapter.getDefaultAdapter();
        mAdapter.getProfileProxy(mContext, new MapServiceListener(), MAP_CLIENT);
    }


    // 获得未读短信
        public void syncUnreadMessages(String address) {
        synchronized (mLock) {
            // 触发同步
            BluetoothDevice remoteDevice;
            try {
                // 获得对应Address的蓝牙远程设备
                remoteDevice = mAdapter.getRemoteDevice(address);
            } catch (java.lang.IllegalArgumentException e) {
                return;
            }

            if (mMapProfile != null) {
                // 触发未读短信获取
                boolean isSuccess = mMapProfile.getUnreadMessages(remoteDevice);
            }
        }
    }
}
```

### 五、蓝牙短信发送功能开发
蓝牙短信发送可参考代码如下(**BluetoothMapClient**对象创建过程省略)：
```java
    // 蓝牙短信发送成功广播注册
    IntentFilter filter = new IntentFilter();
    filter.addAction(BluetoothMapClient.ACTION_MESSAGE_SENT_SUCCESSFULLY);
    filter.addAction(BluetoothMapClient.ACTION_MESSAGE_DELIVERED_SUCCESSFULLY);
    BtServiceApplication.getApplication().registerReceiver(mBroadcastReceiver, filter);
        
    // 蓝牙短信发送成功广播监听
    private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            if (action.equals(BluetoothMapClient.ACTION_MESSAGE_SENT_SUCCESSFULLY)) {
                Logcat.d("Message sent successfully");
            } else if (action.equals(BluetoothMapClient.ACTION_MESSAGE_DELIVERED_SUCCESSFULLY)) {
                Logcat.d("Message delivered successfully");
            }
        }
    };
    

    /**
     * 蓝牙短信发送
     * 
     * @param context   
     * @param address    蓝牙设备对应的Address
     * @param recipients 收件人号码
     * @param message    短信内容
     */
    private void sendMessage(Context context, String address, Uri[] recipients, String message) {
        BluetoothDevice remoteDevice;
        try {
            remoteDevice = mAdapter.getRemoteDevice(address);
        } catch (java.lang.IllegalArgumentException e) {
            Logcat.d(e.toString());
            return;
        }
        if (mMapProfile != null) {
            Logcat.d("Sending reply");
            if (recipients == null) {
                Logcat.d("Recipients is null");
                return;
            }
            if (address == null) {
                Logcat.d("BluetoothDevice is null");
                return;
            }

            // 以下两个Intent设置后，短信发送成就会有对应的广播发送回来，若只需要一个，另一个可直接设置为null，简单看了下源码，目前未看到两个Intent的明显差异，后续再细究
            mSentIntent = PendingIntent.getBroadcast(context, 0, mSendIntent, PendingIntent.FLAG_ONE_SHOT);
            mDeliveredIntent = PendingIntent.getBroadcast(context, 0, mDeliveryIntent, PendingIntent.FLAG_ONE_SHOT);
            mMapProfile.sendMessage(remoteDevice, recipients, message, mSentIntent, mDeliveredIntent);
        }
    }
```


