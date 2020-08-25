# Glide 基础使用

### 1.依赖（先依赖上）

```groovy
dependencies {
    //===========Glide===========
    implementation 'com.github.bumptech.glide:glide:4.9.0'
    annotationProcessor 'com.github.bumptech.glide:compiler:4.9.0'
    //===========Glide===========
}
```

### 2.权限（得有网络）

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

### 3.加载图片（三步走）

```kotlin
private fun loadImage(iv: ImageView, url: String) {
    Glide.with(iv.context).load(url).into(iv)
}
```

### 4.占位图（加载成功前，加载失败后）

```kotlin
private fun loadImageWithHoldeImage(iv: ImageView, url: String, placeHolder: Drawable, error: Drawable) {
        val options = RequestOptions()
            .placeholder(placeHolder)//加载成功前展示
            .error(error)//加载失败后展示

        Glide.with(iv.context)
            .load(url)
            .apply(options)
            .into(iv)
    }
```

### 5.指定图片大小（小心 OOM）

**Glide会自动根据ImageView的大小来决定图片的大小，以此保证图片不会占用过多的内存从而引发OOM**

```kotlin
private fun loadImageWithSize(iv: ImageView, url: String, width: Int, height: Int) {
    val options = RequestOptions()
        .override(width,height)//指定宽高  
  			// .override(Target.SIZE_ORIGINAL) 指定图片原始尺寸Glide就不会再去自动压缩图片，而是会去加载图片的原始尺寸,更高的OOM风险

    Glide.with(iv.context)
        .load(url)
        .apply(options)
        .into(iv)
}
```

### 6.缓存设置

##### 6.1内存缓存（防止应用重复将图片数据读取到内存当中）默认开启

禁用内存缓存（如果你有这种特殊需求的话）

```kotlin
private fun loadImageSkipMemoryChe(iv: ImageView, url: String) {
    val options = RequestOptions()
        .skipMemoryCache(true)//禁用内存缓存
    Glide.with(iv.context)
        .load(url)
        .apply(options)
        .into(iv)
}
```

##### 6.2.磁盘缓存（是防止应用重复从网络或其他地方重复下载和读取数据）

```kotlin
private fun loadImageWithCheStrategy(iv: ImageView, url: String, cacheStrategy: DiskCacheStrategy) {
    val options = RequestOptions()
        .diskCacheStrategy(cacheStrategy)//磁盘缓存的策略
 
    Glide.with(iv.context)
        .load(url)
        .apply(options)
        .into(iv)
}
```

- DiskCacheStrategy.NONE： 表示不缓存任何内容。
- DiskCacheStrategy.DATA： 表示只缓存原始图片。
- DiskCacheStrategy.RESOURCE： 表示只缓存转换过后的图片。
- DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片
- DiskCacheStrategy.AUTOMATIC： 表示让Glide根据图片资源智能地选择使用哪一种缓存策略（默认选项）

### 7.指定加载格式

**Glide内部会自动判断图片格式是图片还是 gif**

如果指定加载格式，不需要Glide自动帮我判断它到底是静图还是GIF图。

```kotlin
private fun loadImageOnlyImage(iv: ImageView, url: String) {
    Glide.with(iv.context)
        .asBitmap() //只允许加载静态图片
        .load(url)
        .into(iv)
}
```

如果是GIF图的话，Glide会展示这张GIF图的第一帧。

### 8.图片变换 （圆角图片、圆形图片）

```kotlin
    private fun loadImageWithTrans(iv: ImageView, url: String) {
        val options = RequestOptions()
            .transform(RoundedCorners(6))//圆角图片
            .circleCrop() //圆形图片
        Glide.with(iv.context)
            .load(url)
            .apply(options)
            .into(iv)
    }
```