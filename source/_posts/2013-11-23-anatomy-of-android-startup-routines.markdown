---
layout: post
title: "anatomy-of-android-startup-routines"
date: 2013-11-23 16:10
comments: true
categories: [android]
keywords: 
description: 
#published: false
---

<pre>

版本 android-4.3_r2.2
========================


涉及到的文件
========================
* /system/core/init/init.c
* /system/core/init/init_parser.c
** init_parse_config_file()
* /system/core/rootdir/init.rc
* /frameworks/base/cmds/app_process/app_main.cpp
** LOCAL_MODULE:= app_process
* /dalvik/vm/Jni.cpp
** JNI_CreateJavaVM()
* /dalvik/vm/Init.cpp
** dvmStartup()

init
========================
main() { init_parse_config_file("/init.rc"); execute_one_command(); }
-> init_parse_config_file() 读init.rc, 按顺序读入:

* ueventd /sbin/ueventd
** class core
* [optional]console /system/bin/sh
** class core
* [optional]adbd /sbin/adbd
** class core
* servicemanager /system/bin/servicemanager
** class core
* vold /system/bin/vold
** class core
* netd /system/bin/netd
** class main
* debuggerd /system/bin/debuggerd
*class main
* ril-daemon /system/bin/rild
** class main
* surfaceflinger /system/bin/surfaceflinger
** class main
* zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
** class main
** socket zygote stream 660 root system
** 本篇主要分析这个
* drm /system/bin/drmserver
** class main
* media /system/bin/mediaserver
** class main
* [optional]bootanim /system/bin/bootanimation
** class main
等内容.

-> init.rc 读到
on boot
    ...
    class start core
    class_start main
on 是一个 SECTION, parse_new_section(), 此SECTION下的每行 state.parse_line() 就是调的 parse_line_action().
class_start 是一个 COMMAND, symbol是 K_class_start(keywords.h用宏拼接出来)
zygote服务有 class main. 在 init.c::main() 中通过
action_for_each_trigger("boot", action_add_queue_tail) 加入到 init_parser.c::action_queue 队列.
最后通过 init.c::main() 中 execute_one_command() 从队列中取出执行
-> builtins.c::do_class_start()
-> service_for_each_class(args[1], service_start_if_not_disabled)
-> service_start_if_not_disabled()
-> service_start() {
  fork();
  if(pid==0)
    create_socket("zygote", stream, 660, root, system);
    publish_socket(); app_process的环境变量增加 ANDROID_SOCKET_zygote="/dev/socket/zygote"
  }

Zygote(app_process进程)
========================
main() {
  parentDir <= "/system/bin"
  if("--zygote") (AppRuntime runtime).start("com.android.internal.os.ZygoteInit", "--startSystemServer"? flag);
  else runtime.start("com.android.internal.os.RuntimkeInit", "--application" or tool? flag); }
-> AppRuntime父类 AndroidRuntime::start() {
  startVm(&mJavaVM, &env); onVmCreated(env); startReg(env); JNI调java的main(), 当前线程作为VM主线程, 直到VM退出. }
  -> AndroidRuntime::startVm()
    -> JNI_CreateJavaVM() { memset((DvmGlobals gDvm), 0); calloc (JavaVMExt *pVM); dvmCreateJNIEnv(); dvmStartup(); }
      -> dvmCreateJNIEnv() { calloc (JNIEnvExt *newEnv); dvmSetJniEnvThreadId(newEnv, self); }
      -> dvmStartup() {
        processOptions() "-Xzygote"设置 gDvm.zygote;
        if(gDvm.zygote) initZygote() else dvmInitAfterZygote(); }
--JAVA--> ZygoteInit#main() {
  registerZygoteSocket(); preload(); gc();
  if("start-system-server") startSystemServer();
  无限循环runSelectLoop(); }
  -> registerZygoteSocket() JAVA端用上 create_socket() 出来的 UNIX Domain socket, 作为server端.
  -> preload() {
    preloadClasses(); "preloaded-classes"文件每行 #ClassName 调用 Class.forName()
    preloadResources(); }
-> startSystemServer() {
  Zygote.forkSystemServer(); 最后调的是 dalvik_system_Zygote.cpp::Dalvik_dalvik_system_Zygote_forkSystemServer()
  if(pid==0) handleSystemServerProcess() }


System Server(app_process子进程)
========================
--SystemServer进程--> handleSystemServerProcess()
-> RuntimeInit#zygoteInit() {
  redirectLogStreams(); 重定向System.out和System.err到logcat系统.
  commonInit(); nativeZygoteInit(); applicationInit(); }
  --JNI--> nativeZygoteInit()
    -> AndroidRuntime.cpp::com_android_internal_os_RuntimeInit_nativeZygoteInit()
    -> app_main.cpp AppRunTime::onZygoteInit() { (ProcessState proc)->startThreadPool() }
    -> ProcessState.cpp::spawnPooledThread(true) {
      t=new PoolThread(isMain==true); // mCanCallJava默认为true
      // android_atomic_add()等类似函数返回操作之前的数
      t->run("Binder_1", 默认priority==PRIORITY_DEFAULT, 默认stack==0) }
      --新线程--> _threadLoop() -> PoolThread::threadLoop()
      -> IPCThreadState::joinThreadPool() // ioctl "/dev/binder" 加入Binder线程池, 无限阻塞获取并执行binder驱动发过来的命令
  -> applicationInit()
    -> invokeStatkcMain("com.android.server.SystemServer", startArgs) // args在 ZygoteInit#startSystemServer()
    -> throw new ZygoteInit.MethodAndArgsCaller(m, argv) 抛异常方式扔掉初始化时多余的栈, 在ZygoteInit#main()捕捉
    -> ZygoteInit#main caller.run()
-> SystemServer#main() { loadLibrary("android_servers"); init1(); }
  -> init1() --JNI--> android_server_SystemServer_init1()
  --C++--> system_init.cpp::system_init() {
    defaultServiceManager(); // handle==0
    JNI调 SystemServer#init2();
    ProcessState::startThreadPool(), joinThreadPool(默认isMain==true); }
    -> SystemServer#init2()
    --新线程--> ServerThread#run() {
      uiHandlerThread = new HandlerThread(); // 起UI线程
      wmHandlerThread = new HandlerThread(); // 起WM线程
      context = ActivityManagerService.main();
      起各个service:
      wm = WindowManagerService.main(context, power, display, inputManager, ...);
      ServiceManager.addService(Context.WINDOW_SERVICE, wm);
      ActivityManagerService.self().setWindowManager(wm);
      wm.displayReady();
      ...
      ActivityManagerService.self().systemReady(runnable调各个service.systemReady); // AMS准备完毕
    }
    -> ActivityManagerService#main() {
      thr = new AThread(); thr.start(); // 新起Looper用线程: 创建AMS, looper.loop()
      (ActivityThread at) = ActivityThread.systemMain(); // 本线程本Looper管理Acitivty生命周期
      (ActivityManagerService m).mMainStack = new ActivityStack(m, context==at.getSystemContext(), true, thr.mLooper);
      m.startRunning();
    }
      -> ActivityThread#systemMain() { thread=new ActivityThread(); thread.attach(system==true); }
        -> ActivityThread#attach() {
          system==true:
            context=new ContextImpl(); context.init();
            app=newApplication(Application.class, context);
            mAllApplications.add(app); mInitialApplication=app; app.onCreate();
          }
    -> WMS#displayReady() {
        displayReady(Display.DEFAULT_DISPLAY);
        mDisplayReady = true;
        mIsTouchDevice = mContext.getPackageManager().hasSystemFeature(PackageManager.FEATURE_TOUCHSCREEN);
        (mPolicy 就是 PhoneWindowManager).setInitialDisplaySize(...);
      }
      -> WMS#displayReady(DEFAULT_DISPLAY) {
        DisplayContent displayContent = getDisplayContentLocked(DEFAULT_DISPLAY);
        mAnimator.addDisplayLocked(displayId);
      }
        -> WMS#getDisplayContentLocked(DEFAULT_DISPLAY) {
          Display display = mDisplayManager.getDisplay(DEFAULT_DISPLAY);
          displayContent = newDisplayContentLocked(display);
        }
        -> DisplayManager#getDisplay(DEFAULT_DISPLAY) { DisplayManager#getOrCreateDisplayLocked(DEFAULT_DISPLAY, false); }
        -> WMS#newDisplayContentLocked(display) {
          displayContent = new DisplayContent(display);
          mDisplaySettings.getOverscanLocked(info.name, rect);
          mDisplayManagerService.setOverscan(display.getDisplayId(), rect...);
          (mPolicy 就是 PhoneWindowManager).setDisplayOverscan(displayContent.getDisplay(), rect...);
        }
    -> AMS#systemReady(goingCallback) {
      goingCallback.run();
      mMainStack.resumeTopActivityLocked(null);
    }
      -> ActivityStack#resumeTopActivityLocked(null)
      -> resumeTopActivityLocked(prev==null, options==null) { next==null: mService.startHomeActivityLocked() }
      -> ActivityServiceManager#startHomeActivityLocked()
      -> ActivityStack#startActivityLocked(caller==null, intent, resolvedType==null, aInfo==Launcher信息,
        resultTo==null, resultWho==null, requestCode==0, callingPid==0, callingUid==0, callingPackage==null,
        startFlags==0, options==null, componentSpecified==false, outActivity==null)
      -> startActivityUncheckedLocked(r==new ActivityRecord(), sourceRecord==null,
        startFlags==0, doResume==true, options==null)
      -> startActivityLocked(r, newTask==true, doResume==true, keepCurTransition==false, options==null) {
        mHistory.size() == 0: mService.mWindowManager.addAppToken(addPos==0 ...);
        doResume==true: resumeTopActivityLocked(null);
      }
        -> resumeTopActivityLocked(null)
        -> resumeTopActivityLocked(prev==null, options==null) {
          (ActivityRecord next)==Launcher
          (ProcessRecord next.app)==null: startSpecificActivityLocked(next, andResume==true, checkConfig==true)
        }
        -> startSpecificActivityLocked() { getProcessRecordLocked()==null: mService.startProcessLocked(); }
        -> AMS#startProcessLocked(r.processName, r.info.applicationInfo, knownToBeDead==true, intentFlags==0,
          hostingType=="activity", r.intent.getComponent(), allowWhileBooting==false, isolated==false) {
            getProcessRecordLocked()==null:
              app=newProcessRecordLocked(null, info, processName, isolated==false)
              startProcessLocked(app, hostingType=="activity", hostingNameStr)
          }
          -> startProcessLocked() { startResult=Process.start("android.app.ActivityThread", ...) }
          -> Process#startViaZygote(processClass=="android.app.ActivityThread", ...)
          -> Process#zygoteSendArgsAndGetResult() { Binder命令zygote(app_process) fork新进程 }

Launcher应用
========================
--新进程Launcher--> ZygoteConnection#handleChildProc()
-> ZygoteInit#invokeStaticMain()
--抛异常方式回调--> ActivityThread#main() {
  thread=new ActivityThread(); thread.attach(false);
  sMainHandler=thread.getHandler(); Looper.loop(); }
-> ActivityThread#attach(system==false) {
  mgr=ActivityManagerNative.getDefault(); mgr.attachApplication(ApplicationThread mAppThread); }
--Binder--> ActivityManagerNative#attachApplication(app)
-> 子类AMS#attachApplication(app)
-> AMS#attachApplicationLocked() {
     ActivityRecord hr=mMainStack.topRunningActivityLocked();
     hr!=null && normalMode, hr.app==null:
       mMainStack.realStartActivityLocked(hr, app, true, true);
   }
  -> ActivityStack#realStartActivityLocked() { app.thread.scheduleLaunchActivity(intent, appToken, ...); }
--Binder--> Launcher进程 ApplicationThread#scheduleLaunchActivity()
-> queueOrSendMessage(H.LAUNCH_ACTIVITY, r)
--Handler H--> ActivityThread#handleLaunchActivity(r, null) { performLaunchActivity(); handleResumeActivity(); }


</pre>