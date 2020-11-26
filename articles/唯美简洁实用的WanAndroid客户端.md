# 简洁唯美的 WanAndroid 客户端

## 1.什么是 WanAndroid 客户端

[WanAndroid](https://wanandroid.com/index) 是[鸿洋](https://github.com/hongyangAndroid)开发并维护的一个专门学习 **Android** 的站点，这里面你可以学习到各种关于 Android 知识。精彩的**每日一问**、你需要的**面试资料、面试题**、当然你也可以在这个上面**分享**知识博客，以及其他好的关于学习 Android 的内容。非常建议学习浏览！

同时鸿洋大佬还提供了 [WanAndroid 的 API](https://wanandroid.com/blog/show/2) 真造福了广大 Android 开发者啊！比心！👍

我就是根据 WanAndroid 提供的 API，写了一个客户端，这样可以在手机上可以继续学习了，真是太方便了！

## 2.WanAndroid 客户端特色以及所用的技术

- 整体项目 Kotlin 语言编写，以及 Kotlin Coroutine 协程的使用。
- 项目采用当前主流架构 MVVM。
- Android Jetpack 的使用包括但不限于`Lifecycle`、`LiveData`、`ViewModel`、`Databinding`、`Room`、`ConstraintLayout`等，未来可能会更多。
- 体验极好的 WanAndroid 客户端，**页面简洁直接但并不缺少美感！**
- 突出重点的模块设计，**每日一问**，我的收藏，**面试题**模块等等！
- 整体的设计、UI、图标、配色，都是根据经过**仔细揣摩精心设计**的，还是很精美的！
- 优秀的用户体验和交互设计
- 整洁的代码风格和标准的命名规范

### WanAndroidJetpack 架构图

![](https://raw.githubusercontent.com/jhbxyz/ArticleRecord/master/images/wanandroid-arch.jpg)

项目采用 `MVVM` 架构，用 `Kotlin` 语音编写，采用 `Retrofit` 和 `Kotlin-Coroutine` **协程**进行网络交互，加载图片 `Glide` 主流加载图片框架，数据存储主要用到了 `Room` 和腾讯的 `MMKV`。

Android Jetpack 是目前 Android 学习开发的趋势，所以我在项目用到了 `Lifecycle`、`LiveData`、`ViewModel`、`Databinding`、`Room`、`ViewPager2`、`ConstraintLayout`、`AndroidX`等 Jetpack 相关的最新技术

我相信这个一个非常不错的学习 MMVM + Kotlin + Jetpack 的项目了！具体细节请看 GitHub 的项目 [WanAndroidJetpack](https://github.com/jhbxyz/WanAndroidJetpack)

**喜欢的点个 [Stars](https://github.com/jhbxyz/WanAndroidJetpack)，有问题的请提 Issues**

## 3.WanAndroid 客户端长什么样子

**我录了个 GIF ，看一下具体内容吧！**

**友情提示：**

> Gif 还有下面的截图和真是 APP 的 UI 细节有出入，比如淡白色的分割线，背景色等等！ 
>
> 下载 APP 体验更佳，一起学起来吧！

![](https://raw.githubusercontent.com/jhbxyz/ArticleRecord/master/images/wan-gif.gif)



### APP 内的截图！

<img src="https://raw.githubusercontent.com/jhbxyz/ArticleRecord/master/images/w-1.jpg" width="350" />&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;<img src="https://raw.githubusercontent.com/jhbxyz/ArticleRecord/master/images/w-2.jpg" width="350" />

<img src="https://raw.githubusercontent.com/jhbxyz/ArticleRecord/master/images/w-3.jpg" width="350" />&nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;&nbsp;&nbsp; &nbsp;&nbsp;<img src="https://raw.githubusercontent.com/jhbxyz/ArticleRecord/master/images/w-4.jpg" width="350" />

<img src="https://raw.githubusercontent.com/jhbxyz/ArticleRecord/master/images/w-5.jpg" width="350" />



## 4.WanAndroid 客户端的功能介绍

整个 APP 主色调为天蓝色，在颜色选择、文字大小、图标方面我都花费了很多心思，整体的设计模块参考主流的 APP 格式，底部有五个 Tab 分别是：**首页、问答、收藏、发现、我的**！

下面我来分别介绍各个模块

#### 首页

* Banner 图
* 置顶文章
*  Feed 流

> 首页主要是由上面三大模块组成！
>
> 会展示大家最新分享的文章博客，每日一问等等
>
> 当然你也可以分享文章！

#### 问答

WanAndroid 相当有特殊的一个模块，非常干的干货，鸿洋会提出每日一问，而且问题都很有深度！由大家来回答，其中有一个优秀的回答者 [陈小缘](https://github.com/wuyr) 而且是一个自定义 View 的大佬！可以关注学习一波哈！

这个模块知识都非常有深度，所以我把它单独拿出来了，就是方便学习！

#### 收藏

收藏也是一个重要的模块主要由下面几部分组成

* 收藏文章：就是自己收藏的文章，可以通过点击列表 Item 上的小心心收藏，也可以在**我的**页面手动添加自己的喜欢的文章。
* 面试相关：对，没错就是为了方便直接看面试题，非常方便！
* 分享文章：这里是自己在 WanAndroid 站点分享的文章，可以是自己写的，也可以你觉得不错的文章！
* 收藏网站：收藏自己喜欢的网站，博客等等！

#### 发现

这个模块包含的内容非常多！基本你在这儿可以找到你想要的任何东西了

* 体系：关于 Android 学习相关的方方面面
* 导航：常用的开发者网站、优秀的博客、Flutter、三分平台、SDK 等等
* 公众号：优秀的 Android 开发者们，同时也包括大厂技术的微信号
* 项目：热门的项目，同时你也可以分享自己的项目
* 项目分类：按照项目的类型分类展示项目模块！

#### 我的

* 登录的入口，登录成功后会展示个人的信息以及积分排名等
* 收藏文章、网站，分享文章的编辑入口
* 设置：目前只是做了一个退出登录的操作

#### 整体总结

- 点击每个 Item 都会进入到具体的文章详情页面
- 详情页面是一个简洁的全屏的体验，同时可以收藏文章和滚动到顶部的一些操作
- 列表上有一键收藏功能（需要登录哟）
- 一键回滚到列表顶部操作

## 5.总结

* 这是一个 Kotlin+MVVM+Jetpack 的项目，如果想学习的可以学习一波儿，具体代码在  [GitHub](https://github.com/jhbxyz/WanAndroidJetpack) 上
* 喜欢的点个 Stars，有问题的请提 Issues
* 代码细节，请看项目 [WanAndroidJetpack](https://github.com/jhbxyz/WanAndroidJetpack)
* 使用的第三方库和日志更新在 GitHub 上
* 下载体验请 [点击下载 ](https://github.com/jhbxyz/WanAndroidJetpack/blob/master/app/apk/app-release.apk?raw=true)



