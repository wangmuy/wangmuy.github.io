---
layout: post
title: "uiautomator 启动流程"
date: 2017-06-23 11:59:57 +0800
categories: [android]
keywords: 
description: 
---

版本: android-6.0.0_r1
===============================================================

运行示例
===============================================================

```
uiautomator runtest AppiumBootstrap.jar -c io.appium.android.bootstrap.Bootstrap -e disableAndroidWatchers false -e acceptSslCerts false
```

shell 脚本
===============================================================

* 位置 frameworks/base/cmds/uiautomator/cmds/uiautomator/uiautomator

```
基础 CLASSPATH=/system/framework/android.test.runner.jar:${base}/framework/uiautomator.jar
收集 runtest 后跟的所有 jar 添加到 CLASSPATH=${CLASSPATH}:${jars}, 设置 args="${cmd} 其余参数${args} -e jars ${jars}"

exec app_process /system/bin com.android.commands.uiautomator.Launcher ${args}
```

com.android.commands.uiautomator.Launcher#main()
===============================================================

* 位置 frameworks/base/cmds/uiautomator/cmds/uiautomator/src/com/android/commands/uiautomator/Launcher.java

```
{ Command command = findCommand(args[0]); command.run(args[1:]); }
-> command.name()=="runtest": RunTestCommand.run(args[1:])
-> RunTestCommand.run()
  -> parseArgs() 解析命令行参数
    -c com.myapp.myclass 和 -e class com.myapp.myclass 通过 addTestClasses() 加到 mTestClasses
    -e runner com.myapp.myrunner 设置到 mRunnerClassName
    -e debug true 设置到 mDebug
    --monkey 设置到 mMonkey=true
    -s 设置到 mParams["outputFormat"]="simple"
    其余放到 mParams map
  -> if(mTestClasses.isEmpty()) addTestCalssesFromjars() 若命令行没指定test class, 查找 jars 指定的所有dex文件中 继承自 UiAutomatorTestCase 且是 top-level 的类, 加到 mTestClasses
  -> getRunner().run(mTestClasses, mParams, mDebug, mMonkey)
      getRunner() 返回 UiAutomatorTestRunner() 或指定的 mRunnerClassName 类实例

    -> UiAutomatorTestRunner.run()
      设置线程的 defaultUncaughtExceptionHandler 报告给 watcher 并退出
      start()
      退出 返回0
        -> start()
          通过 (TestCaseCollector getTestCaseCollector()).addTestClasses(mTestClasses) 添加到 TestCaseCollector.mTestCases 每个 <className, methodName> 一条
          启动 UiAutomatorTestRunner.mHandlerThread 线程 daemon 化(防止进程提前退出？)
          (automationWrapper=new UiAutomationShellWrapper()).connect() 创建 UiAutomationShellWrapper.mHandlerThread 线程
          UiAutomationShellWrapper.mUiAutomation = new UiAutomation(ShellWrapper的 looper, new UiAutomationConnection())
            创建 mClient=IAccessibilityServiceClientImpl(ShellWrapper的 looper), UiAutomationConnection 是连接accessibilityService的binder实现?, 提供instrumentation和跨app测试能力
          mUiAutomation.connect() 注册到accessibilityManagerService 并sleep循环等待binder回调设置 mConnectionId
            -> mUiAutomationConnection.connect(mClient)
              { AccessibilityManager.registerUiTestAutomationService(mToken, client, info) }

          以上是与 accessibilityManagerService 连接上的所有步骤

          mUiDevice = UiDevice.getInstance(); mUiDevice.initialize(new ShellUiAutomatorBridge(automationWrapper.getUiAutomation()))
            UiDevice 的初始化

          testRunResult.addListener(resultPrinter); foreach mTestListeners: testRunResult.addListener() 添加listener
          foreach testCases: preapreTestCase(testCase); testCase.run(testRunResult);
            -> prepareTestCase()
              testCase.setAutomationSupport((mAutomationSupport=new IAutomationSupport() 负责 sendStatus 给 mWatcher ));
              testCase.setUiDevice(mUiDevice);
              testCase.setParams(mParams);

          finally
            resultPrinter.print(testRunResult, runTime, testRunOutput); 输出运行结果
            automationWrapper.disconnect() 从 accessibilityManagerService 注销
            automationWrapper.setRunAsMonkey(false)
            mHandlerThread.quit() 退出线程, 进程结束
```

listener/watcher/结果输出 相关
===============================================================

```
* UiAutomatorTestRunner.java
  * run 线程设置 UncaughtExceptionHandler(): results=new Bundle(); results.putString(); mwatcher.instrumentationFinished(null,0,results);
  * start()
    resultPrinter = new WatcherResultPrinter(testCases.size())
      或 mParams 里包含 "outputFormat":"simple" 时 new SimpleResultPrinter(System.out, true)
    testRunResult = new TestResult();
    testRunResult.addListener(resultPrinter);
    foreach mTestListeners: testRunResult.addListener(l)

    运行测试 foreach testCase: testCase.run(testRunResult)
    异常出错时 resultPrinter.printUnexpectedError(Throwable t); testRunOutput.putStirng("shortMsg", t.getMessage());
    finally resultPrinter.print(testRunResult, runTime, testRunOutput)

* UiAutomatorTestRunner.WatcherResultPrinter.print(result, runTime, Bundle testOutput)
  mPrinter.print(result, runTime, testOutput)
    WatcherResultPrinter 构造函数中
      mStream=new ByteArrayOutputStream();
      mWriter=new PrintStream(mStream);
      mPrinter=new SimpleResultPrinter(mWriter,fullOutput=false) 输出到 ByteArray
  testOutput.putString(..., mStream.toString()) 输出到bundle中
  mWriter.close()
  mAutomationSupport.sendStatus(RESULT_OK, testOutput) 发送给 watcher

  * UiAutomatorTestRunner.SimpleResultPrinter.print(result, runTime, testOutput)
    printHeader(runTime);
    if(fullOutput): printErrors(result); printFailures(result);
    printFooter(result);

* UiAutomatorTestRunner.WatcherResultPrinter.printUnexpectedError(Throwable t)
  (getWriter()==mWriter).printf("...aborted...");
  t.printStackTrace(getWriter())

* 发送给 mWatcher = new FakeInstrumentationWatcher()
FakeInstrumentationWatcher.instrumentationStatus(ComponentName null, int resultCode, Bundle status)
  mRawMode==false: foreach status.keyValue: System.out.println(...)
  notifyAll()
```
