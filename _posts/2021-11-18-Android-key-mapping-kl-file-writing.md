---
layout: post
title:  "Android 按键映射kl文件编写简析"
author: "陈宇瀚"
date:   2021-11-18 13:39:58 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android
---
*以下内容需要在驱动正常的情况下进行*
## kl文件
**kl**（**key layout**）文件是一个映射文件，是**标准linux**与**anroid键值**映射文件，**kl文件**可以有很多个，但是它有一个使用优先级：
```
/system/usr/keylayout/Vendor_XXXX_Product_XXXX_Version_XXXX.kl  
/system/usr/keylayout/Vendor_XXXX_Product_XXXX.kl  
/system/usr/keylayout/DEVICE_NAME.kl  
/data/system/devices/keylayout/Vendor_XXXX_Product_XXXX_Version_XXXX.kl  
/data/system/devices/keylayout/Vendor_XXXX_Product_XXXX.kl  
/data/system/devices/keylayout/DEVICE_NAME.kl  
/system/usr/keylayout/Generic.kl  
/data/system/devices/keylayout/Generic.kl 
```
适配一个新的输入设备，我们需要知道**vendor id**和**product id**。根据这两个**id**创建对应名称的**kl**文件，然后传入设备的对应目录，重启即可看到效果。

### kl文件命名含义
kl文件在源码内的目录：**/frameworks/base/data/keyboards/**

![1.png](/img/in_post/kl_files.png)

目录下的**kl文件**非常多，这里以**Vendor_0b05_Product_4500.kl**为例：

**Vendor_0b05** ：表示生产商代码是**0b05**

**Product_4500** ：表示产品型号为**4500**

之后再跟输入设备的对应**id**进行匹配，就可以知道该**kl文件**所对应的设备。

### 获得输入设备和按键信息
实际开发中我们可以通过**getevent**获取到输入设备的**vendor id**，**product id**和**按键事件值**
以**U918A**面板上的按键为例，通过**adb shell**连接后输入**getevent**，然后点击屏幕左侧面板上第一个的按键有如下输出：
![2.png](/img/in_post/kl_event.png)


以上的输出分为两个部分，**1**代表当前设备上的输入设备，**2**是点击按键后产生的输出。

可以看到面板按键对应的输入设备节点为 **/dev/input/event2** 。

之后通过**getevent -i /dev/input/event2**查看该设备的**vendor id**和**product id**。输出结果如下：

![3.png](/img/in_post/kl_event2.png)


此时我们就可以根据对应的**vendor id**和**product id**创建自己的**kl文件**，然后传入设备中验证，首先去源码对应的目录中找到对应的**kl文件**来增加按键映射，即**Vendor_0416_Product_038f.kl**文件，如果没有该文件，那么证明系统针对该设备的按键做特殊处理，之后就会根据上述所说的使用优先级去加载其他的**kl文件**。这里我们新建一个**Vendor_0416_Product_038f.kl**文件。然后根据分析的按键信息来往里面添加内容。
### 按键信息分析
![4.png](/img/in_post/kl_info.png)

以刚才的输出图中的**2**为例：
我们每次按键会有四个输出，前两行为**按下**，后两行为**抬起**，**0001**指按键（也存在其他设备类型，这里我们不关心），**0066**是对应的**十六进制按键值**，这里就是**驱动**所设置的**按键值**，可以去找驱动提供头文件查看该值所对应的**按键名称**。

末尾的部分，**00000001**为按下，**00000000**为抬起。

在驱动提供的按键值头文件中，**0066**转成**十进制**为**102**,在文件内对应按键如下:
```c++
#define KEY_HOME		102
```
然后在我们默认的**kl文件**，即**Generic.kl**中找到**102**对应**Android**中的按键值如下：
看下**kl**文件中**key 102**对应的功能值
```
key 102   MOVE_HOME
```
这里即将**驱动**上报的**KEY_HOME**转成了**Android**的**MOVE_HOME**按键，**Android**对应的**按键值列表头文件**目录为：
**/frameworks/native/include/android/keycodes.h**，如果我们要修改**驱动层**上报的按键在**Android**所对应的按键值，那么就可以在该头文件查找对应的按键，然后在**kl文件**进行配置。
### kl文件编写
之前我们根据输入设备创建了一个**Vendor_0416_Product_038f.kl**文件，接下来要根据上面的分析来进行该文件的编写，
以刚才的按键为例，若我不想该按键对应**MOVE_HOME**，想要它直接对应**HOME**按键，那么就要在**kl文件**内添加如下语句：

```
# 十六进制按键值
key 0x66 HOME   
or
# 十进制按键值
key 102 HOME    
```


然后将该文件推入设备**system/usr/keylayout/**，重启点击即可看到按键值的变化。在应用内可以通过**onKeyDown**等方法来监听按键值。同样的我们也可以新建按键值，然后分别在**keycodes.h**和 **frameworkbase/core/res/res/values/attrs.xml** ，**frameworks/base/core/java/android/view/KeyEvent.java**、**framework/native/include/input/InputEventLabels.h**、**kl文件**添加按键值和映射， 然后在应用内监听。

下面是我的设备中其他按键的映射修改，分别映射到了静音，音量减和音量加，语音助手几个**Android**按键上：
```
key 113 VOLUME_MUTE
key 114 VOLUME_DOWN
key 115 VOLUME_UP
key 582 VOICE_ASSIST
```

