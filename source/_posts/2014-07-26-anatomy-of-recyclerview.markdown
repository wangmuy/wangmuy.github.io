---
layout: post
title: "anatomy-of-recyclerview"
date: 2014-07-26 17:23:01 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---

<pre>

Scroll 时的调用顺序
============================================================

RecyclerView#requestChildFocus()
-> RecyclerView#requestChildRectangleOnScreen()
-> LinearLayoutManager#requestChildRectangleOnScreen()
-> RecyclerView#smoothScrollBy(int, int)

</pre>