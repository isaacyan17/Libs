# Acitivity Launch 

![](https://cdn.nlark.com/yuque/0/2021/jpeg/1227097/1637831882832-d2422729-b170-448f-ac1f-d1ab291cf26e.jpeg)
## 简介
Launcher启动Activity/app的简易时序图

[点击查看【processon】](https://www.processon.com/view/link/619fa24d1e0853431b12b64d)


## 1. Launcher通知AMS

```java
/*
/android/app/Activity.java
 源码,Launcher继承 Activity
*/

  @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

// 跟进
	 @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        ...
        
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            startActivityForResult(intent, -1);
        }
    }

//startActivityForResult

  public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
      //分析一
      //mParent代表当前类的父类,这个值从attach方法中传递过来
      //看完后面的流程,我们会知道,attach是ActivityThread启动以后,
      //执行performLaunchActivity时调用的.
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            
            //分析二, execStartActivity
            ...
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }

/*
	分析一  mParent
    /ActivityThread.performLaunchActivity
   
*/
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

		省略其他代码	
        ...
                //完成activity的一些重要数据的初始化
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken);
       
        //这里我们看到mParent = r.parent
        //r = ActivityClientRecord   它是ActivityThread的一个静态类，它代表一个Activity实例。
        /*
        	这里我们先归纳下后面的流程，
            AMS通知ActivityStackSupervisor调用startActivityLocked()，
            startActivityLocked这个方法中创建了ActivityRecord，
            ActivityRecord将它的token(即Binder对象)传递给ActivityThread的内部类ApplicationThread,
            ActivityThread在创建ActivityClientRecord对象的时候,将本身的parent传递给了ActivityClientRecord,
            这里我们可以得出,r.parent==null 这个判断语句即是在判断当前App进程是否已经创建.
 				
        */
        

        ...
    }

/*
	返回源码,分析二
    mInstrumentation.execStartActivity
*/

 public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options){
 	...
        //ams.startActivity();
                  int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getOpPackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
     //检查Activity的启动结果
            checkStartActivityResult(result, intent);
 }
	



```

## 2. AMS通知Zygote开辟进程
上一步,Launcher将信息通过Binder传递到了AMS(ATMS),我们接着跟进

```java
/*
	(ATMS)/AMS 
*/
  @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }

// startActivityAsUser

private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
        assertPackageMatchesCallingUid(callingPackage);
       ...

        // 分析
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setCallingFeatureId(callingFeatureId)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setUserId(userId)
                .execute();

    }
/*
	getActivityStartController().obtainStarter返回的是一个ActivityStarter,
    也就是说AMS通知ActivityStarter继续执行逻辑
   ActivityStarter中进行一系列配置,最终交由ActivityStackSupervisor
   	
*/

int executeRequest() {
	...
    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
                restrictedBgActivity, intentGrants);
}

////ActivityStartSupervisor 
 void startSpecificActivityLocked(ActivityRecord r, boolean andResume, boolean checkConfig) {
 	...
        //分析一
     if (wpc != null && wpc.hasThread()) {
            try {
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }
            knownToBeDead = true;
        }
     ...
         //上面的wpc为空,意味着我们需要先创建一个进程
          final Message msg = PooledLambda.obtainMessage(
                    ActivityManagerInternal::startProcess, mService.mAmInternal, r.processName,
                    r.info.applicationInfo, knownToBeDead, "activity", r.intent.getComponent());
            mService.mH.sendMessage(msg);
     

 }

/*
	wpc = WindowProcessController,
    如果app进程启动,那么atms会保存一个wpc信息,用这个信息可以判断是否存在应用进程.
    那么如果wpc是空,也就意味着我们需要先创建一个app进程
*/

//进程没有创建,消息发给AMS.startProcessLocked->Process.start->ZygoteProcess.start

private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
              if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                primaryZygoteState = ZygoteState.connect(mSocket);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
            }
            maybeSetApiBlacklistExemptions(primaryZygoteState, false);
            maybeSetHiddenApiAccessLogSampleRate(primaryZygoteState);
        }
    

        // The primary zygote didn't match. Try the secondary.
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            try {
                secondaryZygoteState = ZygoteState.connect(mSecondarySocket);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
            }
            maybeSetApiBlacklistExemptions(secondaryZygoteState, false);
            maybeSetHiddenApiAccessLogSampleRate(secondaryZygoteState);
        }
...
    }

/**
	创建一个socket并通知Zygote去fork一个进程.
*/

```


## 3. 创建ActivityThread

```java
/*
	zygote创建进程的过程中,同时也创建了ActivityThread
*/
// frameworks/base/core/java/com/android/internal/os/RuntimeInit.java

protected static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        Class<?> cl;

        try {
            cl = Class.forName(className, true, classLoader);
        } catch (ClassNotFoundException ex) {
            throw new RuntimeException(
                    "Missing class when invoking static main " + className,
                    ex);
        }

        Method m;
        try {
            m = cl.getMethod("main", new Class[] { String[].class });
        } catch (NoSuchMethodException ex) {
            throw new RuntimeException(
                    "Missing static main on " + className, ex);
        } catch (SecurityException ex) {
            throw new RuntimeException(
                    "Problem getting static main on " + className, ex);
        }

        int modifiers = m.getModifiers();
        if (! (Modifier.isStatic(modifiers) && Modifier.isPublic(modifiers))) {
            throw new RuntimeException(
                    "Main method is not public and static on " + className);
        }

        /*
         * This throw gets caught in ZygoteInit.main(), which responds
         * by invoking the exception's run() method. This arrangement
         * clears up all the stack frames that were required in setting
         * up the process.
         */
        return new MethodAndArgsCaller(m, argv);
    }

  /*
  	 通过反射实例化了ActivityThread,ActivityThread的main方法启动了looper方法,同时反馈给ams创建完毕
  */
//frameworks/base/core/java/android/app/ActivityThread.java

public static void main(String[] args) {
	...
        //ActivityThread 是主线程, attach方法用于反馈AMS创建结果
     ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);
}


```

## 4.创建Application
以上,AMS通过通知Zygote创建app进程,从而同时ZygoteProcess反射创建了App进程的ActivityThread.
```java
/*
	ActivityThread.attach方法最终转回到AMS的attachApplicationLocked
    
*/
private boolean attachApplicationLocked(@NonNull IApplicationThread thread,
            int pid, int callingUid, long startSeq) {
	...
        //创建Application
        
    thread.bindApplication(processName, appInfo, providerList, null, profilerInfo,
                        null, null, null, testMode,
                        mBinderTransactionTrackingEnabled, enableTrackAllocation,
                        isRestrictedBackupMode || !normalMode, app.isPersistent(),
                        new Configuration(app.getWindowProcessController().getConfiguration()),
                        app.getCompat(), getCommonServicesLocked(app.isolated),
                        mCoreSettingsObserver.getCoreSettingsLocked(),
                        buildSerial, autofillOptions, contentCaptureOptions,
                        app.getDisabledCompatChanges(), serializedSystemFontMap);
    
    ...
    synchronized (mProcLock) {
        //设置WindowProcessController, 后面也就可以用wpc判断进程是否存在.
        
                app.makeActive(thread, mProcessStats);
                
    }
    ...
        // 启动入口Activity
    didSomething = mAtmInternal.attachApplication(app.getWindowProcessController());
           
}

/*
	thread.bindApplication , 分发到ActivityThread,并通过H,也就是Handler创建Application.
*/


```

## 5. 启动Activity

```java
/*
	frameworks/base/core/java/android/app/ActivityThread.java
*/

private void handleBindApplication(AppBindData data) {
	...
        //设置时间格式
    DateFormat.set24HourTimePref(is24Hr);
    //分析一
     if (ii != null) {
            initInstrumentation(ii, data, appContext);
        } else {
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
        }
    Application app;
     app = data.info.makeApplication(data.restrictedBackupMode, null);
   ...
     mInitialApplication = app;
    
    //初始化ContentProvider
     if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    
                    installContentProviders(app, data.providers);
                }
            }
    //完成ContentProvider初始化 后调用onCreate
     mInstrumentation.onCreate(data.instrumentationArgs);
    mInstrumentation.callApplicationOnCreate(app);
    
    /**
    	Instrumentation 每个App经常都有一个(ActivityThread创建),
        makeApplication()方法最终会调用Application的attach()方法.
   		接着初始化ContentProvider,
        然后再调用onCreate方法 callApplicationOnCreate
        到这里Application的的创建过程基本走完了,我们Activity呢?
    */
    
 	
    /**
    	在AMS中,attachApplicationLocked()调用了主要这几步,
        thread.bindApplication(...),
       
        app.makeActive(thread, mProcessStats);
        
        mAtmInternal.attachApplication(...),
        
        attachApplication会执行到ActivityThread的handleLaunchActivity()
        
    */
    
     public Activity handleLaunchActivity(ActivityClientRecord r,
            PendingTransactionActions pendingActions, Intent customIntent) {
     	
         //初始化WindowManagerService
         WindowManagerGlobal.initialize();
          final Activity a = performLaunchActivity(r, customIntent);
         
     }
    
    // performLaunchActivity
    
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    	
        //创建ContextImpl对象,即Context上下文
        ContextImpl appContext = createBaseContextForActivity(r);
        
        
        //创建Activity
        Activity activity = null;
        activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess(isProtectedComponent(r.activityInfo),
                    appContext.getAttributionSource());
        
        //执行attach方法,创建了PhoneWindow
        activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback,
                        r.assistToken, r.shareableActivityToken);
        
        //设置Activity主题
        int theme = r.activityInfo.getThemeResource();
        
        //调用Activity的onCreate方法
        if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
        
        
        
        
    }
}

```
