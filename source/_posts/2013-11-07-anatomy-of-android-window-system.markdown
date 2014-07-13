---
layout: post
title: "anatomy-of-android-window-system"
date: 2013-11-07 15:20
comments: true
categories: 
keywords: 
description: 
published: false
---

版本 android-4.3_r2.2
========================


涉及到的文件
========================

SystemServer进程
========================

应用进程
========================
承接 anatomy-of-android-startup-routines:
ActivityThread#handleLaunchActivity(r, null) { performLaunchActivity(); handleResumeActivity(); }

-> ActivityThread#performLaunchActivity() {
   activity = newActivity();
   Application app=makeApplication();
   appContext = createBaseContextForActivity();
   activity.attach(appContext, this, ..., app, ...);
   mInstrumentation.callActivityOnCreate(activity, r.state); // activity.performCreate() -> activity.onCreate();
   if(!r.activity.mFinished):
     activity.performStart();
   if(!r.activity.mFinished && r.state!=null):
     mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
   if(!r.activity.mFinished):
     mInstrumentation.callActivityOnPostCreate(activity, r.state); }

  -> Instrumentation#newActivity() { ClassLoader.loadClass(className).newInstance(); }
  -> LoadedApk#makeApplication() { // 创建Application类的Context
       ContextImpl appContext=new ContextImpl(); appContext.init(this, null, mActivityThread);
       (Application app)=mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
       appContext.setOuterContext(app);
       instrumentation.callApplicationOnCreate(app);
     }
    -> ContextImpl#类装载时 static { registerService(WINDOW_SERVICE 等各种service); }
  -> ActivityThread#createBaseContextForActivity(r, activity) {
       ContextImpl appContext = new ContextImpl();
       appContext.init(r.packageInfo, r.token, this);
       appContext.setOuterContext(activity;);
       baseContext=appContext;
       if(没有"debug.second-display.pkg"属性) return baseContext;
     }
  -> Activity#attach() -> Activity#attach(context==ContextImpl appContext, ...) {
       attachBaseContext(context); // mBase==appContext
       mFragments.attachActivity(this, mContainer, null);
       (Window mWindow) = PolicyManager.makeNewWindow(this); // 得到 Window 子类 PhoneWindow
       mWindow.setCallback(this);
       mUiThread = Thread.currentThread();
       mMainThread = 参数 ActivityThread aThread;
       mWindow.setWindowManager(getSystemService(WINDOW_SERVICE));
       mWindowManager = mWindow.getWindowManager(); // 取到WindowManagerImpl
     }
    -> PolicyManager#makeNewWindow() { (com.android.internal.policy.impl.Policy sPolicy).makeNewWindow(); }
      -> Policy#makeNewWindow() {
           return new PhoneWindow(context); } // extends Window. mWindowAttributes = new WindowManager.LayoutParams();
    -> (Window PhoneWindow父类)#setWindowManager(wm, appToken, appName, hardwareAccelerated) {
    	wm!=null:
    	  mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
    	  // PhoneWindow 作为本进程所有Window的parent }
      -> WindowManagerImpl#createLocalWindowManager(parentWindow) { return new WindowManagerImpl(mDisplay, parentWindow); }


ActivityThread#handleResumeActivity(r.token, clearHide==false, r.isForward==false,
  reallyResume = (!r.activity.mFinished && !r.startsNotResumed) == true) {
  ActivityClientRecord r = performResumeActivity(token, clearHide==false);
  Activity a = r.activity;
  r.window==null && !a.mFinished && willBeVisible:
    r.window = r.activity.getWindow(); // 就是 PhoneWindow
    decor = r.window.getDecorView();
    decor.setVisibility(INVISIBLE);
    a.mDecor = decor;
    if(a.mVisibleFromClient==初始化为true):
      a.mWindowAdded=true;
      WindowManager.LayoutParams l = r.window.getAttributes();
      l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
      l.softInputMode |= (forwardBit==0);
      (WindowManagerImpl wm).addView(decor, l);
    if(!r.activity.mFinished && willBeVisible && r.activity.mDecor!=null && !r.hideForNow):
      r.activity.mVisibleFromServer = true;
      if(r.activity.mVisibleFromClient) r.activity.makeVisible();
    if(!r.onlyLocalRequest) Looper.myQueue().addIdleHandler(new Idler());
    if(reallyResume) ActivityManagerNative.getDefault().activityResumed(token);
}

  -> performResumeActivity() { if!(r.activity.mFinished) r.activity.performResume(); }
    -> Activity#performResume() {
    	performRestart(); // mStopped==false: pass
    	mInstrumentation.callActivityOnResume(this); // activity.mResumed=true; activity.onResume()
    	mFragments.dispatchResume();
    	onPostResume(); }

  -> PhoneWindow#getDecorView() { if(mDecor==null) installDecor(); return mDecor; }
    -> installDecor() { if(mDecor==null) mDecor=generateDecor(); }
      -> generateDecor() { return new DecorView(getContext(), featureId==-1); }
         // PhoneWindow.DecorView extends FrameLayout implements RootViewSurfaceTaker

  -> WindowManagerImpl#addView()
  -> (单实例WindowManagerGlobal mglobal)#addView(view==DecorView decor, params, display, parentWindow==PhoneWindow) {
       root = new ViewRootImpl(view.getContext(), display);
       view 加到 mViews[], root 加到 mRoots[].
       (ViewRootImpl root).setView(view, wparams==params, panelParentView 非SUB_WINDOW类==NULL);
     }
    -> ViewRootImpl#ctor() {
    	mSurface = new Surface(); // 空surface, 以后通过 readFromParcel() 填充
    	mWindowSession = WindowManagerGlobal.getWindowSession();
    	mThread = Thread.currentThread();
      mWindow = new W(this); // W extends IWindow.Stub
      mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this);
    	mChoreographer = Chogreographer.getInstance();
    	loadSystemProperties(); }
      -> WindowManagerGlobal#getWindowSession() {
      	   if(sWindowSession==null):
      	     windowManager = getWindowManagerService() {
      	     	sWindowManagerService = BinderProxy端 ServiceManager.getService("window"); }
      	     sWindowSession = windowManager.openSession(
      	     	(单实例InuptMethodManager imm).getClient(), imm.getInputContext() );
      	     // BinderProxy端 IWindowSession, systemServer进程的WMS线程增加一个 Session.
      	     return sWindowSession;
         }
         --systemServer进程的WMS所在线程--> WMS#openSession() { new Session(service==WMS, ...); }
           -> Session#ctor() { mService.mInputMethodManager.addClient(
           	InputMethodClient client, inputContext, mUid==应用的Uid, mPid==应用的Pid); }
    -> ViewRootImpl#setView(view==DecorView decor, attrs=wparams, panelParentView==NULL) {
         mView == null:
           mView = decor;
           view instanceof RootViewSurfaceTaker:
             mSurfaceHolderCallback = ((RootViewSurfaceTaker)view).willYouTakeTheSurface(); // == null
           mSurfaceHolder == null: enableHardwareAcceleration(decor.getContext(), attrs);
           mAdded = true;
           requestLayout();
           mWindowAttributes.inputFeatures==0: mInputChannel = new InputChannel();
           (Session mWindowSession).addToDisplay(mWindow, mSeq==0, mWindowAttributes==attrs,
             getHostVisibility(), mDisplay.getDisplayId(), mAttachInfo.mContentInsets, mInputChannel);
       }
      -> (PhoneWindow.DecorView implements RootViewSurfaceTaker)#willYouTakeTheSurface() {
           mFeatureId==-1 < 0: return PhoneWindow.mTakeSurfaceCallback;
        }

      -> ViewRootImpl#enableHardwareAcceleration() {
          // HardwareRenderer.isAvailable() == true
          mAttachInfo.mHardwareRenderer = HardwareRenderer.createGlRenderer(2, translucent);
          mAttachInfo.mHardwareAccelerated = mAttachInfo.mHardwareAccelerationRequested = true;
        }
        -> HardwareRenderer#createGlRenderer()
        -> Gl20Renderer#create(translucent) { if(GLES20Canvas.isAvailable(): new Gl20Renderer(translucent); }

      -> Session#addToDisplay()
        --Binder通信--> WindowManagerService#addWindow(session, client==W, seq==0, attrs,
            viewVisibility, displayId, outContentInsets, outInputChannel) {
            type == TYPE_BASE_APPLICATION;
            (WindowToken token) 在 ActivityStack#startActivityLocked() -> WMS#addAppToken() 中设置, 不为空.
            (AppWindowToken atoken) 同上, 不为空.
            win = new WindowState(this, session, client, token, attachedWindow, appOp[0],
              seq, attrs, viewVisibility, displayContent);
            (WindowManagerPolicy mPolicy 就是 PhoneWindowManager).adjustWindowParamsLw(win.mAttrs);
            mPolicy.prepareAddWindowLw(win, attrs);
            outInputChannel != null:
              InputChannel.openInputChannelPair(name);
              win.setInputChannel(inputChannels[0]);
              inputChannels[1].transferTo(outInputChannel);
              mInputManager.registerInputChannel(win.mInputChannel, win.minputWindowHandle);
            win.attach();
            mWindowMap.put(client.asBinder(), win);
            type == TYPE_BASE_APPLICATION:
              addWindowToListInOrderLocked(win, true);
            win.mWinAnimator.mEnterAnimationPending = true;
            if(displayContent.isDefaultDisplay) mPolicy.getContentInsetHintLw(attrs, outContentInsets);
            mInputMonitor.setUpdateInputWindowsNeededLw();
            if(win.canReceiveKeys()) focusChanged = updateFocusedWindowLocked();
            assignLayersLocked(displayContent.getWindowList());
          }
            -> WindowState#ctor(service, Session s, IWindow c, WindowToken token, WindowState attachedWindow, int appOp, int seq, WindowManager.LayoutParams a, int viewVisibility, final DisplayContent displayContent) {
              mSession = s;
              mWindowId = new IWindowId.Stub() {}
              (IWindow c).asBinder().linkToDeath(new DeathRecipient(), 0);
              type == TYPE_BASE_APPLICATION:
                mBaseLayer = mPolicy.windowTypeToLayerLw(a.type) * ... + ...;
                mSubLayer = 0;
              mWinAnimator = new WindowStateAnimator(this);
              mWinAnimator.mAlpha = a.alpha;
            }
            -> WindowState#attach() { mSession.windowAddedLocked(); }
              -> Session#windowAddedLocked() {
                mSurfaceSession = new SurfaceSession();
                (WMS mService).mSessions.add(this);
                mNumWindow++;
              }

  -> Activity#makeVisible() {
    mWindowAdded == true: pass
    mDecor.setVisibility(View.VISIBLE);
  }
    -> PhoneWindow.DecorView#setVisibility(View.VISIBLE) -> (父类View#)setVisibility(View.VISIBLE)
    -> (父类View#)setFlags(flags==View.VISIBLE, mask==VISIBILITY_MASK) {
        (changed & VISIBILITY_MASK) != 0:
          mPrivateFlags |= PFLAG_DRAWN;
          invalidate(true);
          needGlobalAttributesUpdate(true);
      }
    -> (父类View#)invalidate(true) {
      HardwareRenderer.RENDER_DIRTY_REGIONS == true:
        (p==mParent 这里是 ViewRootImpl).invalidateChild(this, r);
    }
      -> ViewRootImpl#invalidateChild(child==PhoneWindow.DecorView, dirty==r)
      -> ViewRootImpl#invalidateChildInParent(null, dirty==r) { mWillDrawSoon 初始化为false: scheduleTraversals(); }
      -> ViewRootImpl#scheduleTraversals() {
        mTraversalScheduled = true;
        mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL,
          mTraversalRunnable, null);// ViewRootImpl#doTraversal()
        scheduleConsumeBatchedInput();
      }


ViewRootImpl#mChoreographer 是 ThreadLocal 的, 依附在 ViewRootImpl 所在的 Looper 上.
USE_VSYNC 时用
(FrameDisplayEventReceiver mDisplayEventReceiver).scheduleVsync() --VSYNC--> onVsync()
  --Choreographer#mHandler--> doFrame() -> 从 CALLBACK_INPUT, CALLBACK_ANIMATION, CALLBACK_TRAVERSAL 队列取回调执行.

ViewRootImpl#doTraversal()
  -> ViewRootImpl#performTraversals() {
    mIsInTraversal = true;
    mWillDrawSoon = true;
    mFirst == ture:
      mFullRedrawNeeded = true;
      mLayoutRequested = true;
      attachInfo.mSurface = mSurface;
      (host==mView DecorView).dispatchAttachedToWindow(attachInfo, 0);
      ...
      host.fitSystemWindows();
    mLayoutRequest == true && !mStopped:
      ensureTouchModeLocally();
      measureHierarchy(host, lp, res, disiredWindowWidth, desiredWindowHeight);
    hadSurface = mSurface.isValid(); // == false, 下面relayoutWindow填充
    mFirst == true:
      relayoutWindow(params, viewVisibility, insetsPending);
    !mStopped:
      performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    didLayout:
      performLayout(lp, desiredWindowWidth, desiredWindowHeight);
    mFirst == true:
      mView.requestFocus(View.FOCUS_FORWARD);
    !hadSurface && mSurface.isValid(): { newSurface = true; mFullRedrawNeeded = true; }
    // If we are creating a new surface, then we need to
    // completely redraw it.  Also, when we get to the
    // point of drawing it we will hold off and schedule
    // a new traversal instead.
    mWillDrawSoon = false;
    ...
    scheduleTraversals(); // 第二遍时会调 performDraw()
    mIsInTraversal = false;
  }
    -> (DecorView 父类 ViewGroup)#dispatchAttachedToWindow(attachInfo, visibility==0) {
      for child: child.dispatchAttachedToWindow(info, ...); // (View child).mAttachInfo = info
      // 动态添加的view也可以在 addView() -> addViewInner() 时设置 mAttachInfo
    }
    -> (DecorView 父类 ViewGroup)#fitSystemWindows() {
      done = (super==View).fitSystemWindows(); // View#computeFitSystemWindows()
      if(!done):
        for child:
          done = child.fitSystemWindows();
    }
      -> (DecorView 父类 View)#fitSystemWindows() -> DecorView#internalSetPadding()
        -> View#internalSetPadding() { if(changed): requestLayout(); }
    -> ViewRootImpl#measureHierarchy() -> ViewRootImpl#performMeasure() { (DecorView mView).measure(); }
      -> (DecorView 父类View)#measure() -> DecorView#onMeasure() -> MeasureSpec#makeMeasureSpec()
      -> (父类FrameLayout)#onMeasure() {
        for(getChildCount()) measureChildWithMargins(child, ...);
        setMeasuredDimension(resolveSizeAndState(), ...);
      }
      -> (父类View)#measureChildWIthMargins() { child.measure(); }
      --如果有child--> (View或其子类)#measure() -> (View或其子类)#onMeasure() -> (View或其子类)#setMeasuredDimension()

    -> ViewRootImpl#relayoutWindow() { (IWindowSession mWindowSession).relayout(); }
    --Binder通信--> Session#relayout()
    -> WindowManagerService#relayoutWindow() {
      (WindowStateAnimator winAnimator).applyEnterAnimationLocked();
      outSurface.copyFrom(winAnimator.createSurfaceLocked());
      ...
      performLayoutAndPlaceSurfacesLocked();
    }
      -> WindowStateAnimator#createSurfaceLocked() {
        (WMS mService).makeWindowFreezingScreenIfNeededLocked(mWin);
        mSurfaceControl = new SurfaceControl(mSession.mSurfaceSession, attrs.getTitle(), w, h, format, flags);
        SurfaceControl.openTransaction();
        mSurfaceControl.setPosition(mSurfaceX, mSurfaceY);
        mSurfaceLayer = mAnimLayer;
        mSurfaceControl.setLayerStack(mLayerStack);
        mSurfaceControl.setLayer(mAnimLayer);
        mSurfaceControl.setAlpha(0);
      }
      -> WMS#performLayoutAndPlaceSurfacesLocked() {
          loopCount = 6;
          do {
            mTraversalScheduled = false;
            performLayoutAndPlaceSurfacesLockedLoop();
            ...
            loopCount--;
            } while(mTraversalScheduled && loopCount > 0);
        }
        -> WMS#performLayoutAndPlaceSurfacesLockedLoop() -> WMS#performLayoutAndPlaceSurfacesLockedInner() {
            ...
            performLayoutLockedInner(displayContent, repeats == 1, false);
            ...
            scheduleAnimationLocked();
          }
          -> WMS#performLayoutLockedInner() {
              (mPolicy 就是 PhoneWindowManager).beginLayoutLw(isDefaultDisplay, dw, dh, mRotation);
              // 计算 mNavigationBar.computeFrameLw() 和 mStatusBar.computeFrameLw()
              for WindowState win:
                !gone && !win.mLayoutAttached:
                  win.preLayout();
                  mPolicy.layoutWindowLw(win, win.mAttrs, null);
            }
            -> PhoneWindowManager#layoutWindowLw(WindowState win, win.mAttrs, attached==null) {
                win.computeFrameLw(pf, df, of, cf, vf);
                // 设置 mContainingFrame, mDisplayFrame, mParentFrame, mOverScanFrame,
                // mContentFrame, mVisibleFrame, mFrame, mOverscanInsets, mContentInsets, mVisibleInsets, mCompatFrame 等
              }
          -> WMS#scheduleAnimationLocked() {
              mAnimationScheduled = true;
              mChoreographer.postCallback(CALLBACK_ANIMATION, mAnimator.mAnimationRunnable, null;)
            }

    -> ViewRootImpl#performDraw() { draw(fullRedrawNeeded); 如果有mSurfaceHolder,调SurfaceHolder.Callback; }
    -> ViewRootImpl#draw(fullRedrawNeeded) {
        sFirstDrawComplete 初始化为false: for() mHandler.post(sFirstDrawHandlers.get(i));
        !dirty.isEmpty():
          attachInfo.mHardwareRenderer.isEnabled() 初始化为false, attachInfo.mHardwareRenderer.isRequested() 初始化== true:
            attachInfo.mHardwareRenderer.initializeIfNeeded();
            mFullRedrawNeeded = true;
            scheduleTraversals();
            // 下一遍 performTraversals() -> performDraw() -> draw() 时 attachInfo.mHardwareRenderer.draw(),
            // 就是 Gl20Renderer#draw()
      }
      -> (Gl20Renderer 父类HardwareRenderer)#initializeIfNeeded(width, height, surface) {
          isRequested() == true, !isEnabled() == true:
            initialize(surface);
            setup(width, height);
        }
        -> (Gl20Renderer 父类GlRenderer)#initialize(surface) {
            initializeEgl(); // sEgl = com.google.android.gles_jni.EGLImpl()
            mGl = createEglSurface(surface);
            mCanvas = createCanvas(); // GLES20Canvas
            setEnabled(true);
          }
          -> Gl20Renderer#createCanvas() { mGlCanvas = new GLES20Canvas(mTranslucent); }
          -> GLES20Canvas#ctor(record==false, translucent) {
              mRenderer = nCreateRenderer(); // C++: new OpenGLRenderer()
              setupFinalizer(); // new CanvasFinalizer(mRenderer)
            }

      -> (Gl20Renderer 父类GlRenderer)#draw(view, attachInfo, callbacks, dirty) {
          dirty = beginFrame(canvas, dirty, surfaceState); // canvas == mCanvas
          DisplayList displayList = buildDisplayList(view, mCanvas);
          status = prepareFrame(dirty);
          saveCount = canvas.save();
          callbacks.onHardwarePreDraw(canvas);
          displayList != null:
            status |= drawDisplayList(attachInfo, canvas, displayList, status);
          callbacks.onHardwarePostDraw(canvas);
          canvas.restoreToCount(saveCount);
          onPostDraw();
          swapBuffers(status);
          attachInfo.mIgnoreDirtyState = false;
        }
        -> (Gl20Renderer 父类HardwareRenderer)#beginFrame() -> nBeginFrame()
          --JNI--> android_view_HardwareRenderer_beginFrame() {
              EGLDisplay display = eglGetCurrentDisplay();
              EGLSurface surface = eglGetCurrentSurface(EGL_DRAW);
              eglBeginFrame(display, surface);
            }
        -> (Gl20Renderer 父类Glrenderer)#buildDisplayList(view, canvas) { view.getDisplayList(); }
          -> View#getDisplayList() -> View#getDisplayList(mDisplayList, isLayer==false) {
              if displayList!=null: // 非第一次
                dispatchGetDisplayList();
                // DecorView 父类 ViewGroup#dispatchGetDisplayList { for child: child.getDisplayList(); }
              else: // 第一次
                !isLayer: mRecreateDisplayList = true;
                displayList = (Gl20Renderer mAttachInfo.mHardwareRenderer).createDisplayList(name); // new GLES20DisplayList
                invalidateParentCaches(); // mParent.mPrivateFlags |= PFLAG_INVALIDATED
              HardwareCanvas canvas = displayList.start(width, height); // GLES20RecordingCanvas
              layerType 初始化为 LAYER_TYPE_NONE:
                computeScroll();
                canvas.translate(-mScrollX, -mScrollY);
                (View#)draw(canvas);
              displayList.end();
              displayList.setCaching(caching);
              setDisplayListProperties(displayList);
            }
            -> GLES20DisplayList#start(width, height) {
                mCanvas = GLES20RecordingCanvas.obtain(this); // GLES20RecordingCanvas
                mCanvas.start(); // { mDisplayList.mBitmaps.clear(); mDisplayList.mChildDisplayLists.clear(); 都是ArrayList}
                mCanvas.setViewport(width, height);
                mCanvas.onPreDraw(null); // --JNI--> nPrepare() --C++--> (OpenGlRenderer renderer)->prepareDirty()
              }
              -> GLES20RecordingCanvas#obtain(DisplayList) {
                  new GLES20RecordingCanvas();
                  // 父类 GLES20Canvas(true, true) { nCreateDisplayListRenderer(); // C++:new DisplayListRenderer }
                }
            -> (DecorView 父类View)#draw(canvas) { // DecorView#draw() -> FrameLayout#draw() -> ViewGroup#draw()
                // light version
                onDraw(canvas);
                dispatchDraw(canvas);
                onDrawScrollBars(canvas);
                // full version
                canvas.saveLayer();
                onDraw(canvas);
                dispatchDraw(canvas);
                canvas.drawRect();
                onDrawScrollBars(canvas);
              }
              -> ViewGroup#dispatchDraw() {
                  for child: drawChild(canvas, child, drawingTime) -> (View child).draw(canvas, this, drawingTime)
                }
                -> View#draw(canvas, parent==ViewGroup, drawingTime) {
                    useDisplayListProperties == true;
                    hardwareAccelerated == true: // GLES20Canvas 父类HardwareCanvas 直接返回
                      caching = true;
                    layerType == LAYER_TYPE_NONE: 初始化值
                      hasDisplayList = canHaveDisplayList(); // true
                    useDisplayListProperties &= hasDisplayList; // true
                    useDisplayListProperties == true:
                      displayList = getDisplayList(); // !!递归调自己
                    hasNoCache == true:
                      (HardwareCanvas 子类 GLES20RecordingCanvas canvas).drawDisplayList(displayList, null, flags);
                  }
              -> GLES20RecordingCanvas#drawRect() {
                super.drawRect(); 
                // (父类 GLES20Canvas)#drawRect() -> nDrawRect() --JNI--> (OpenGLRenderer renderer).drawRect()
                recordShaderBitmap(paint); // mDisplayList.mBitmaps.add()

            -> GLES20DisplayList#end() {
                (GLES20RecordingCanvas mCanvas).onPostDraw();
                mFinalizer == null:
                  mFinalizer = new DisplayListFinalizer(mCanvas.end(0));
                mCanvas.recycle();
                mCanvas = null;
                mValid = true;
              }
              -> GLES20RecordingCanvas#end(0) -> (父类 GLES20Canvas)#getDisplayList(0)
                --JNI--> (DisplayListRenderer renderer).getDisplayList(DisplayList displayList==NULL) {
                  C++: return new DisplayList(*this); }
              -> GLES20RecordingCanvas#recycle() {
                  mDisplayList = null;
                  resetDisplayListRenderer(); // (父类 GLES20Canvas)#resetDisplayListRenderer() -> nResetDisplayListRenderer
                  sPool.release(this); // 放入缓存池, 方便下次重用
                }
        -> (Gl20Renderer 父类GlRenderer)#prepareFrame(dirty) -> Gl20Renderer#onPreDraw(dirty)
          -> (GLES20Canvas mGlCanvas)#onPreDraw(dirty) --JNI--> nPrepareDirty()
            -> (OpenGLRenderer renderer).prepareDirty() {
                setupFrameState();
                if(mSnapshot->fbo == 0)
                  syncState();
                  updateLayers();
                else
                  startFrame();
              }
              -> OpenGlRenderer::setupFrameState() {
                  mCaches.clearGarbage();
                  mSnapshot = new Snapshot();
                  mSnapshot->fbo = getTargetFbo();
                }
        -> GLES20Canvas#drawDisplayList() --JNI--> nDrawDisplayList()
          --C++--> (OpenGLRenderer renderer)::drawDisplayList() {
              if(CC_UNLIKELY(mCaches.drawDeferDisabled)):
                status = startFrame();
                displayList.replay(replayStruct, 0);
                return status | replayStruct.mDrawGlStatus;
              displayList.defer(deferStruct, 0);
              flushLayers();
              status = startFrame();
              return status | deferredList.flush(*this, dirty);
            }
        -> (Gl20Renderer 父类Glrenderer)#swapBuffers(status) {
            if(status & DisplayList.STATUS_DREW)
              (EGL10 sEgl).eglSwapBuffers(sEglDisplay, mEglSurface);
              checkEglErrors();
          }
          -> (EGL10 sEgl)#eglSwapBuffers() --JNI--> jni_eglSwapBuffers() --C++--> eglSwapBuffers(dpy, sur)