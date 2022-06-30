---
layout: post
title:  "Android AutoMotive系统预置GMS"
author: "陈宇瀚"
date:   2022-06-27 09:57:35 +0800
header-img: "img/img-head/img-head-automotive.jpg"
categories: article
tags:
  - Android
  - AutoMotive
---
## GMS介绍
**GMS**全称**为GoogleMobile Service**，即**谷歌移动服务**。**GMS**是Google开发并推动Android的动力，是谷歌程序运行的基础。

**GMS**提供有**GooglePlay**、**Search**、**Google语音**、**Gmail**、**Contact Sync**、**Calendar Sync**、**Talk**、**Google Maps**、**Street View**、**YouTube**、**Android Market**等服务，**GMS**为安卓上的谷歌公司系列应用提供支持
## GMS资源获取
由于**Google**官方未开放相关的服务，应用的资源下载，所以我们要选择其他方式获取，其中最为常用的是从[Open GApps](https://opengapps.org/)（https://opengapps.org/)网站上下载，**Open GApps**项目是一个开源项目，由非官方人员负责维护，主要给**自定义ROM**免费配备**GMS**，每晚（欧洲时间）更新一次Google相关的APP，包含适配各种架构，安卓版本的Google相关的服务，应用的资源文件。

### 版本的选择
当我们进入[Open GApps](https://opengapps.org/)后可看到如下的选项界面，比较清晰，根据需求进行选择下载：

![](/img/in_post/Open_Gapps.png)


针对不同的**Variant**包含的内容，网上已有介绍，这里也粘贴一下：

类型 | 描述
---|---
aroma | 与**super**版所包含的**GApps**相同，但是在**Recovery**中引入了图形化界面。
super | 包含了所有**GApps**，像韩语日语中文拼音中文注音输入法等。（请注意：如果你是用的是基于原生的ROM ，本版本会替换相机，通讯录等等所有有关应用）。
stock | 类似于**Google Pixel**出厂内置的**GApps**，相比**super**版少了其他语种的输入法以及**Google**地球等。
full | 与 **stock** 版所包含的内容相同，但此版本不会替换手机原本的应用.
mini | 包含基础的 **Google** 服务框架，以及一些影响力较大的**GApps**，相比**full**版去掉了**Docs**等应用。
micro | 包含基础的**Google**服务框架和 **Gmail** 等常见**GApps**。
nano | 包含基础的**Google**服务框架，但不会有其他不必要的**GApps**。
pico | 包含最迷你的**Google**服务框架，但由于框架并非完整，部分**GApps**可能无法运行
tvstock | 此软件包适用于**Android TV**设备。 它包括**Nexus Player**标配的所有**GApps**。
tvmini | 适用于**Android TV**设备，只包含一些影响力较大的**GApps**。

这里我建议下一个**pico**版本和**super**，导入**pico**相关的内容后可以正常运行，根据需要去**super**版本内拿其余的**apk**。

### 资源包的解析
下载后解压可看到如下目录形式的文件结构：

![](/img/in_post/Gapps_zip.png)

其中
- **Core**是核心应用和资源的目录，也就是我们需要预置的部分；
- **GApps**是其他的原生apk，根据需求判断是否需要添加；
- **Optional**是一些可选的库，可一并导入进系统；
- **tar-arm**、**unzip-arm**、**zip-arm**是一些解压缩的工具；
- 其余的文件还包含一些安装脚本，可自动将相关的内容安装至我们的手机中，由于我们是预置进系统内，所以不需要关注这部分的东西；

由于提取相关的**apk**和资源手动弄起来比较慢，这里提供一个我自己写的脚本，放到解压后的目录的根目录，然后执行脚本就可以将所有我们需要的东西整理输出到一个**out**目录，我们再根据目录的结构预置进我们的系统即可；

```sh
basepath=$(cd `dirname $0`; pwd)

mkdir -p out/priv-app
mkdir -p out/common
mkdir -p out/app

cd $basepath/Core && find . -name '*.lz' | xargs -n1 lzip -d && ls *.tar | xargs -n1 tar -xvf && cd ..
cd $basepath/GApps && find . -name '*.lz' | xargs -n1 lzip -d && ls *.tar | xargs -n1 tar -xvf && cd ..
cd $basepath/Optional && find . -name '*.lz' | xargs -n1 lzip -d && ls *.tar | xargs -n1 tar -xvf && cd ..

find -name priv-app | xargs -i cp -rf {} out/
find -name common | xargs -i cp -rf {} out/
find -name app | xargs -i cp -rf {} out/

cp g.prop out/common/etc/
```

执行后对应的**out**目录的内部结构如下：

![Gapps_dir.png](/img/in_post/Gapps_dir.png)

这里介绍其中几个比较重要的应用
1. Google服务框架: **GoogleServicesFramework**;
2. Google核心服务:**PrebuiltGmsCore**；
3. GooglePlay:**Phonesky**；

*注意：我们预置相关的应用，编写相应Android.mk时，保留应用原有的签名，不要使用我们的系统签名去覆盖它*

## GMS预置

### GMS资源预置
#### priv-app的预置
预置**Android.mk**的编写，由于应用较多，所以这里以一个应用为例，其他的依样画葫芦即可：
```mk
##############GoogleServicesFramework##################
include $(CLEAR_VARS)
#
## Module name should match apk name to be installed
LOCAL_MODULE := GoogleServicesFramework
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := GoogleServicesFramework.apk
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_CERTIFICATE := PRESIGNED
LOCAL_DEX_PREOPT := true
LOCAL_PRIVILEGED_MODULE := true
include $(BUILD_PREBUILT)
```
#### app的预置
同样的以一个应用为例
```mk
##############YouTube##################
include $(CLEAR_VARS)
## Module name should match apk name to be installed
LOCAL_MODULE := YouTube
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := YouTube.apk
LOCAL_MODULE_CLASS := APPS
LOCAL_MODULE_SUFFIX := $(COMMON_ANDROID_PACKAGE_SUFFIX)
LOCAL_CERTIFICATE := PRESIGNED
LOCAL_DEX_PREOPT := false
LOCAL_OVERRIDES_PACKAGES += \
                YouTubeLeanback
include $(BUILD_PREBUILT)
```
#### common目录的预置
**common**目录内部的文件只需要拷贝进**system**目录下即可，根据需要写在对应的**mk**内
```mk
    # google config
    PRODUCT_COPY_FILES += $(call find-copy-subdir-files,*,xxxxxx/common,/system)
```

当所有都配置好之后编译烧录即可，当然前提是系统分区的空间充足，不然可能会编译不通过；
之后烧录进设备，联网之后会出现设备未认证的提示，根据提示进行认证操作即可，这部分网上教程很多，这里也不展开。

### AutoMotive系统GooglePlay预置问题
预置时需要注意除了**GooglePlay**以外基本都不太会受设备类型的影响，而**GooglePlay**根据不同的设备类型有不同的应用**apk**，内部可搜索的应用数量也跟设备内支持的**feature**有关（这部门的内容比较细，没有研究的很深入，后续有机会再整理）。虽然有不同的**apk**，但它们对应的包名都为**com.android.vending**。

根据设备类型可分为如下几种：

类型 | 描述
---|---
手机版**Phonesky** | 应用类型最多，最丰富的版本，我们所需要的也是这种版本。
电视版**Tubesky** | 电视版本的谷歌商店，经过测试，触摸无法控制，需要靠外部的输入设备(如键盘，鼠标，遥控器)等才可控制应用内部的内容，对触摸设备没做支持？所以也不考虑使用。
车机(AutoMotive)版 | 目前**AutoMotive**版本没有对应的**apk**，我所用的**apk**为安装手机版**GooglePlay**，然后由于系统存在**AutoMotive**的**Feature**，所以**GooglePlay**自动更新匹配我的设备，从而得来的**apk**(/data/app/com.android.vendingxxxx 目录下），这种方式拿到的**apk**是被分包处理后的，若要预置到系统中，暂时还没找到合适的方式；若只预置**base.apk**会出现某些资源缺失的问题，暂时不太可用。
手表版 | 为穿戴类安卓设备定制的谷歌商店，我这边没有研究，也就不展开。

那么如何在我们的车机预置手机版的**GooglePlay**：
### 预置不同于设备类型的GooglePlay
前面有提到**GooglePlay**受设备类型的影响，正常情况我们将手机版**GooglePlay**安装在**车机**设备上是无法运行的，打开**GooglePlay**会有如下的弹窗：

![google_play_store_error.png](/img/in_post/google_play_store_error.png)

此时其实后台后自动帮我们将**GooglePlay**更新到车机版本，等待更新完成之后也能打开。

![automotve_googleplay.png](/img/in_post/automotve_googleplay.png)

车机版**GooglePlay**界面

可以看到车机版的应用数量较少，可能不太符合我们的需求，所以需要兼容运行手机版**GooglePlay**。

通过**jadx-gui**对手机版**GooglePlay**即**Phonesky.apk**进行反编译，搜索**automotive**我们可以看到内部有一些针对**automotive**, **watch**,**tv**等**feature**类型的判断，虽然无法具体的看到是什么逻辑，但是我们可以大致猜测出**GooglePlay**的内容和启动跟**feature**有一定的关系，这部分在官网也有一定的介绍。

![jadx_1.png](/img/in_post/jadx_1.png)

这里我们取其中一个代码来看一下：
```java
package defpackage;

import android.content.Context;
import android.content.pm.ApplicationInfo;
import android.os.Build;
import com.google.firebase.FirebaseCommonRegistrar;

/* compiled from: PG */
/* renamed from: aepj  reason: default package */
/* loaded from: classes.dex */
public final /* synthetic */ class aepj implements aetv {
    private final /* synthetic */ int e;
    public static final /* synthetic */ aepj d = new aepj(3);
    public static final /* synthetic */ aepj c = new aepj(2);
    public static final /* synthetic */ aepj b = new aepj(1);
    public static final /* synthetic */ aepj a = new aepj(0);

    private /* synthetic */ aepj(int i) {
        this.e = i;
    }

    @Override // defpackage.aetv
    public final String a(Object obj) {
        int i = this.e;
        if (i == 0) {
            ApplicationInfo applicationInfo = ((Context) obj).getApplicationInfo();
            return (applicationInfo == null || Build.VERSION.SDK_INT < 24) ? "" : String.valueOf(applicationInfo.minSdkVersion);
        } else if (i == 1) {
            ApplicationInfo applicationInfo2 = ((Context) obj).getApplicationInfo();
            return applicationInfo2 != null ? String.valueOf(applicationInfo2.targetSdkVersion) : "";
        } else if (i != 2) {
            Context context = (Context) obj;
            String installerPackageName = context.getPackageManager().getInstallerPackageName(context.getPackageName());
            return installerPackageName != null ? FirebaseCommonRegistrar.a(installerPackageName) : "";
        } else {
            Context context2 = (Context) obj;
            return context2.getPackageManager().hasSystemFeature("android.hardware.type.television") ?
            "tv" : context2.getPackageManager().hasSystemFeature("android.hardware.type.watch") ?
            "watch" : (Build.VERSION.SDK_INT < 23 || !context2.getPackageManager().hasSystemFeature("android.hardware.type.automotive")) ?
            (Build.VERSION.SDK_INT < 26 || !context2.getPackageManager().hasSystemFeature("android.hardware.type.embedded")) ?
            "" : "embedded" : "auto";
        }
    }
}
```
可以看到代码内通过**PackageManager**的**hasSystemFeature**方法去判断设备的类型，并返回对应的类型字符串，在源码中找到对应的方法和文件(**framework/base/core/java/android/app/ApplicationPackageManager.java**)，结合内部的方法来看，除了**hasSystemFeature**，还有一个获得系统所有支持的**Feature**的方法：**getSystemAvailableFeatures**，我们同样在反编译后的代码中搜索。

![jadx_2.png](/img/in_post/jadx_2.png)

看到同样有调用，那么我们就都一起改掉。
#### GMS适配修改方案
**framework/base/core/java/android/app/ApplicationPackageManager.java**
```java
public class ApplicationPackageManager extends PackageManager {
    ....
    public FeatureInfo[] getSystemAvailableFeatures() {
        try {
            ParceledListSlice<FeatureInfo> parceledList =
                    mPM.getSystemAvailableFeatures();
            if (parceledList == null) {
                return new FeatureInfo[0];
            }
            final List<FeatureInfo> list = parceledList.getList();
            // add start ——————————————————————
            if ("com.android.vending".equals(mContext.getPackageName()) || "com.google.android.gms".equals(mContext.getPackageName())) {
                // googleplay and gms service rmove automotvie feature
                for (int i = 0; i < list.size(); i++) {
                    if(PackageManager.FEATURE_AUTOMOTIVE.equals(list.get(i).name)) {
                        Log.i(TAG, "getSystemAvailableFeatures mContext.getPackageName() = " + mContext.getPackageName() + " i = " + i);
                        list.remove(i);
                    }
                }
            }
            // add end ——————————————————————
            final FeatureInfo[] res = new FeatureInfo[list.size()];
            for (int i = 0; i < res.length; i++) {
                res[i] = list.get(i);
            }
            return res;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

    @Override
    public boolean hasSystemFeature(String name) {
        return hasSystemFeature(name, 0);
    }

    @Override
    public boolean hasSystemFeature(String name, int version) {
        // add start ——————————————————————
        if ("com.android.vending".equals(mContext.getPackageName()) || "com.google.android.gms".equals(mContext.getPackageName())) {
            // googleplay and gms service rmove automotvie feature
            Log.i(TAG, "hasSystemFeature mGmsFeatureType = " + mGmsFeatureType + " name = "+ name);
            if (PackageManager.FEATURE_AUTOMOTIVE.equals(name)) {
                return false;
            }
        }
        // add end ——————————————————————
        try {
            return mPM.hasSystemFeature(name, version);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
}
```
逻辑很简单，就是根据对应的包名移除对应的**feature**。之后再次编译操作，就可以成功在我们的设备上运行手机版的**GooglePlay**了，运行效果如下:

![googleplay.png](/img/in_post/googleplay.png)

## GMS一次性认证方案
前面有提到，我们的设备烧录之后都需要根据**Google**官方流程对设备进行验证，如果是大量出货的设备，若没有与**Google**合作，那么每一台都需要走一遍验证流程，非常耗时和繁琐，所以我们需要通过一些方式跳过这个认证的过程，保证我们的设备在烧录之后不需要进行认证也可以使用谷歌服务，下面介绍一下具体的实现方式。
### GMS认证跳过方案
根据官网的认证操作流程，我们知道注册认证的**android_id**的获取方式如下： 
```
adb root 
abd shell
sqlite3 /data/data/com.google.android.gsf/databases/gservices.db
select * from main where name = "android_id";
```
所以我们只需要保证我们每台设备的**android_id**都是我们已经认证过的设备**id**即可。
针对这一方案，可用的实现方案如下：
1. 将已经认证设备的**Google服务**相关的初始数据库 **/data/data/com.google.android.gsf/databases** 和 **/data/data/com.google.android.gms/databases** 复制出来；
2. 将已认证的**Google服务**数据库打包进系统固件中；
3. 编写脚本，第一次开机时启动，使用已认证的**Google服务**数据库覆盖设备内对应的数据库；
4. 检查设备**android_id**，确认成功替换，且没有设备待认证的通知；


