---
layout: post
title:  "实现一个简单开机自启服务"
author: "陈宇瀚"
date:   2021-08-03 18:00:00 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android
  - 开机服务
---
# 实现一个简单开机自启服务

作为一个简单的总结，以我自己开发的测试服务testautostart为例，主要分为以下四个部分
1. 在platform/vendor/xxxx/xxxx目录下创建一个自定服务的文件夹testautostart，用于存放程序代码和makefile文件和依赖库等东西；
2. 编写自定Service代码和makefile文件Andorid.mk；
3. 通过mm编译后，在out/target/product/car_u804mc/system/bin下生成的可执行文件通过adb推入设备的/system/bin/目录下；
4. 调用可执行文件，成功执行了Native Service的main方法；
5. 编写自启动**rc**文件，在平台源码**device**目录中新增测试服务编译；
6. 设置SELinux相关权限

## 编写测试服务testautostart
目录结构
/platform/vendor/xxxx/xxxx/testautostart

├── testautostart.cpp ：具体服务实现

└──  Android.bp ：bp编译文件


**testautostart.cpp**文件
```c++
#include <stdio.h>
#include <unistd.h> 
#include <android/log.h>
#define LOG_TAG    "native-log"
#define LOGI(...)  __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
#define LOGE(...)  __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)
#define LOGD(...)  __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)
#define LOGW(...)  __android_log_print(ANDROID_LOG_WARN, LOG_TAG, __VA_ARGS__)
#define LOGV(...)  __android_log_print(ANDROID_LOG_VERBOSE, LOG_TAG, __VA_ARGS__)
int main()
{
    while(1){
 	   LOGI("Hello world! Native\n");
    	   sleep(1);
	}
	return 0;
}
```
**Android.bp**文件
```mk
cc_binary {
    srcs: ["testautostart.cpp"],
    shared_libs: ["liblog"],
    name: "testautostart",
    init_rc: ["testautostart.rc"]
}
```
进行编译，在设备内容过adb执行结果如下
![image](/img/in_post/realize_simple_bootup_service_one.png)

## 编写自启动rc文件
在上述目录内新增**testautostart.rc**文件
```rc
service testautostart /system/bin/testautostart
calss main
user system
group system
```
这里的**service**代表要定义一个**service**，名字为**testautostart**，之后的目录对应的是这个**service**对应的可执行文件在系统中的位置（源码编译后在系统的位置）。**user**和**group**代表服务归属哪个用户和用户组。

### 添加服务编译配置
以我手头的MT2712举例，在**platform/device/mediatek/mt2712/device.mk**中配置：
```mk
#testautostart
PRODUCT_PACKAGES +=testautostart
```
添加后编译固件时就会将我们的测试服务编译进系统

## SELinux配置
### 配置文件安全上下文
在**platform/device/mediatek/mt2712/sepolicy/non_plat/file_context**
```
/system/bin/testautostart u:object_r:testautostart_exec:s0
```
**u:object_r:testautostart:s0**代表了其SELinux用户、SELinux角色、类型和安全级别分别为**u、object_r、testautostart、s0**

### 编译安全策略te文件
在**platform/device/mediatek/mt2712/sepolicy/non_plat**目录下新增**testautostart.te**文件，内容如下:
```
type testautostart, domain;     //赋予testautostart domain属性，testautostart 属于domain集合
type testautostart_exec, exec_type, file_type; //添加testautostart的可执行文件定义

init_daemon_domain(testautostart)//把testautostart_exec（客体）转换成testautostart（进程域）？不太理解，之后再研究研究

typeattribute testautostart coredomain;//typeattribute用于关联属性，声明后testautostart具有coredomain属性
```
编译后烧录，确认rc文件已经编译生成在**/system/etc/init/** 目录下(不同公司源码或者版本不同目录都有可能不同)，通过**ps -A | grep testautostart**查看到服务没启动

### SElinux 未通过权限配置
此时我们的服务应该是未启动起来的，我们要通过
```
dmesg | grep avc
或
logcat -v time *:V | grep avc
```
了解到缺少的权限，比如我的测试文件输出的日志信息如下：
[图片上传失败...(image-797db6-1627988486795)]
根据selinux的语法：

【缺少什么权限】avc: denied ：getattr

【谁缺少权限】 scontext:u:r:shell:s0

【对什么缺少权限】 tcontext：u:object_r:testautostart:s0

【对什么类型的文件，即tcontext类型】 tclass：file

那么我们在我们的te文件中添加权限：
```
allow shell testautostart:file { getattr };
```
再次编译后烧录，之后就可以看到服务正在运行:
![image](/img/in_post/realize_simple_bootup_service_two.png)

![image](/img/in_post/realize_simple_bootup_service_three.png)

### 关于SELinux一个方便的调试方法
我们可以在**te**文件内加入如下语句：
```
#hal_ipod is also permissive to permit setenforce.
permissive testautostart;
```
该语句可以让**SELinux**针对我们的**testautostart**使用**Permissive**模式执行，这样就不需要一个一个排查avc，查看缺少的权限一个个添加，这种方式我们的**testautostart**可以直接运行通过，并同样通过过滤avc，直接一次性看到所有缺少的权限，方便我们一次性添加进tew文件中

