---
layout: post
title: "libgdx-skin-load-anatomy"
date: 2014-09-22 17:35:06 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---

<pre>

构造函数里设置 SkinLoader
AssetManager.ctor() { setLoader(Skin.class, new SkinLoader(resolver==new InternalFileHandleResolver())) }
-> AssetManager.setLoader(Skin.class, suffix==null, SkinLoader)

用户程序调 load
AssetManager.load(fileName, Skin.class)
-> AssetManager.load(fileName, Skin.class, parameter==null) {
  getLoader(Skin.class, fileName);
  loadQueue.add(new AssetDescriptor(fileName, Skin.class, parameter))
}

用户程序调 finishLoading
AssetManager.finishLoading() { while(!update()) ThreadUtils.yield() }
-> AssetManager.update() { nextTask(); updateTask(); }
AssetManager.nextTask() { assetDesc = loadQueue.removeIndex(0); addTask(assetDesc) or inc ref }
AssetManager.updateTask() { task.update(); addAsset(fileName, type, task.getAsset()) }
-> AssetLoadingTask.update() { handleSyncLoader() or handleAsyncLoader() }
SkinLoader 是 AsynchronousAssetLoader, 这里会调用 handleAsyncLoader()
-> AssetLoadingTask.handleAsyncLoader() {
    // 执行多次,  因为submit到线程池中, 被调 call().
    // SkinLoader.getDependencies() 加入装载 TextureAtlas 的依赖
    exec0:
	dependenciesLoaded == false && depsFuture == null:
	    depsFuture = (AsyncExecutor executor).submit(this)
	exec1:
	dependenciesLoaded == false && depsFuture.isDone():
	    dependenciesLoaded = true
	可能的exec2:
	loadFuture == null && !asyncDone:
	    loadFuture = executor.submit(this)
	    // 会在线程池中执行 AssetLoadingTask.call() -> 即本loader的 loadAsync()
	最后的exec:
	asyncDone:
	    asset = asyncLoader.loadSync(manager, fileName, resolve(loader, assetDesc), params)
}

SkinLoader.loadSync() { new Skin(atlas).load(file) }
-> Skin.load(skinFile) { getJsonLoader().fromJson(Skin.class, skinFile) }
-> Json.fromJson(type==Skin.class) { this.readValue(type, null, new JsonReader().parse(file)) }

JsonReader.parse(file) { JsonReader.parse(file.reader("UTF-8")) }
-> JsonReader.parse(Reader reader) -> JsonReader.parse(data, 0, offset)

FileHandle.reader(charset) { return new InputStreamReader(read(), charset) }
FileHandle.read() { return new FileInputStream(file()) }
FileHandle.file() { return java.io.File }

Skin.getJsonLoader(skinFile) { return 匿名Json新类 }
匿名Json类.readValue(Skin.class, null, jsonData) {
	jsonData.isString() == false:
	    super.readValue(Skin.class, elementType==null, jsonData)
}

Json.readValue(Skin.class, elementType==null, jsonData) {
	jsonData.isObject() == true:
	    typeName == null: className = null
	    serializer = classToSerializer.get(Skin.class) // 在Skin.java匿名类中有SetSerializer
	    serializer.read(this, jsonData, type)
}
-> 匿名Json类.read() { for(valueMap) readnamedObjects(json, ClassReflection.forName(valueMap.name()), valueMap) }

</pre>