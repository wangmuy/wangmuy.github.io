---
layout: post
title: "anatomy-of-recyclerview"
date: 2014-07-26 17:23:01 +0800
comments: true
categories: [android]
keywords: 
description: 
#published: false
---

<pre>

版本: 25.1.0

Scroll
============================================================

# 调用顺序

RecyclerView#requestChildFocus()
-> RecyclerView#requestChildRectangleOnScreen()
-> LayoutManager#requestChildRectangleOnScreen()
-> RecyclerView#smoothScrollBy(int, int)


# 回调

## 添加/删除

addOnScrollListener/removeOnScrollListener (OnScrollListener)

## 调用

### ScrollListener.onScrolled()

<- RecyclerView.dispatchOnScrolled()
1. <- RecyclerView.scrollByInternal()
  1.1 <- scrollBy(x, y)
    1.1.1 <- RV.LayoutManager.requestChildRectangleOnScreen() { dx!=0 || dy!=0 }
    1.1.2 <- RV.LayoutManager.performAccessibilityAction() { vScroll != 0 || hScroll != 0 }
  1.2 <- RV.onTouchEvent() ACTION_MOVE 触摸移动
  1.3 <- RV.onGenericMotionEvent() SOURCE_CLASS_POINTER ACTION_SCROLL 鼠标滚轮
2. <- RecyclerView.dispatchLayoutStep3() { dispatchOnScrolled(0,0) } layout引起的 item range changed
3. <- RV.ViewFlinger.run() {
  hresult=LayoutManager.scrollHorizontallyBy(); 线性布局 会调到 LinearLayoutManager.scrollBy()
  vresult=LayoutManager.scrollVeriticallyBy();
  (hresult != 0 || vresult != 0) dispatchOnScrolled() }

### RV.smoothScrollBy()
-> RV.ViewFlinger.smoothScrollBy()


### RV.scrollToPosition()
-> LayoutManager.scrollToPosition()


onBindViewHolder()
============================================================
RV.Adapter.onBindViewHolder(holder, position)
1. <- RV.Adapter.onBindViewHolder(holder, position, payloads)
2. <- RV.Adapter.bindViewHolder(holder, position)
  <- RV.Recycler.tryBindViewHolderByDeadline() 前提: 判断 RecyclerPool.willBindInTime()
  2.1 <- RV.Recycler.bindViewToPosition()
  2.2 <- RV.Recycler.tryGetViewHolderForPositionByDeadline()
    <- RV.Recycler.getViewForPosition()
    2.2.1 <- LayoutState.next()
    2.2.2 <- LinearLayoutManager.next()
      <- LLM.layoutChunk()
      <- LLM.fill()
      2.2.2.1 <- LLM.onLayoutChildren()
        2.2.2.1.1 <- RV.dispatchLayoutStep2()
          <- 2.2.2.1.1.1 RV.dispatchLayout()
            <- RV.onLayout() # 一般调用位置
          <- 2.2.2.1.1.2 RV.onMeasure() 只在 LayoutManager.setAutoMeasureEnabled(true)
        2.2.2.1.2 <- RV.dispatchLayoutStep1()
      2.2.2.2 <- LLM.layoutForPredictiveAnimations()
      2.2.2.3 <- LLM.scrollBy()
      2.2.2.4 <- LLM.onFocusSearchFailed()



版本: 23.0.1

从notifyDatasetChanged() 到实际的view被重新添加和layout
============================================================
问题:
什么时候加进动画?
Adapter.notifyDataSetChanged() { (AdapterDataObservable mObservable).notifyChanged() } 每个 Observable.onChanged()
-> RecyclerViewDataObserver.onChanged() {
    setDataSetChangedAfterLayout() 每个没有shouldIgnore的viewHolder(现有的和mRecycler的) 添加 FLAG_ADAPTER_POSITION_UNKNOWN
    if(没有pendingUpdates) requestLayout()
--UI Handler下一次处理--> RV.onLayout() -...-> onBindViewHolder()
}


notifyItemRemoved(pos) 到实际的view被删除和重新layout
============================================================
Adapter.notifyItemRemoved(pos) { mObservable.notifyItemRangeRemoved(pos,1) } 每个 Oberservable.onItemRangeRemoved()
-> RecyclerViewDataObserver.onItemRangeRemoved() {
  AdapterHelper.onItemRangeRemoved()一般==true: 加到AdapterHelper.mPendingUpdates数组
    triggerUpdateProcessor() }
-> RecyclerViewDataObserver.triggerUpdateProcessor() { 一般mHasFixedSize==false: mAdapterUpdateDuringMeasure=true; requestLayout() }

--UI Handler下一次处理--> RV.onMeasure() {
  mAdapterUpdateDuringMeasure==true: ... processAdapterUpdatesAndSetAnimationFlags(); ... mLayout.onMeasure() }
  -> RV.processAdapterUpdatesAndSetAnimationFlags() {
  ... mItemAnimator != null && mLayout.supportsPredictiveItemAnimations(): mAdapterHelper.preProcess(); .. }
    -> AdapterHelper.preProcess()
      -> AdapterHelper.applyRemove(op) { type==POSITION_TYPE_NEW_OR_LAID_OUT: postponeAndUpdateViewHolders(op) }
      -> AdapterHelper.postponeAndUpdateViewHolders(op) { 加到mPostponedList; (RV内部匿名实例 mCallback).offsetPositionsForRemovingLaidOutOrNewView() }
      -> RV.offsetPositionRecordsForRemove(posStart, count, false); RV.mItemsAddedOrRemoved=true
      -> { holder.flagRemovedAndOffsetPosition(); mRecycler.offsetPositionRecordsForRemove(); requestLayout() }
      -> Recycler.offsetPositionRecordsForRemove()
      -> Recycler.recycleCachedViewAt(idx) { addViewHolderToRecycledViewPool(); mCachedViews.remove(idx) }
    -> 设置 animationTypeSupported=true; 设置 mState.mRunSimpleAnimations=true; mState.mRunPredictiveanimations=true
  -> RV.onLayout()
  -> RV.dispatchLayout() {
    processAdapterUpdatesAndSetAnimationFlags(); ...
    mLayout.onLayoutChildren(); 可能修改 mState.mRunSimpleAnimations
    if(mState.mRunSimpleAnimations) { ...
      animateDisappearance(disappearingItem) 每个disappearing and removed item
      animateAppearance(itemHolder) 每个appearing and added item
      animateMove(postHolder) 每个persistent item
      animateChange(oldHolder) 每个changing item
    }
  -> RV.animateDisappearance(holder) {
    addAnimatingView(holder)
    if(没有真的移除) mItemAnimator.animateMove()==true: postAnimationRunner()
    else(真的移除) holder.setIsRecyclable(fasle); mItemAnimator.animateRemove()加到mPendingRemovals==true: postAnimationRunner()
    }
    -> addAnimatingView() { mRecycler.unscrapView(holder); 一般 mChildHelper.hide(view) }
      -> ChildHelper.hide() -> ChildHelper.hideInternal(view) { 加到 mHiddenViews }
    -> postAnimationRunner() { 动画runnable post到choreographer上 }
  }
--choreographer下一帧--> 执行动画 RV.mItemAnimatorRunner
-> DefaultItemAnimator.runPendingAnimations() { animateRemoveImpl(每个removed的holder); ... }
-> DefaultItemAnimator.animateRemoveImpl() { view.animate().alpha(0).setDuration(默认120ms).start() }
--动画执行完--> 动画回调 onAnimatinoEnd() { view.setalpha(1); dispatchRemoveFinished(holder); dispatchFinishedWhenDone() }
  -> (DefaultItemAnimator 父类 ItemAnimator).dispatchRemoveFinished() { mListener.onRemoveFinished() }
  -> RV.(ItemAnimatorRestoreListener mItemAnimatorListener).onRemoveFinished() {
    item.setIsRecyclable(true); removeAnimatingView(view) 或 removeDetachedView(view) }
    -> RV.removeAnimatingView(view) {
      mChildHelper.removeViewIfHidden(view);
      if(removed): mRecycler.unscrapView(); mRecycler.recycleViewHolderInternal() }
      -> ChildHelper.removeViewifHidden() {
        从mBucket移除; unhideViewInternal(view); (mCallback==RV匿名).removeViewAt(idx) 真正从RV移除 }
        -> ChildHelper.unhideViewInternal() { 从 mHiddenViews 移除 }
</pre>