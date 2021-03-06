---
layout: post
title:  "Android 根据已装应用的签名限制其他应用安装"
author: "陈宇瀚"
date:   2018-04-26 16:41:42 +0800
categories: article
tags:
  - Android 
  - Framework
---

参考文章：https://blog.csdn.net/loongembedded/article/details/54090873
只有使用特定签名签的apk才可以安装，其他任何apk都不能安装

最好是应用预装一个使用对应签名的应用

需要修改的文件有以下几个
/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java

/frameworks/base/core/java/android/content/pmPackageManager.java

/packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallAppProgress.java

/packages/apps/PackageInstaller/res/values/string.xml

## 1 PackageManagerService.java
/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java
添加**getSignaturesByPackage**()方法
```java
//add start
private Signature[] getSignaturesByPackage(){
    Signature[] signatures = null;
    //已签名的已安装应用包名，与该应用签名不同的应用无法安装
    String packageName = "com.xxxx.xxxxx";
    PackageSetting ps = mSettings.mPackages.get(packageName);
    if (ps != null) {
        PackageParser.Package pkg = ps.pkg;
        if (pkg == null) {
            pkg = new PackageParser.Package(packageName);
            pkg.applicationInfo.packageName = packageName;
            pkg.applicationInfo.flags = ps.pkgFlags | ApplicationInfo.FLAG_IS_DATA_ONLY;
            pkg.applicationInfo.publicSourceDir = ps.resourcePathString;
            pkg.applicationInfo.sourceDir = ps.codePathString;
            pkg.applicationInfo.dataDir = getDataPathForPackage(packageName,    0).getPath();
            // pkg.applicationInfo.nativeLibraryDir = ps.nativeLibraryPathString;
    }
    
    PackageInfo packageInfo2 = generatePackageInfo(pkg, PackageManager.GET_SIGNATURES,  UserHandle.getCallingUserId());
    
    if(packageInfo2 != null){
    signatures = packageInfo2.signatures;
    }
    }
    mIsInstallApkFlag = false;
    return signatures;
}
//add end
```
在**scanPackageLI**方法下加入如下代码
```java
private PackageParser.Package scanPackageLI(PackageParser.Package pkg, int parseFlags , int scanFlags, long currentTime, UserHandle user) throws PackageManagerException {
    boolean success = false;
    //add start  
    //获得安装的应用的签名信息
    Signature[] originalSignatures = getSignaturesByPackage();
    if (originalSignatures != null) {
        if (compareSignatures(originalSignatures, pkg.mSignatures) !=   PackageManager.SIGNATURE_MATCH) {
            int mLastScanError = PackageManager.INSTALL_FAILED_INVALID_SIGNATURES;
            throw new PackageManagerException(mLastScanError,
            "禁止安装，签名不符");
    }
}
//add end
...
````
同时要import缺少的类。
## 2 PackageManager.java
/frameworks/base/core/java/android/content/pm/PackageManager.java
加入错误代码 **INSTALL_FAILED_INVALID_SIGNATURES**
```java
/** 
* Installation return code: this is passed to the {@link IPackageInstallObserver} by 
* {@link #installPackage(android.net.Uri, IPackageInstallObserver, int)} if 
* the installed package hasn't the expected signature 
*  @hide 
*/
public static final int INSTALL_FAILED_INVALID_SIGNATURES = -26;
```
## 3 InstallAppProgress.java
/packages/apps/PackageInstaller/src/com/android/packageinstaller/InstallAppProgress.java
加入针对添加的错误代码的处理
```java
private Handler mHandler = new Handler() {  
    public void handleMessage(Message msg) {  
        switch (msg.what) {  
            ...
            //add start
        } else if (msg.arg1 ==  PackageManager.INSTALL_FAILED_INVALID_SIGNATURES){
            // Generic error handling for all other error codes.  
            centerTextDrawable.setLevel(1);
            centerExplanationLabel = getExplanationFromErrorCode(msg.arg1);
            //centerTextLabel = R.string.install_failed_invalid_signature;  
            centerTextLabel = R.string.install_failed_invalid_signature;
            mLaunchButton.setVisibility(View.INVISIBLE);
            //add end
        }  else {
            // Generic error handling for all other error codes.
            centerTextDrawable.setLevel(1);
            centerExplanationLabel = getExplanationFromErrorCode(msg.arg1);
            centerTextLabel = R.string.install_failed;
            mLaunchButton.setVisibility(View.INVISIBLE);
        }
    }
}
```
## 4 strings.xml
/packages/apps/PackageInstaller/res/values/string.xml
添加错误信息**install_failed_invalid_signature**
```xml
<string name="install_failed_invalid_signature">禁止安装，签名不符</string>
```

