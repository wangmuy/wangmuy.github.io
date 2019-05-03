---
layout: post
title: "Fresco 源码分析"
date: 2016-09-07 11:03:58 +0800
categories: [android]
tags: [android, fresco]
description: 
---

# SimpleDraweeView 显示流程

```
SDV: SimpleDraweeView
PDCBuilder: PipelineDraweeControllerBuilder
ADCBuilder: AbstractDraweeControllerBuilder
ADController: AbstractDraweeController
```

## 设置

```
SDV.setImageURI() {
  controller = PDCBuilder.setUri(uri).setOldController(getController()).build()
  setController(controller)
}
-> (PDCBuilder 父类 ADCBuilder).buildController()
  -> PDCBuilder.obtainController() { 清空现在的listeners }
  -> ADCBuilder.maybeAttachListeners(controller) {
    只剩 ADCBuilder.mBoundControllerListeners, 此成员在 ADCBuilder ctor中赋值为null
    }
-> (SDV 父类 DraweeView).setController()
  -> DraweeHolder.setController()
```

## 触发

```
在 WindowManger系统 SDV.onAttach() 时
(SDV 父类 DraweeView).onAttach()
-> DraweeView.doAttach()
  -> DraweeHolder.onAttach()
    -> DraweeHolder.attachOrDetachController()
      -> DraweeHolder.attachController()
        -> AbstractDraweeController.onAttach()
          -> ADController.submitRequest() {
            若cache到, 直接调用 onNewResultInternal();
            若无cache, mDataSource.subscribe(DataSubscriber), 在 onNewResultImpl()调用 onNewResultInternal() 或 onFailureInternal() }
-回调-> ADController.onNewResultInternal() {
  mSettableDraweeHierarchy.setImage(drawable..)
  isFinished: { getControllerListener().onFinalImageSet() }
  否则: { getControllerListener().onIntermediateImageSet() }
```

## 获取

### 线程

```
Fresco.initialize()
-> ImagePipelineFactory.initialize(ImagePipelineConfig) { new ImagePipelineFactory() }
  -> ImagePipelineFactory.ctor() { mThreadHandoffProducerQueue = DefaultExecutorSupplier.mLightWeightBackgroundExecutor }
```

### 提交到线程

```
(父类 ADController).submitRequest()
-> PipelineDraweeController.getDataSource()
  -> PDCBuilder.getDataSourceForRequest()
    -> ImagePipeline.fetchDecodedImage()
      -> ImagePipeline.submitFetchRequest() { ClosableProducerToDataSourceAdapter.create() }
        -> (ClosableProducerToDataSourceAdapter 父类 AbstractProducerToDataSourceAdapter).ctor()
          -> BitmapMemoryCacheGetProducer.produceResults()
            -> ThreadHandoffProducer.produceResults()
              -> ThreadHandoffProducerQueue.addToQueueOrExecute() { mExecutor.execute() }
                -executor线程-> StatefulProducerRunnable ThreadHandoffProducer.statefulRunnable.onSuccess()
                  -> (BitmapMemoryCacheKeyMultiplexProducer 父类 MultiplexProducer).produceResults()
                    -> MultiplexProducer.startInputProducerIfHasAttachedConsumers()
                      -> BitmapMemoryCacheProducer.produceResults()
                        -> DecodeProducer.produceResults()
                          -> LocalImageProgressiveDecoder.ctor()
                            -> 父类 ProgressiveDecoder.ctor() { JobRunnable 此job被执行时 doDecode() }


AbstractDataSource.subscribe() {
  mDataSourceStatus在ctor赋初始值 IN_PROGRESS:
    mSubscribers.add(Pair(dataSubscriber, ADController.mUiThreadImmediateExecutor))
  shouldNotify==false: 忽略 }

JobScheduler
```