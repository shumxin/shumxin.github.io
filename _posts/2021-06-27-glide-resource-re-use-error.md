---
layout: post
title:  "Glide 资源重用错误 - Cannot draw a recycled Bitmap"
date:   2021-06-27 13:27:28 +0800
categories: android
---

## 一. 问题描述

### 引言

最近看了我司某产品的 Bugly 上的崩溃分析统计，注意到 `Canvas: trying to use a recycled bitmap android.graphics.Bitmap@e9a8c74` 异常有点醒目。

![bugly_crash_report](/res/images/210627_bugly_crash_report.png){:height="138px" width="720px"}

### 问题调用栈

```java
java.lang.RuntimeException: Canvas: trying to use a recycled bitmap android.graphics.Bitmap@de87ba3
    at android.graphics.BaseCanvas.throwIfCannotDraw(BaseCanvas.java:66)
    at android.graphics.MiuiCanvas.throwIfCannotDraw(MiuiCanvas.java:329)
    at android.graphics.RecordingCanvas.throwIfCannotDraw(RecordingCanvas.java:277)
    at android.graphics.BaseRecordingCanvas.drawBitmap(BaseRecordingCanvas.java:88)
    at app.abk.draw(SourceFile:101)
    at android.widget.ImageView.onDraw(ImageView.java:1436)
    at android.view.View.draw(View.java:22489)
    at android.view.View.updateDisplayListIfDirty(View.java:21357)
    at android.view.View.draw(View.java:22217)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4540)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4299)
    at android.view.View.updateDisplayListIfDirty(View.java:21348)
    at android.view.View.draw(View.java:22217)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4540)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4299)
    at android.view.View.draw(View.java:22492)
    at android.view.View.updateDisplayListIfDirty(View.java:21357)
    at android.view.View.draw(View.java:22217)
    at android.view.ViewGroup.drawChild(ViewGroup.java:4540)
    at android.view.ViewGroup.dispatchDraw(ViewGroup.java:4299)
    at android.widget.AbsListView.dispatchDraw(AbsListView.java:2705)
    at android.view.View.draw(View.java:22492)
    at android.widget.AbsListView.draw(AbsListView.java:4448)
    ...
```

查看对应版本的 mapping 文件确认 `app.abk` 对应 `com.bumptech.glide.load.resource.bitmap.GlideBitmapDrawable`

## 二. 问题分析

### 1. 提取主要信息

正常来说根据异常信息的堆栈很容易定位到相关业务代码的出错位置，而该堆栈信息乍一看有点懵

首先对于 `Canvas: trying to use a recycled bitmap android.graphics.Bitmap@xxxxxxxx；`

```java
protected void throwIfCannotDraw(Bitmap bitmap) {
    if (bitmap.isRecycled()) {
        throw new RuntimeException("Canvas: trying to use a recycled bitmap " + bitmap);
    }
    if (!bitmap.isPremultiplied() && bitmap.getConfig() == Bitmap.Config.ARGB_8888 &&
            bitmap.hasAlpha()) {
        throw new RuntimeException("Canvas: trying to use a non-premultiplied bitmap "
                + bitmap);
    }
    throwIfHwBitmapInSwMode(bitmap);
}
```

原因是 `ImageView#onDraw` 的时候尝试使用已回收的 `bitmap` 进而造成该崩溃。

知道上述原因对问题的分析无实质性帮助，根据 `com.bumptech.glide.load.resource.bitmap.GlideBitmapDrawable` 信息首先可以确认是用到 Glide 加载图片相关的业务代码，堆栈信息中有 `AbsListView` 可以确认和其相关子类的控件有关，如 `ListView` 和 `GridView`。结合这两点能很快确认和某一块的业务代码相关。

### 2. 检索类似异常

既然和 Glide 相关，所以直接去 github 上 [glide](https://github.com/bumptech/glide) 的官方项目检索相关 [issue](https://github.com/bumptech/glide/issues?q=Canvas%3A+trying+to+use+a+recycled+bitmap+)，发现该问题的反馈还挺多，查看了几个类似问题后最终指向了官方关于常见错误一个说明文档 [common-errors](http://bumptech.github.io/glide/doc/resourcereuse.html#common-errors)

其中关于 `Cannot draw a recycled Bitmap` 官方的说明如下

> Glide 的 `BitmapPool` 是固定大小的。当 `Bitmap` 从中被踢出而没有被重用时，Glide 将会调用 `recycle()`。如果应用在向 Glide 指出可以安全地回收之后 “不经意间” 继续持有 `Bitmap`，则应用可能尝试绘制这个 `Bitmap`，进而在 `onDraw` 方法中造成崩溃。
> 一种可能的情况是，一个目标被用于两个 `ImageView`，而其中一个在 `Bitmap` 被放到 `BitmapPool` 中后仍然试图访问被回收后的 `Bitmap`。基于以下因素，要复现这种复用错误可能很困难：1）`Bitmap` 何时被放入池中，2）`Bitmap` 何时被回收，3）何种尺寸的 `BitmapPool` 和内存缓存会导致 `Bitmap` 的回收。可以在你的 `GlideModule`中加入下面的代码片段，以使这个问题更容易复现：

```java
@Override
public void applyOptions(Context context, GlideBuilder builder) {
    int bitmapPoolSizeBytes = 1024 * 1024 * 0; // 0mb
    int memoryCacheSizeBytes = 1024 * 1024 * 0; // 0mb
    builder.setMemoryCache(new LruResourceCache(memoryCacheSizeBytes));
    builder.setBitmapPool(new LruBitmapPool(bitmapPoolSizeBytes));
}
```

> 上面的代码确保没有内存缓存，且 `BitmapPool` 的尺寸为 0；因此 `Bitmap` 如果恰好没有被使用，它将立刻被回收。这是为了调试目的让它更快出现。

### 3. 尝试复现

结合上述说明我们可以尝试在项目代码里面对怀疑的地方进行相关日志添加再模拟场景尝试复现。实际的业务代码场景入手其实可以先定位大致位置，尝试触发大量图片加载触发 Glide `BitmapPool` 进行资源回收后来进行场景复现。这里我就简单通过 [Demo](https://github.com/shumxin/Drimo/tree/main/glideresreuseerror) 来复现该崩溃。

首先按照官网说明配置 `MemoryCache` 和 `BitmapPool` 大小为 0

```kotlin
@GlideModule
class MyAppGlideModule : AppGlideModule() {
    
    override fun applyOptions(context: Context, builder: GlideBuilder) {
        val bitmapPoolSizeBytes = 1024 * 1024 * 0 // 0mb
        val memoryCacheSizeBytes = 1024 * 1024 * 0 // 0mb
        builder.setMemoryCache(LruResourceCache(memoryCacheSizeBytes.toLong()))
        builder.setBitmapPool(LruBitmapPool(bitmapPoolSizeBytes.toLong()))
        builder.setLogLevel(Log.VERBOSE)
    }

    override fun isManifestParsingEnabled(): Boolean {
        return false
    }
}
```

需要注意的一点是:

> 如果你在 Kotlin 编写的类里使用 Glide 注解，你需要引入一个 kapt 依赖，以代替常规的 `annotationProcessor` 依赖。[见下载和设置](https://muyangmin.github.io/glide-docs-cn/doc/download-setup.html#kotlin)

[Demo](https://github.com/shumxin/Drimo/tree/main/glideresreuseerror) 尝试复现首次实现如下：

```kotlin
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    Glide.with(holder.imageView.context).load(dataSet[position]).into(holder.imageView)
}
```

发现一直无法复现相应崩溃场景，虽然在此之前已修复对应我司产品项目中的崩溃问题。看到这篇 [Blog](https://blog.csdn.net/Conan9715/article/details/117914506) 后才发现当时虽然解决了问题，但是深挖的还不够。

如该博主所说，当使用 `into(imageView)` 的方式加载图片，不会抛出异常。`into(imageView)` 的方式后面会`setResourceInternal(null)`，最终会调用到 `ImageView.setImageDrawable(null)`。这样在 `ImageView onDraw` 时判断 `mDrawable == null` 时直接返回了。

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    if (mDrawable == null) {
        return; // couldn't resolve the URI
    }

    ...
}
```

当使用 `into(Target)` 的方式则可能会导致抛出该异常，看了下我们的项目代码确实最终封装用到的是 `SimpleTarget`。

```kotlin
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    Glide.with(holder.imageView.context).load(dataSet[position])
        .into(object : CustomViewTarget<View, Drawable>(holder.imageView) {
            override fun onLoadFailed(errorDrawable: Drawable?) {

            }

            override fun onResourceReady(
                resource: Drawable,
                transition: Transition<in Drawable>?
            ) {
                holder.imageView.setImageDrawable(resource)
            }

            override fun onResourceCleared(placeholder: Drawable?) {

            }
        })
}
```

使用 `into(target)` 的方式复现了该崩溃

```java
io.github.shumxin.glideresreuseerror E/AndroidRuntime: FATAL EXCEPTION: main
    Process: io.github.shumxin.glideresreuseerror, PID: 23165
    java.lang.RuntimeException: Canvas: trying to use a recycled bitmap android.graphics.Bitmap@3b85e5e
        at android.graphics.BaseCanvas.throwIfCannotDraw(BaseCanvas.java:66)
        at android.graphics.RecordingCanvas.throwIfCannotDraw(RecordingCanvas.java:277)
        at android.graphics.BaseRecordingCanvas.drawBitmap(BaseRecordingCanvas.java:88)
        at android.graphics.drawable.BitmapDrawable.draw(BitmapDrawable.java:548)
        at android.widget.ImageView.onDraw(ImageView.java:1436)
        at android.view.View.draw(View.java:22350)
        at android.view.View.updateDisplayListIfDirty(View.java:21226)
        at android.view.View.draw(View.java:22081)
        ...
```

### 4. 原因分析

这里 `ImageView` 在 `onDraw` 阶段用到的 `mDrawable` 实际是 `BitmapDrawable`，其持有了对应的 `Bitmap` 对象，该 `Bitmap` 对象在 Glide 的 `LruBitmapPool#put` 方法中当不满足缓存条件时则会调用 `bitmap.recycle()` 进行回收。`RecyclerView` 滑动时当出现 `ViewHolder` 复用时，新的图片资源还未获取到时，该 `ViewHolder` 中的 `ImageView` 用之前请求的图片资源进行绘制时，对应该图片资源的 `mDrawable` 中的 `Bitmap` 已经被回收，遂会抛出该异常。

### 5. 解决方案

结合原因分析，最初我在工程项目里面的实现方案则是通过对崩溃的地方的自定义 ImageView，并对其 `onDraw()` 方法进行重写。判断如果当前持有的 `mDrawable` 是 `BitmapDrawable`，当其持有的 `bitmap.isRecycled`，则不触发最终的绘制操作。

```kotlin
override fun onDraw(canvas: Canvas?) {
    if (drawable is BitmapDrawable) {
        (drawable as BitmapDrawable).bitmap?.let {
            if (it.isRecycled) {
                return
            }
        }
    }
    super.onDraw(canvas)
}
```

上述方式可以解决该问题，但是其中的一个缺陷是需要知道哪些类型的 `drawable` 会持有 `bitmap`，比如 3.x 版本的 Glide 有 `GlideBitmapDrawable`，其内部也持有 `bitmap`，对于上述的处理就需要再增加一个分支处理。

那有没有更合适的方式来处理该问题呢，对于实现 CustomViewTarget 中 `onLoadFailed(@Nullable Drawable errorDrawable)、onResourceReady(@NonNull R resource, @Nullable Transition<? super R> transition)、onResourceCleared(@Nullable Drawable placeholder)` 三个方法时，其中 `onResourceCleared` 很容易被忽略。

关于 `onResourceCleared(@Nullable Drawable placeholder)` 的说明如下

> A required callback invoked when the resource is no longer valid and must be freed. You must ensure that any current Drawable received in onResourceReady(Object, Transition) is no longer used before redrawing the container (usually a View) or changing its visibility. Not doing so will result in crashes in your app.  

`onResourceCleared(@Nullable Drawable placeholder)` 是由 `onLoadCleared(@Nullable Drawable placeholder)` 调用

关于 `onLoadCleared(@Nullable Drawable placeholder)` 的说明如下

> A mandatory lifecycle callback that is called when a load is cancelled and its resources are freed.
You must ensure that any current Drawable received in onResourceReady(Object, Transition) is no longer used before redrawing the container (usually a View) or changing its visibility.  

再看下官方文档相关的说明 [链接](http://bumptech.github.io/glide/doc/resourcereuse.html#loading-a-resource-into-a-target-clearing-or-reusing-the-target-and-continuing-to-reference-the-resource)

> **往Target中加载资源，清除或重用Target，并继续引用该资源**
>
> 最简单的比较这个错误的办法是确保所有对资源的引用都在 `onLoadCleared()` 调用时置空。通常，加载一个 `Bitmap` 然后对 `Target` 解引用，并且不要再次调用 `into()` 或 `clear()`，这样是安全的。然而，加载了一个 `Bitmap`，清除这个 `Target`，并在之后继续持有 `Bitmap` 引用是不安全的。类似地，加载资源到一个 `View` 上然后从 `View` 中获取这个资源 (通过 `getImageDrawable()` 或任何其他手段)，并在其他某个地方继续引用它，也是不安全的。

当 Glide 回调 `onResourceCleared` 后即准备将相关的 `bitmap` 进行回收，所以我们只需要在 `onResourceCleared` 的时候主动 `setImageDrawable(null)` 不再持有接下来将被回收的 `Bitmap` 即可解决问题

```kotlin
override fun onResourceCleared(placeholder: Drawable?) {
    holder.imageView.setImageDrawable(null)
}
```

## 参考

- [1] [Glide Resource Reuse](http://bumptech.github.io/glide/doc/resourcereuse.html)
- [2] [Glide 导致的 RuntimeException: Canvas: trying to use a recycled bitmap android.graphics.Bitmap](https://blog.csdn.net/Conan9715/article/details/117914506)
