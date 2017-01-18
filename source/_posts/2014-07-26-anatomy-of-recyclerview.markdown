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

</pre>