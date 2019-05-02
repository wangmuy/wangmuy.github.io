---
layout: post
title: "Android 输入系统 源码分析"
date: 2013-11-04 16:45
categories: [android]
description: 
---

版本 android-4.3_r2.2
========================


涉及到的文件
========================
* frameworks/base/services/input/InputDispatcher.cpp
  * InputDispatcher
  * InputDispatcherThread
* frameworks/base/services/input/InputReader.cpp
  * InputReader
  * InputReaderThread
* frameworks/base/services/input/EventHub.cpp
* frameworks/base/services/java/com/android/server/input/InputManagerService.java
* frameworks/base/services/jni/com_android_server_input_InputManagerService.cpp
* frameworks/base/core/jni/android_view_InputChannel.cpp
  * NativeInputChannel
* frameworks/base/services/input/InputManager.cpp
* frameworks/native/libs/utils/Threads.cpp
* /frameworks/base/include/androidfw/InputTransport.h .cpp
  * InputPublisher
  * InputChannel


初始化( SystemServer.java: ServerThread#run() )
========================

```
NIM: NativeInputManager
InputManagerService#ctor()
--JNI--> nativeInit()
--C++--> NativeInputManager::ctor()
-> InputManager::ctor(new eventHub(), NIM, NIM)
    {mDispatcher=InputDispatcher::ctor(NIM),
     mReader=InputReader::ctor(eventHub==new eventHub in NIM, NIM, mDispatcher), InputReader::mQueuedListener=QueuedInputListener::ctor(mDispatcher)) }
-> InputManager::initialize() { InputReaderThread::ctor(mReader), InputDispatcherThread::ctor() }

InputManagerService#setWindowManagerCallbacks(wm.getInputMonitor())
    { mWindowManagerCallbacks == InputMonitor#ctor() }
```

启动( SystemServer.java: ServerThread#run() )
========================

```
InputManagerService#start()
--JNI--> nativeStart()
--C++--> InputManager::start() { mDispatcherThread->run(), mReaderThread->run() }

InputDispatcherThread::run() [重载 Thread::run]
    { androidCreateRawThreadEtc() -> pthread_create()
     --新线程--> Thread::_threadLoop() --无限循环--> InputDispatcherThread::threadLoop() -> Dispatcher::dispatchOnce() }

InputReaderThread::run() [重载 Thread::run]
    --新线程--> ... --无限循环--> InputReaderThread::threadLoop()
    -> mReader::loopOnce()
      { mEventHub::getEvents(timeout, eventBuffer), processEventsLocked(eventBuffer), 更新inputDevices列表, mQueuedListener::flush() }
    -> EventHub::getEvents()
        { (mNeedToScanDevices 初始化为 true) scanDevicesLocked(),
          无限循环: 处理 ReopenDevice/ScanDevice/pendingEvent, epoll_wait(device_fds) 获取 pendingEvent }

EventHub::scanDevicesLocked()
-> EventHub::scanDirLocked("/dev/input") { foreach devFile: openDeviceLocked(devname) }
-> EventHub::openDeviceLocked()
    { LoadConfigurationLocked(), epoll_ctl(CTL_ADD), addDeviceLocked() }
-> EventHub::addDeviceLocked() { 加到 mDevices }
```

建立连接
========================

```
如果没有任何连接, InputDispatcher会扔掉接收到的按键.
ViewRootImpl#setView()
    { mInputChannel=InputChannel#ctor,
      mWindowSession.addToDisplay(..., mInputChannel),
      mInputEventReceiver = WindowInputEventReceiver#ctor((mInputChannel==transfer过来的客户端channel), Looper.myLooper()) }

mWindowSession.addToDisplay()
--Binder--> Session#addToDisplay(..., outInputChannel==mInputChannel)
-> WindowManagerService#addWindow(..., outInputChannel)
    { inputChannels=InputChannel#openInputChannelPair(name),
      (WindowState win).setInputChannel(inputChannels[0]),
      inputChannels[1].transferTo(outInputChannel), 结果是 ViewRootImpl#mInputChannel 赋值为 socketpair() 客户端
      (InputManagerService mInputManager).registerInputChannel(win.mInputChannel==inputChannels[0], ...), 结果是 InputDispatcher 中一个 Connection::inputChannel 赋值为 socketpair() 服务器端 }

WindowInputEventReceiver#ctor()
-> InputEventReceiver#ctor()
--JNI--> InputEventReceiver::nativeInit() { receiver=NativeInputEventReceiver::ctor(), receiver->initialize() }
-> receiver->initialize()
-> setFdEvents() { mMessageQueue->getLooper()->addFd(mInputConsumer.getChannel()->getFd()) 客户端, ViewRootImpl Looper }

InputChannel#openInputChannelPair()
--JNI--> nativeOpenInputChannelPair()
--C++--> InputChannel::openInputChannelPair([0]==serverChannel, [1]==clientChannel)
    { fd[2]=socketpair(), serverChannel.mFd=fd[0], clientChannel.mFd=fd[1] }

InputChannel#transferTo()
--JNI--> nativeTransferTo() { outInputChannel=inputChannels[1], inputChannels[1]==null }

InputManagerService#registerInputChannel(inputChannel)
--JNI--> nativeRegisterInputChannel(inputChannel)
--C++--> (NativeInputManager im)->registerInputChannel(env, inputChannel, ...)
-> InputManager::->getDispatcher()->registerInputChannel(inputChannel, ...)
-> InputDispatcher::registerInputChannel()
    { Connection::ctor(inputChannel, ...),
      mLooper->addFd(inputChannel->getFd(), ...) }

Looper::addFd()
    { epoll_ctl(ADD/MOD), 加入到 mRequests 队列 }

接收event
------------------------
ViewRootImpl#requestLayout() / invalidate() / setLayoutParams() 等
-> ViewRootImpl#scheduleTraversals()
-> ViewRootImpl#scheduleConsumeBatchedInput()
--Looper机制--> (ConsumeBatchedinputRunnable mConsumedBatchedInputRunnable)#run()
-> ViewRootImpl#doConsumeBatchedInput()
-> (WindowInputEventReceiver mInputEventReceiver).consumeBatchedInputEvents()
-> InputEventReceiver#consumeBatchedInputEvents()
--JNI--> nativeConsumeBatchedInputEvents(receiver)
--C++--> (NativeInputEventReceiver receiver)::consumeEvents()
    { 无限循环: [
        (InputConsumer mInputConsumer==WRAPS(inputChannel == InputEventReceiver 构造时传入)).consume(),
        (JNI调java) InputEventReceiver#dispatchInputEvent()] }

InputEventReceiver#dispatchInputEvent()
 --重载--> WindowInputEventReceiver::onInputEvent(event)
 -> ViewRootImpl#enqueueInputEvent()

InputConsumer::consume()
-> mChannel->receiveMessage(), 是 socketpair() 客户端
    { UNIX Socket ::recv(mFd) }
```

InputReader处理
========================

```
InputReaderThread::processEventsLocked(deviceId, rawEvents, count)
-> --::processEventsForDeviceLocked(deviceId, rawEvent, batchSize) [或 add/remove)DeviceLocked / handleConfigurationChangedLocked]
-> mDevices中取 (InputDevice device)::process(rawEvents, count)
    { for rawEvents: 命令扔掉 or { for mMappers: InputMapper::process(rawEvent) } }
    mMappers 在 InputReader::createDeviceLocked() 中添加, 包括 (Switch/Vibrator/Keyboard/Cursor/MultiTouch/SingleTouch/Joystick)InputMapper
-> InputMapper子类::process(rawEvent), 以Keyboard为例 { mReader->mQueuedListener::notifyKey(args) }
-> QueuedInputListener::notifyKey(args) { (Vector<NotifyArgs*> mArgsQueue).push(new NotifyKeyArgs(*args)) }

(QueuedInputListener mQueuedListener)::flush()
-> NotifyArgs::notify(mInnerListener==InputDispatcher)
-> InputDispatcher::notifyKey(NotifyKeyArgs) { mPolicy->interceptKeyBeforeQueueing(), enqueueInboundEventLocked(newEntry) }

InputDispatcher::enqueueInboundEventLocked(entry) { mInboundQueue.enqueueAtTail(entry) }


InputReaderThread和InputDispatcherThread两线程对 inBoundQueue 的访问是通过 InputDispatcher::mLock 锁来保护的.
```

InputDispatcher处理
========================

```
InputDispatcher::dispatchOnce()
    { dispatchOnceInnerLocked() 最终目的 UNIX Socket send 给注册的 inputChannel,
      mLooper->pollOnce() 最终目的 epoll_wait(inputChannelFd) 确认应用返回的已处理消息 }

dispatchOnceInnerLocked() [或 runCommandsLockedInterruptible()]
------------------------
    { mInboundQueue.dequeueAtHead() , 以键盘 TYPE_KEY 为例 dispatchKeyLocked() }

InputDispatcher::dispatchKeyLocked(currentTime, entry, dopReason, nextWakeupTime)
    { (policy==NIM) intercept key 转为 CommandEntry, 或 扔掉, 或
      findFocusedWindowTargetsLocked(), dispatchEventLocked(currentTime, entry, inputTargets) }
-> dispatchEventLocked() { foreach (InputDispatcher::Connection connection): prepareDispatchCycleLocked() }
-> prepareDispatchCycleLocked()
-> enqueueDispatchEntriesLocked() { enqueueDispatchEntryLocked(), 若之前为空则 startDispatchCycleLocked() }
-> enqueueDispatchEntryLocked() { connection->outboundQueue.enqueueAtTail(dispatchEntry) }

(policy==NIM) intercept key
-> InputDispatcher::postCommandLocked()
--重新以CommandEntry形式dispatch--> InputDispatcher::runCommandsLockedInterruptible()
    { (commandEntry->command函数指针==doInterceptKeyBeforeDispatchingLockedInterruptible)(commandEntry) }
-> (mPolicy==NIM)->interceptKeyBeforeDispatching()
--JNI回调java--> InputManagerService#interceptKeyBeforeDispatching()
--JAVA--> (mWindowManagerCallbacks==InputMonitor).interceptKeyBeforeDispatching()
-> (WindowManagerService mService).(WindowManagerPolicy mPolicy==Binder通信的PhoneWindowManager).interceptKeyBeforeDispatching()
--Binder通信--> PhoneWindowManager#interceptKeyBeforeDispatching()
    { HOME/MENU/SEARCH 等全局键处理 }

InputDispatcher::startDispatchCycleLocked(currentTime, connection)
    { 以键盘 TYPE_KEY 为例 connection->( InputPublisher inputPublisher ==WRAPS(mChannel)== connection 建立时的 inputChannel == NativeInputChannel 建立时的 inputChannel )->publishKeyEvent(),
      如果成功则 connection->outboundQueue.dequeue(dispatchEntry), connection->waitQueue.enqueueAtTail(dispatchEntry) }
-> InputPublisher::publishKeyEvent()
-> (InputChannel mChannel)::sendMessage(InputMessage::ctor()) { UNIX Socket ::send(mFd, msg ...) }

InputDispatcher::Connection 类在 InputDispatcher.h 中. 所有的connection在 InputDispatcher::registerInputChannel() 中注册.

mLooper->pollOnce()
------------------------
-> Looper::pollInner()
    { epoll_wait(inputChannelFd), foreach event: [pushResponse(events, mRequests[idx])],
      foreach in mMessageEnvelopes: messageEnvelope.handler->handleMessage(messageEnvelope.messge),
      foreach in mResponses: response.request.callback->handleEvent() }
-> Looper::pushResponse() { 添加到 mResponses 队列 }
-> response.request.callback->handleEvent()
-> InputDispatcher::handleReceiveCallback()
    { connection->inputPublisher.receiveFinishedSignal(), (InputDispatcher d)->finishDispatchCycleLocked() }

InputPublisher::receiveFinishedSignal()
-> mChannel->receiveMessage(), 是 socketpair() 服务器端

InputDispatcher::finishDispatchCycleLocked()
-> InputDispatcher::onDispatchCycleFinishedLocked() { 通知并准备下个 dispatch cycle }
--mCommandQueue队列--> InputDispatcher::doDispatchCycleFinishedLockedInterruptible()
    { connection->waitQueue.dequeue(dispatchEntry), startDispatchCycleLocked() }


Looper机制
----------------------------------
ActivityThread#main()
-> Looper#loop() { 无限循环: 处理 (MessageQueue queue).next() }
-> MessageQueue#next()
--JNI--> nativePollOnce()
--C++--> NativeMessageQueue::pollOnce()
-> Looper::pollOnce()
```
