---
layout: post
title:  "解决Android 应用targetSdkVersion小于24（Android N）运行在高版本设备无法全屏的BUG"
author: "陈宇瀚"
date:   2021-10-26 17:51:00 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android
---

## 前言
进行公司车机设备开发时，由于屏幕比例比较特殊（1920x720），导致部分应用显示时左侧和底部有很大的黑边，使用**dumpsys**分析黑边的**View**，移除后发现仍存在，后确定为低版本默认比例问题，耗费了几个小时，记录下这个问题。

### 出现问题的条件：
1. Android 应用：**targetSdkVersion** < **24** （Android N）；
2. 设备的分辨率宽高比大于**1.86**（**1.86**为Android低版本最大宽高比）；
3. Android 应用的**AndroidManifest.xml**内未设置**android:resizeableActivity**="true"属性；

## 问题解决方案
问题根据不同的情况有不同的解决方式
1. 若拥有应用源码，则可以直接将**targetSdkVersion**设置为大于等于**24**即可；
2. 若拥有应用源码，但不能修改**targetSdkVersion**版本，那么就在
应用的**AndroidManifest.xml**内设置**android:resizeableActivity**="true"属性；
3. 若拥有应用源码，但不能修改**targetSdkVersion**版本，但**compileSdkVersion=26**时，
可以尝试在的**AndroidManifest.xml**内设置**android:maxAspectRatio=""**="宽高比"属性；
4. 若没有应用源码，则需要尝试修改系统层来达到适配的作用，具体修改见如下代码：

### 系统层修改
**/frameworks/base/core/java/android/content/pm/PackageParser.java**
**PackageParser**在解析**APK**的**AndroidManifest.xml**文件时，有如下代码：
```java
public class PackageParser {
    ....
    private static final float DEFAULT_PRE_O_MAX_ASPECT_RATIO = 1.86f;
    ....
    private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
            throws XmlPullParserException, IOException {
        final ApplicationInfo ai = owner.applicationInfo;
        final String pkgName = owner.applicationInfo.packageName;
        
        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestApplication);
        ....
        
        // 这里会去判断xml内是否有设置android:resizeableActivity属性
        if (sa.hasValueOrEmpty(R.styleable.AndroidManifestApplication_resizeableActivity)) {
            if (sa.getBoolean(R.styleable.AndroidManifestApplication_resizeableActivity, true)) {
                // 设置该标志位后，应用的Activity可调整大小
                ai.privateFlags |= PRIVATE_FLAG_ACTIVITIES_RESIZE_MODE_RESIZEABLE;
            } else {
                // 设置该标志位后，应用的Activity不可调整大小
                ai.privateFlags |= PRIVATE_FLAG_ACTIVITIES_RESIZE_MODE_UNRESIZEABLE;
            }
        } else if (owner.applicationInfo.targetSdkVersion >= Build.VERSION_CODES.N) {
            // 若应用未设置android:resizeableActivity属性，且其targetSdkVersion >= Android N （24）
            // 设置该标志位后，应用的Activity可调整大小
            ai.privateFlags |= PRIVATE_FLAG_ACTIVITIES_RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION;
-       }
+        }else {
+            ai.privateFlags |= PRIVATE_FLAG_ACTIVITIES_RESIZE_MODE_RESIZEABLE;
+        }
      
    }
    
}
```
根据以上代码可以看到，默认情况下没有对未设置**android:resizeableActivity**属性，且**targetSdkVersion** < **24**的应用不会设置其**Activity**可调整大小，所以我们需要为这种情况加上对应的配置，即添加上述代码中的相应内容。修改后编译进设备，即可看到应用全屏显示了。

## 问题分析
除了上面分析到的没有**Activity**可调整大小标志位导致的无法全屏问题，还有一个是关于默认最大宽高比（**DEFAULT_PRE_O_MAX_ASPECT_RATIO**）的问题，通过修改该值也可以实现应用的全屏，不过有时候屏幕比例变化较大时该值可能要经常改动，有点麻烦。下面主要是从源码来分析下该值是如何起作用的。
**/frameworks/base/core/java/android/content/pm/PackageParser.java**
```
public class PackageParser {
    ....
    private static final float DEFAULT_PRE_O_MAX_ASPECT_RATIO = 1.86f;
    ....    
    private void setMaxAspectRatio(Package owner) {
        // 对于 O 之前的应用程序默认为 (1.86) 16.7:9 纵横比，对于 O 及更高版本则未设置。
        float maxAspectRatio = owner.applicationInfo.targetSdkVersion < O
                ? DEFAULT_PRE_O_MAX_ASPECT_RATIO : 0;

        if (owner.applicationInfo.maxAspectRatio != 0) {
            // 如果应用设置了android:maxAspectRatio 属性值，则使用应用程序所配置的最大宽高比。
            maxAspectRatio = owner.applicationInfo.maxAspectRatio;
        } else if (owner.mAppMetaData != null
                && owner.mAppMetaData.containsKey(METADATA_MAX_ASPECT_RATIO)) {
            // 未设置则设置为默认的1.86
            maxAspectRatio = owner.mAppMetaData.getFloat(METADATA_MAX_ASPECT_RATIO, maxAspectRatio);
        }

        for (Activity activity : owner.activities) {
            // If the max aspect ratio for the activity has already been set, skip.
            if (activity.hasMaxAspectRatio()) {
                continue;
            }

            // 这里可以获取activity设置的maxAspectRatio属性，若应用和activity都设置了，则以activity的优先
            final float activityAspectRatio = activity.metaData != null
                    ? activity.metaData.getFloat(METADATA_MAX_ASPECT_RATIO, maxAspectRatio)
                    : maxAspectRatio;
            // 将最大宽高比设置给activity
            activity.setMaxAspectRatio(activityAspectRatio);
        }
    }
}
```
以上就是为何应用无法全屏显示的原因，以及另外一种修改为何也能生效的原因。
