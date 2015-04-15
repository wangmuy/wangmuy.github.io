---
layout: post
title: "libgdx-dynamic-freetype-flow"
date: 2015-01-19 14:01:02 +0800
comments: true
categories: [libgdx]
keywords: 
description: 
#published: false
---

<pre>

1. JSON读取
Skin.getJsonLoader() -> new Json() -> json.setSerializer()
  读json文件size属性, new FreeTypeBitmapFontStub()
  FreeTypeBitmapFontStub extends BitmapFont
  FreeTypeBitmapFontStub.data instanceof FreeTypeBitmapFontData

2. Label初始化
new Label() -> Label.setStyle() {
  FreeTypeFontManager.isFreeTypeFont(style.font)==true {
    data.setSize(style.fontSize)
    new FontInfo(data.getSize())
    font = FreeTypeFontManager.getInstance().refresh(style.font==stub, newInfo)
  }
  cache = new BitmapFontCache(font) // cache里的 font == 第一次refresh出来的

3. 更新文字
Label.setText() -> Widget.invalidateHierarchy()
-> Label.invalidate() { needsLayout=true; sizeInvalid=true }
渲染 -> draw() -> Widget.validate() -> Label.layout()
-> computeSize() -> cache.requireSequence(text) {
  FreeTypeFontManager.isFreeTypeFont(this.font) == true:
    this.font = FreeTypeFontManager.getInstance().refresh(this.font, curFontInfo) }
-> FreeTypeFontManager.refresh(font, newInfo) {
  isStub == false
  newInfo不含在当前子串里:
    getFont(newInfo.size)
}

</pre>