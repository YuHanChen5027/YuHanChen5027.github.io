---
layout: post
title:  "Android persistent属性原理分析"
author: "陈宇瀚"
date:   2021-09-17 17:45:00 +0800
header-img: "img/img-head/img-head-boot-process.jpg"
categories: article
tags:
  - Android
  - 开机流程
---
# Android persistent属性原理分析

[TOC]
*以下代码基于Android9.0*

# persistent属性的定义
开发**系统应用**时，有时我们需要应用常驻，被杀死后能够自动重启，此时就需要使用到**persistent**属性,
下面是关于该属性在framework层的定义，属性的定义位于
**persistent**属性定义在**platform/frameworks/base/core/res/res/values/attrs_manifest.xml**文件内：
```xml
    <!-- 控制应用程序是否处于特殊持久模式的标志，通常情况下不应该被使用，该标志位能够保证应用程序一直运行 -->
    <attr name="persistent" format="boolean" />
```

# persistent属性的使用
开发**系统应用**时，有时我们需要应用常驻，被杀死后能够自动重启，此时我们就要在应用的**AndroidManifest.xml**中设置
```xml
android:persistent="true"
```
设置后应用就具备了以下两个特性：
1. 系统启动时该应用也会启动
2. 应用被杀死后，系统会重启该应用

# persistent属性的原理

## persistent属性的解析
当我们应用安装或启动的过程中，会对**AndroidManifest.xml**进行解析，解析相关的代码位于
**platform/frameworks/base/core/java/android/content/pm/PackageParser.java**：
```java
public class PackageParser {
    ....
    private boolean parseBaseApplication(Package owner, Resources res,
            XmlResourceParser parser, int flags, String[] outError)
            throws XmlPullParserException, IOException {
                final ApplicationInfo ai = owner.applicationInfo;
                final String pkgName = owner.applicationInfo.packageName;
        
                // 获取 AndroidManifest 的属性
                TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifestApplication);
                ....
                // 获取所设置的 persistent 值，默认为 false
                if (sa.getBoolean(
                    com.android.internal.R.styleable.AndroidManifestApplication_persistent,
                    false)) {
                // 检查应用是否支持这个权限
                final String requiredFeature = sa.getNonResourceString(com.android.internal.R.styleable
                    .AndroidManifestApplication_persistentWhenFeatureAvailable);
                    if (requiredFeature == null || mCallback.hasFeature(requiredFeature)) {
                        // 将 persistent 的 flag 设置到应用信息中
                        ai.flags |= ApplicationInfo.FLAG_PERSISTENT;
                    }
                }
                ....
    }
    ....
    
}
```
解析后就会将应用的各种信息保存在**PKMS**中的一个存储所有应用信息的一个**Map**中，其中设置了**persistent**的应用就包含了**ApplicationInfo.FLAG_PERSISTENT**标志位，之后在系统的启动过程中就会根据这个标志位控制应用的启动。

## persistent应用的启动

persistent应用的启动发生在**AMS**的**systemReady**方法内，这一部分通过**PKMS**获取到所有**persistent为true**的应用列表，之后对列表进行遍历，通过**addAppLocked**方法将应用一个个启动起来。
对应代码的位于
**/platform/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java**
```java
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    ....
    // 该静态常量用于判断应用是否persistent应用
    private static final int PERSISTENT_MASK =
            ApplicationInfo.FLAG_SYSTEM|ApplicationInfo.FLAG_PERSISTENT;
            
    // 当系统服务启动时，AMS执行systemReady方法
    public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
            ....
            synchronized (this) {
                // 1.启动所有persistent属性为true的应用
                startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);
                ....
            }
            
    }
    ....
    void startPersistentApps(int matchFlags) {
        if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL) return;

        synchronized (this) {
            try {
                // 从PKMS的应用MAP中拿到所有具有FLAG_PERSISTENT标志位的应用
                final List<ApplicationInfo> apps = AppGlobals.getPackageManager()
                        .getPersistentApplications(STOCK_PM_FLAGS | matchFlags).getList();
                for (ApplicationInfo app : apps) {
                    // 过滤掉包名为android的应用
                    if (!"android".equals(app.packageName)) {
                        //2. 添加并启动该APP进程
                        addAppLocked(app, null, false, null /* ABI override */);
                    }
                }
            } catch (RemoteException ex) {
            }
        }
    }
            
}
```
### 获得所有具有FLAG_PERSISTENT标志位的应用
**platform/frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java**
```java
public class PackageManagerService extends IPackageManager.Stub
        implements PackageSender {
    ....
    // 存放所有应用的信息的Map
    final ArrayMap<String, PackageParser.Package> mPackages =
            new ArrayMap<String, PackageParser.Package>();
    ....
        
    @Override
    public @NonNull ParceledListSlice<ApplicationInfo> getPersistentApplications(int flags) {
        if (getInstantAppPackageName(Binder.getCallingUid()) != null) {
            return ParceledListSlice.emptyList();
        }
        return new ParceledListSlice<>(getPersistentApplicationsInternal(flags));
    }

    private @NonNull List<ApplicationInfo> getPersistentApplicationsInternal(int flags) {
        final ArrayList<ApplicationInfo> finalList = new ArrayList<ApplicationInfo>();

        synchronized (mPackages) {
            final Iterator<PackageParser.Package> i = mPackages.values().iterator();
            final int userId = UserHandle.getCallingUserId();\
            // 遍历mPackages
            while (i.hasNext()) {
                final PackageParser.Package p = i.next();
                if (p.applicationInfo == null) continue;

                ....
                // 判断应用信息是否有FLAG_PERSISTENT标志位
                if ((p.applicationInfo.flags & ApplicationInfo.FLAG_PERSISTENT) != 0
                        && (!mSafeMode || isSystemApp(p))
                        && (matchesUnaware || matchesAware)) {
                    PackageSetting ps = mSettings.mPackages.get(p.packageName);
                    if (ps != null) {
                        ApplicationInfo ai = PackageParser.generateApplicationInfo(p, flags,
                                ps.readUserState(userId), userId);
                        if (ai != null) {
                            // 将应用的ApplicationInfo添加进列表
                            finalList.add(ai);
                        }
                    }
                }
            }
        }

        return finalList;
    }
}
```
### 添加并启动应用进程
**AMS**内有一个存放所有正在启动的**persistent应用**的**List**：**mPersistentStartingProcesses**，后续在**启动应用**和**重启应用**时都会使用到该**List**。
```java
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
        
    /**
     * 正在启动的persistent应用程序列表。
     */
    final ArrayList<ProcessRecord> mPersistentStartingProcesses = new ArrayList<ProcessRecord>();
    
    ....
    final ProcessRecord addAppLocked(ApplicationInfo info, String customProcess, boolean isolated,
            String abiOverride) {
        return addAppLocked(info, customProcess, isolated, false /* disableHiddenApiChecks */,
                abiOverride);
    }
    
 final ProcessRecord addAppLocked(ApplicationInfo info, String customProcess, boolean isolated,
            boolean disableHiddenApiChecks, String abiOverride) {
        // ProcessRecord是用于描述进程的数据结构
        ProcessRecord app;
        // 传入的isolated为false
        if (!isolated) {
            // 第一次启动，这里查找应用所在的进程返回都为null
            app = getProcessRecordLocked(customProcess != null ? customProcess : info.processName,
                    info.uid, true);
        } else {
            app = null;
        }
        if (app == null) {
            // 为应用创建ProcessRecord
            app = newProcessRecordLocked(info, customProcess, isolated, 0);
            updateLruProcessLocked(app, false, null);
            updateOomAdjLocked();
        }
        ....
        
        if ((info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
            app.persistent = true;
            app.maxAdj = ProcessList.PERSISTENT_PROC_ADJ;
        }
        // 整个启动过程是异步的，所以这里仍需要判断应用线程是否为null，同时判断应用是否正在启动中（在mPersistentStartingProcesses列表内）
        if (app.thread == null && mPersistentStartingProcesses.indexOf(app) < 0) {
            // 应用没有在启动，则将应用添加到mPersistentStartingProcesses列表
            mPersistentStartingProcesses.add(app);
            // 启动应用
            startProcessLocked(app, "added application",
                    customProcess != null ? customProcess : app.processName, disableHiddenApiChecks,
                    abiOverride);
        }

        return app;
    }
    ....
}
```
系统启动时应用的**ProcessRecord**都未创建，所以在**addAppLocked**内首先通过**newProcessRecordLocked**为应用程序创建**ProcessRecord**，之后调用**startProcessLocked**来启动应用程序，启动的过程不是这部分的重点，就不再详细说明。
## persistent应用启动完成
当应用启动完毕后就会调用到**ActivityThread.java**内的**attach**方法，方法内调用了**AMS**的**attachApplication**方法，之后再到**attachApplicationLocked**，该方法内部就会将应用移除**mPersistentStartingProcesses**列表，表明应用启动完毕，同时为应用并注册一个死亡监听器**AppDeathRecipient**，用于应用被异常杀死后的重启。
```java
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
        ....    

        private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
            ....
            final String processName = app.processName;
            try {
                AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
                thread.asBinder().linkToDeath(adr, 0);
                app.deathRecipient = adr;
            } catch (RemoteException e) {
                app.resetPackageList(mProcessStats);
                //如果注册死亡接收器失败，也会重新启动App进程
                startProcessLocked(app, "link fail", processName);
                return false;
            }
            ....
            // 将应用移除正在启动的持久性应用列表
            mPersistentStartingProcesses.remove(app);
            ....
        }
}
```
## persistent应用死亡后重启
当应用被杀死后，就会调用死亡接收器**AppDeathRecipient**的**binderDied**方法，方法内根据应用是否是**persistent应用**来控制是否重启，整个流程如下：
```java
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    ....   
    private final class AppDeathRecipient implements IBinder.DeathRecipient {
        final ProcessRecord mApp;
        final int mPid;
        final IApplicationThread mAppThread;

        AppDeathRecipient(ProcessRecord app, int pid,
                IApplicationThread thread) {
            mApp = app;
            mPid = pid;
            mAppThread = thread;
        }

        @Override
        public void binderDied() {
            synchronized(ActivityManagerService.this) {
                // 1. 调用appDiedLocked
                appDiedLocked(mApp, mPid, mAppThread, true);
            }
        }
    }
    
    final void appDiedLocked(ProcessRecord app, int pid, IApplicationThread thread,
            boolean fromBinderDied) {
        ....
        if (app.pid == pid && app.thread != null &&
                app.thread.asBinder() == thread.asBinder()) {
            ....
            // 2. 调用appDiedLocked
            handleAppDiedLocked(app, false, true);
            ....
        }
        ....
    }
    
    private final void handleAppDiedLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart) {
        int pid = app.pid;
        // 3. 调用appDiedLocked
        boolean kept = cleanUpApplicationRecordLocked(app, restarting, allowRestart, -1,
                false /*replacingPid*/);
        ....
    }
 
    private final boolean cleanUpApplicationRecordLocked(ProcessRecord app,
            boolean restarting, boolean allowRestart, int index, boolean replacingPid) {
        ....
        // 判断应用是否是persistent应用
        if (!app.persistent || app.isolated) {
            // 如果不是persistent应用，则直接被清理掉
            ....
        } else if (!app.removed) {
            // 如果是persistent应用，则保留相应的信息
            // 判断其是否在待启动应用程序mPersistentStartingProcesses列表
            // 不在的话则添加，并设置restart为true
            if (mPersistentStartingProcesses.indexOf(app) < 0) {
                mPersistentStartingProcesses.add(app);
                restart = true;
            }
        }
        
        if (restart && !app.isolated) {
            // 重新启动我们的应用进程
            if (index < 0) {
                ProcessList.remove(app.pid);
            }
            addProcessNameLocked(app);
            app.pendingStart = false;
            // 启动应用程序进程
            startProcessLocked(app, "restart", app.processName);
            return true;
        } else if (app.pid > 0 && app.pid != MY_PID) {
            // 不需要重启的应用
            boolean removed;
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.remove(app.pid);
                mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
            }
            mBatteryStatsService.noteProcessFinish(app.processName, app.info.uid);
            if (app.isolated) {
                mBatteryStatsService.removeIsolatedUid(app.uid, app.info.uid);
            }
            app.setPid(0);
        }
    }
}
```
经过这个过程之后，**persistent**属性为**true**的应用程序进程就会被重启。

