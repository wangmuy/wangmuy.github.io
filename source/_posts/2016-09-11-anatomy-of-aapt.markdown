---
layout: post
title: "anatomy-of-aapt"
date: 2016-09-11 16:42:19 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---
# 概念
```
Resource Id 32位 = 8(packageId) + 8(typeId) + 16(nameId)
ResourceTable.cpp::makeResId()
```

# framework-res典型调用
```bash
out/host/linux-x86/bin/aapt package -u -x -z  --pseudo-localize   -M frameworks/base/core/res/AndroidManifest.xml -S device/myvendor/overlay/frameworks/base/core/res/res -S frameworks/base/core/res/res -A frameworks/base/core/res/assets  --min-sdk-version 22 --target-sdk-version 22 --product phone --version-code 22 --version-name 5.1.1-2.0.16   -F out/target/common/obj/APPS/framework-res_intermediates/package-export.apk
```

# 流程

## 总体流程

```
Main.cpp main() { 收集命令行内容到bundle }
-> handleCommand(bundle)
-> Command.cpp doPackage(bundle) {
  assets=new AaptAssets();
  assets.slurpFromArgs(bundle) 收集所有文件信息
  builder = new ApkBuilder(configFilter)
  if(有资源文件) buildResources(bundle, assets, builder) 收集和编译resource, 生成id
  assets.applyJavaSymbols() { 决定哪些内容会输出到 R.java
		若是framework-res, AaptAssets.mHavePrivateSymbols==false 没有排除任何内容
		若是普通应用, 这里排除掉所有framework中定义的私有内容. }
  writeResourceSymbols(bundle, assets, ...)
  writeProguardFile(bundle, assets)
  if(outputAPKFile) addResourcesToBuilder(assets, builder) 生成apk }

-> Resource.cpp buildResources() {
  parsePackage(androidManifestFile) 取包名, revisionCode, minSdk
  packageType = 命令行 -x==System, --shared-lib==SharedLibrary, --feature-of==AppFeature
  table = ResourceTable(bundle, assets.getPackage(), packageType)
  table.addIncludedResources(bundle, assets)

  applyFileOverlay() 作用于 OVERLAY_SET=drawable/layout/anim/animator/interpolator/transition/xml/raw/color/menu/mipmap, 这些目录若有overlay, 对应文件替换成overlay中的文件(文件整体替换)

  对drawable/mipmap preProcessImages()
  对OVERLAY_SET makeFileResources(bundle, assets, &table, ...)

  编译每个overlay中的values目录里的xml文件
  foreach(overlay)
    compileResourceFile(bundle, assets, file, it.getParams(), overlay=true, &table)

  table.assignResourceIds() {
		package.applyPublicTypeOrder() {
			public类型 的按 Type.mPublicIndex 重新排序(之后的 p.getOrderedTypes() 返回的就是已排序Type)
			Type.mPublicIndex 通过 addPublic() 的顺序设置, 就是在 values/public.xml 中的定义顺序 }
		遍历 p.getOrderedTypes(): 每个type.getOrderedConfigs() 的 entry 生成 attribute
		遍历 p.getOrderedTypes():
		  每个type.applyPublicEntryOrder() 按 entryId(即资源id中的nameId) 重新排序(之后的 t.getOrderedConfigs() 返回就是已排序ConfigList)
			按顺序 ConfigList.setEntryIndex()
		按顺序遍历 每个 type中的 ConfigList中的 Entry: entry.assignResouceIds() 赋值bag中每个key的资源id }

  编译 layout/anim/animator/interpolator/transition/xml/drawable/color/menu 中的xml文件
  compileXmlFile()

  编译自动生成的xml
  编译manifest
  生成最终resource table: table.addSymbols(assets->getSymbolsFor("R")) 按已排序好的 Type 和 ConfigList 添加 symbol
	重新flatten, 输出到resources.arsc
  validate检查一遍
}

-> ResourceTable.cpp compileResourceFile(*bundle, &assets, ..., *outTable) {
  设 symbols = asset.getSymbolsFor("R")
  switch(标签)
	case "skip": 跳过
	case "eat_comment": 跳过
	case "public": {
	  取type,name,id. outTable.addPublic() -> Type.addPublic()
		Type.mPublicIndex = typeId(即资源id中间8位值)
	  symbols.addNestedSymbol(type, srcPos)
	  symbols.makeSymbolPublic(name, srcPos) }
	case "public-padding": 略
	case "private-symbols": 略
	case "java-symbol": 略
	case "add-resouce": 略
	case "declare-stylable": {
	  取name
	  symbols.addNestedSymbol("stylable", srcPos)
	  symbols.addNestedSymbol(name, srcPos)
	  遍历XML子tag: compileAttribute(file, block, ..., &itemIdent, inStylable==true)
	  symbols.addSymbol(itemIdent, 0, srcPos) }
	case "attr": compileAttribute(file, block, myPackage, outTable, outIdent==NULL, inStylable==false)
	case "item": 略
	case "string": 略
	case "drawable": curTag=curType="drawable", curFormat=TYPE_REFERENCE|TYPE_COLOR
	case "color": curTag, curType, curFormat
	case "bool"/"integer"/"dimen"/"fraction"/"style"/"plurals": 基本同上
	case "bag": curType=取type, curIsBag=true
	case "array": curFormat=parse_flags(取format)
	case "string-array": 略
	case "integer-array": 略

	对bag/style/plurals/array/string-array/integer-array: curIsBag=true 解释: bag其实就是 自定义枚举集合 的意思
	对array/string-array/integer-array: curIsBagReplaceOnOverwrite=true

	if(curIsBag)
	  outTable.startBag()
	  parseAndAddBag() {
	  	outTable.addBag() overlay overwrite==true时, framework已有的可以覆盖, 没有的要用add-resource }
	else
	  parseAndAddEntry(ResourceTable *outTable) {
	  	对于overlay, 传入的overwrite==true, framework已有的可以覆盖, 没有的要用add-resource
			outTable.addEntry() 实际添加到ResourceTable中对应 Type 的
			  (DefaultKeyedVector<String16, sp<ConfigList> > mConfigs).valueFor(entry)
			}

  检查确认每个resource都有 default variant
}

-> compileAttribute() {
	attr = PendingAttribute(myPackage, inFile, block, inStylable)
	取name, format
	attr.createIfNeeded(outTable)
	取min, max, localization, 若有则 outTable.addBag()
	遍历XML子tag:
	  非 TYPE_ENUM或TYPE_FALGS, 报错
	  if 第一次: outTable.addBag(bagKey=="^type", value==type数字, replace==true)
	  outTable.addBag(bagKey==当前enum/flag的name, value==XML value字段, replace==false)
	appendTypeInfo()
	outTable.appendTypeComment() }

-> PendingAttribute.createIfNeeded() {
	if(added) return;
	added = true
	outTable.addBag(bagKey=="^type", value==type数字, replace==false) }
	可以得出
	  attr标签 format/min/max/localization 做一个bag, item是自己;
	  否则 若有子tag, 子tag必须是enum/flag, 做一个bag, 每个子tag做成一个item

-> addBag(srcPos, package, type, name, bagParent, bagKey, value, *style, &params, replace, isId, format){
	e=ResourceTable::getEntry(package, type, name, srcPos, replace, params)
	e.addToBag(sourcePos, bagKey, value, style, replace, isId, format) }

-> ResouceTable::getEntry(overlay==replace) {
    Type t=getType(package, type, sourcePos, doSetIndex)
    t.getEntry(name, sourcePos, config, dotSetIndex, overlay, bundle.autoOverlay) }

-> Entry.addToBag(...) {
	makeItABag(srcPos) 将自己的mType设为 TYPE_BAG
	item=Item(srcPos, isId, value, style, format)
	if(已有key):
	  if(!replace) 报错
	  else mBag.replaceValueFor(key, item)
	mBag.add(key, item)
	mBag是Entry成员, 类型是 KeyedVector<String16, Item> }
	Item 类型有 TYPE_BAG/TYPE_ITEM, 当类型是TYPE_BAG时 mBag 包含当前bag的所有item


ResourceTable.cpp compileXmlFile(*bundle, &assets, *table, XMLNode& root, AaptFile& target) {
  root.assignResouceIds(assets, table) 设置的 id 会在 flatten 时用到
	root.parseValues(assets, table) 记录当前解析的xml源码行号
}
```

# 写到 R.java 的流程

```
Command.cpp::doPackage() { framework 生成时包名就是自身, 所以 havePrivateSymbols() 为 false }
-> Resource.cpp::WriteResourceSymbols(AaptAssets assets, includePrivate=true) {
	遍历 assets->getSymbols(): 这里的symbol就是上面的 addSymbol() 加的
	  可能不同 类前缀包名/R.java 输出位置
	  WriteSymbolClass(AaptSymbols symbols)
}
-> WriteSymbolClass(AaptSymbols symbols) {
	  // framework-res 编译过程中 havePrivateSymbols()==false, 所以下面的遍历没有跳过任何内容
	  遍历 symbols->getSymbols() 所有 TYPE_INT32 类型(跳过非 javaSymbol): (AaptSymbolEntry sym).int32Val
	  遍历 symbols->getSymbols() 所有 TYPE_STRING 类型(跳过非 javaSymbol): sym.stringVal.string()
		遍历 symbols->getNestedSymbols(), styleableSymbols除外: 递归 WriteSymbolClass()
		writeLayoutClasses(styleableSymbols)
		if emitCallback: writeResourceLoadedCallback() }
  -> 递归调用 WriteSymbolClass
```

{% comment %}

# overlay 的 values 目录 添加某些xml标签出错
## 原因
overlay中定义的 attr 若和framework中相同, 会在 compileAttribute() 报错

## 解决
目标
cp -r frameworks/base/core/res/res/ device/myvendor/overlay/frameworks/base/core/res/ 全部拷贝过来都可用

1. attr 重复定义报错
PendingAttribute::createIfNeeded() 注释掉重复报错
2. overlay attr createIfNeeded() addBag() 重复添加bag报错
createIfNeeded() outTable.addBag() 调用加参数 replace==true
3. 第2步添加replace后再添加attr时 传到 getEntry() 的 overlay==replace==true 导致
-x 时自动添加 autoAddOverlay(代码 or 脚本 实现?)

!!! 2,3步 可以改为 compileAttribute()多传一个 isOverlay=compileResouceFile()的overwrite参数, replace=isOverlay

4. compileAttribute() addBag() 重复添加item报错
compileAttribute() 所有addBag() 调用加参数 replace==true
5. 验证无attr重复定义时可以正确编译
6. 验证整个framework res拷贝过来 可以正确编译
7. 验证编普通应用(如Launcher2) 可以正确编译

{% endcomment %}