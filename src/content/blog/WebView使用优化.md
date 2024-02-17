---
pubDatetime: 2018-03-27
title: WebView 使用优化
featured: false
draft: false
tags:
  - Android
description: 为提升WebView在安居客APP中的加载速度，发起WebView优化项目，配合前端同学对HTML页面框架以及缓存方案的调整，进而整体提升HTML页面的用户体验。
---

为提升WebView在安居客APP中的加载速度，发起WebView优化项目，配合前端同学对HTML页面框架以及缓存方案的调整，进而整体提升HTML页面的用户体验。

<!-- more -->

# WebView使用优化

为提升WebView在安居客APP中的加载速度，发起WebView优化项目，配合前端同学对HTML页面框架以及缓存方案的调整，进而整体提升HTML页面的用户体验。

## WebView使用现状分析

关于WebView初始化过程的详细分析可参见[WebView性能、体验分析与优化](https://tech.meituan.com/WebViewPerf.html)。

> 当App首次打开时，默认是并不初始化浏览器内核的；只有当创建WebView实例的时候，才会创建WebView的基础框架。所以与浏览器不同，在App中打开WebView的第一步并不是建立连接，而是启动浏览器内核。

安居客目前加载H5页面的是ShareWebViewActivity，其中包含一个WebView控件，每次打开一个H5页面（H5页面内部链接除外），都会new一个全新的Activity，进行一系列初始化工作。这其中最大的耗时元凶就是WebView控件的初始化。

为了获得WebView页面加载时间的各项数值，在ShareWebViewActivity的生命周期方法onCreate中打log，在WebViewClient的onPageStarted和onPageFinished方法中打log，我们定义“从new WebView(...)到onPageStarted(...)之间的时间“做为**WebView的初始化时间**，也就是**从WebView开始构建，到WebView发起url请求之间的时间**。

WebView的初始化时间在WebView首次初始化和非首次初始化时并不相同。我们定义以下几个时间指标：

- 首次初始化时间：客户端冷启动后，第一次打开WebView的初始化时间；
- 二次初始化时间：在打开过WebView后，退出WebView，再重新打开WebView的初始化时间。
- 页面load时间：WebViewClient的onPageStarted和onPageFinished之间的耗时。

测试环境（后续测试如无特殊说明皆为该测试环境下的测试结果）
测试手机：三星Galaxy S6 Edge+ SM-G9280
测试系统：Android 7.0
测试APP：安居客APP
测试方式：APP首页“安居头条”H5页，三次平均值
测试结果：（单位：毫秒）

| 首次初始化时间 | 二次初始化时间 |
| :------------: | :------------: |
|     248.5      |     90.33      |

###

根据以上测试结果，在启动APP之后首次打开H5页时，有近250毫秒的时间时用来做WebView初始化的，这段时间里既没有发起请求，也没有做任何解析。

由于在后续的其它方案中，WebView的初始化工作并不会同页面的初始化工作一同进行，所以这里在制定统一的测试时间指标时，我们把初始化时间定义为整个页面的初始化时间，也就是从ShareWebViewActivity的onCreate方法开始，到WebViewClient的onPageStarted(...)方法之间的耗时。各项指标定义如下：

- 页面初始化时间：从页面开始创建起，到WebView发起URL请求的onPageStarted方法止；
- 页面load时间：从WebView发起URL请求起，到WebView URL page请求结束止；
- 总耗时：从页面开始创建起，到WebView URL page请求结束止。

同时为了排除对象内存缓存和WebView本地缓存对测试结果的影响，设置了三个测试用例：

- 冷启动进入页面：首次冷启动APP，在首页点击“安居头条”，跳转安居头条H5页面；
- 再次进入页面：返回首页，点击“安居头条”，跳转安居头条H5页面；
- 清除缓存重启APP进入页面：在设置的测试页面清除WebView缓存，杀掉APP进程重新启动，在首页点击“安居头条”，跳转安居头条H5页面。

针对以上指标和测试用例，对当前项目中的H5页面加载方式进行测试，结果如下表：

|                         | 页面初始化时间 | 页面load时间 | 总耗时 |
| ----------------------- | -------------- | ------------ | ------ |
| 冷启动进入页面          | 315            | 1174         | 1489   |
| 再次进入页面            | 130            | 737          | 867    |
| 清除缓存重启APP进入页面 | 309            | 1120         | 1429   |

## 方案一：全局WebView

WebView的初始化完全发生在APP端，首次初始化占用大量的时间，考虑在APP启动时就初始化一个WebView实例，在ShareWebViewActivity中直接获取这个WebView add到父布局中使用。
需要注意的是，不管用户是否打开H5页，内存中都有有一个WebView的实例，在一定程度上会增大APP内存的使用；且全局的WebView需要注意在退出时对浏览记录的清理工作，从而避免对下次使用时的goBack操作产生干扰。

实现：

1. 在AnjukeApp.java中新增全局WebView变量，在APP init过程中初始化该变量备用；
2. 新增isWebViewClearHistory标识，在ShareWebViewActivity finish时置true，在WebViewClient的onPageFinished方法中读取该字段，如果为true就对WebView.clearHistory()并重置该标识为false。（注意：webView#clearHistory()方法仅仅是清空当前页面之前的浏览历史 list。）

测试结果：（单位：毫秒）

|                         | 页面初始化时间 | 页面load时间 | 总耗时 |
| ----------------------- | -------------- | ------------ | ------ |
| 冷启动进入页面          | 186            | 1555         | 1741   |
| 再次进入页面            | 76             | 891          | 967    |
| 清除缓存重启APP进入页面 | 131            | 1058         | 1189   |

## 方案二：全局WebView + preload

根据方案一的测试结果，将 WebView 全局提前初始化之后，省去了首次的初始化事件，明显提升了第一次页面请求的速度。
为什么要做 preload？

1. DNS 缓存：DNS会在系统级别进行缓存，对于WebView的地址，如果使用的域名与native的API相同，则可以直接使用缓存的DNS而不用再发起请求。安居客的 API 请求域名为 api.anjuke.com，而 H5页面的域名主要为m.anjuke.com；因此 WebView 的首次 loadUrl 并不会有 H5的 DNS 缓存可以使用，仍然需要获取m.anjuke.com的 IP 地址。
2. JS 缓存：由前端同学提供一个空白的公共资源页面，此页面不显示任何实际内容，但包含一些前端站通用的资源（CSS，JS？）等，加载一次即可实现通用资源的缓存，后续使用无需再进行资源的请求。

实现：

1. 全局 WebView 实现同方案一，在 WebView 初始化完成之后调用一次 loadUrl 加载公共资源地址；
2. 对页面历史的处理，在方案一的基础上，新增对goBack()方法的控制，因为默认的公共资源页会在 url list 的最底部。

测试结果：（单位：毫秒）

|                         | 页面初始化时间 | 页面load时间 | 总耗时 |
| ----------------------- | -------------- | ------------ | ------ |
| 冷启动进入页面          | 113            | 1256         | 1369   |
| 再次进入页面            | 100            | 1149         | 1249   |
| 清除缓存重启APP进入页面 | 128            | 1231         | 1359   |

## 方案三： H5页独立进程 + WebView 全局 + preload

独立进程的H5页面的方案并不会对 WebView 的加载网页的速度上有任何显式的提升，但是对整个 APP 的性能上会有很多的益处，所以在 WebView 优化的同时把独立进程方案一并实现了。

为什么 WebView 的独立进程很有必要？

- WebView 加载网页的时候可能占用大量内存，APP 被系统回收的概率增高；独立进程系统会分配给独立的内存空间，增大了 APP 主进程的可用内存，减小 OOM 风险；
- WebView 由于自身或者系统问题等难以定位解决的 bug 导致 crash；独立进程发生崩溃不影响主进程的持续运行，降低 APP 的 crash 率。

如何开启独立进程？

- 在AndroidManifest.xml中声明组件（activity、service、receiver、provider）时，标注process属性为其它值`android:process=":h5"`，则该进程的完成名称为“applicationId:h5“；
- 启动组件时，如果进程不存在，系统会为该组件创建进程。

独立进程有独立的内存空间，不同进程之间无法直接共享数据，所以独立进程开发需要注意以下几点：

1. Application 的 onCreate 方法会多次调用；
2. 主进程的单例模式合和全局变量失效；
3. H5进程与主进程的通信。

经过分析，在安居客当前项目中，H5的展示页ShareWebViewActivity如果使用单独的H5进程，最大的问题就是对当前登录用户状态数据的获取。因为用户登录、注销操作皆发生在主进程，所以存在主进程对该数据的更改。这里要使用到跨进程通信的Binder机制，来完成在h5进程与主进程的通信，取得最新的登录用户数据。关于Binder跨进程通信机制原理的详细分析可参考[图文详解 Android Binder跨进程通信的原理](https://www.jianshu.com/p/4ee3fd07da14)。

实现：

- 在AndroidManifest.xml中标记ShareWebViewActivity的进程为“:h5”进程；
- 新建WebViewInstance单例，负责全局 WebView 的管理（具体类实现同方案二）；
- 新建WebViewRemoteService到:h5进程，在onCreate方案中进行全局WebView的初始化工作；
- 在App的onCreate方法中startService（多进程模式下，Application会多次创建，这里注意区分仅主进程时start即可）；
- 在ShareWebViewActivity中通过WebViewInstance获取并使用全局WebView（同方案二）；
- 新建IWebViewAidlInterface.aidl接口文件，声明方法`getLoginedUser()`（目前主要提供用户登录状态info的获取），编译生成IWebViewAidlInterface.java类；
- 新建WebViewService到主进程，在ShareWebViewActivity onCreate时bind此Service，返回实现IWebViewAidlInterface接口方法的Binder，充当h5进程与主进程通信的桥梁；
- 在ShareWebViewActivity onCreate时绑定WebViewService，获取返回Binder，转换成IWebViewAidlInterface实例，进而调用getLoginedUser()方法，实现跨进程通信。

整体实现如图所示：
![BinderFlow](http://7u2h4k.com1.z0.glb.clouddn.com/Binder.001.jpeg)

异常情况分析：

- APP被杀死，主进程和h5进程皆被kill，再次打开APP在onCreate中开启 WebViewRemoteService，进而创建h5进程；
- h5进程在不可见状态（无H5页面显示）时被系统kill，通过在 LogCat 手动点击 kill process 来模拟，h5进程会自动重新创建，WebViewRemoteService也会自动重启；
- ShareWebViewActivity页面出现异常导致crash，系统会弹出应用程序已停止弹窗；如果点击重新启动应用，h5进程会重新创建，WebViewRemoteService也会自动重启；如果取消，则h5进程dead，再次打开ShareWebViewActivity页面时，在显示loading之前会有一定时间的白屏，为新h5进程的创建和初始化时间，稍后页面即可正常显示。

时间测试结果（单位：毫秒）：

|                         | 页面初始化时间 | 页面load时间 | 总耗时 |
| ----------------------- | -------------- | ------------ | ------ |
| 冷启动进入页面          | 163            | 1587         | 1750   |
| 再次进入页面            | 106            | 717          | 823    |
| 清除缓存重启APP进入页面 | 131            | 1007         | 1138   |

由于在WebView独立进程方案中，h5进程会长期存有一个WebView，目前并没有对他设置失效回收机制，这也就是引起了对内存使用上的一些顾虑。从内存占用的角度对独立进程方案进行分析，设置了这样的几个测试用例，分别对独立进程方案下主进程和h5进程分别的内存占用，以及原始方式主进程的内存消耗进行记录：

1. 打开app，展示首页；
2. 点击首页“安居头条”，进入安居头条H5页面；
3. 返回首页；
4. 再次点击首页“安居头条”，进入安居头条H5页面；
5. 返回首页；
6. 在首页不进行任何操作，静默五分钟。

内存占用测试结果（单位：MB）：

|          | 进程名 | 首页 | 首次打开H5 | 返回首页 | 二次打开H5 | 返回首页 | 首页静默 |
| -------- | ------ | ---- | ---------- | -------- | ---------- | -------- | -------- |
| 独立进程 | App    | 98   | 45.6       | 96.5     | 53.3       | 99.7     | 99.2     |
|          | H5     | 20   | 107.6      | 54.9     | 104.7      | 56.7     | 55.7     |
|          | TOTAL  | 118  | 153.2      | 151.4    | 158        | 156.4    | 154.9    |
| 原始方式 | App    | 146  | 153        | 153      | 159.8      | 155      | 142.4    |

## WebView各种缓存机制

根据HTML标准，到目前为止，H5 一共有6种缓存机制，有些是之前已有，有些是 H5 才新加入的。

1. 浏览器缓存机制
2. Dom Storgage（Web Storage）存储机制
3. Web SQL Database 存储机制
4. Application Cache（AppCache）机制
5. Indexed Database （IndexedDB）
6. File System API

除了第6点是H5新加入的特性，WebView还不支持之外，其它五种缓存机制在WebView中均可以通过打开开关或者进行简单配置即可开启。下面对这几种缓存机制进行简单的解释以及当前的使用情况进行分析。

### 浏览器缓存

浏览器缓存机制是指通过 HTTP 协议头里的 Cache-Control（或 Expires）和 Last-Modified（或 Etag）等字段来控制文件缓存的机制。

Cache-Control 用于控制文件在本地缓存有效时长。最常见的，比如服务器回包：Cache-Control:max-age=600 表示文件在本地应该缓存，且有效时长是600秒（从发出请求算起）。在接下来600秒内，如果有请求这个资源，浏览器不会发出 HTTP 请求，而是直接使用本地缓存的文件。

Last-Modified 是标识文件在服务器上的最新更新时间。下次请求时，如果文件缓存过期，浏览器通过 If-Modified-Since 字段带上这个时间，发送给服务器，由服务器比较时间戳来判断文件是否有修改。如果没有修改，服务器返回304告诉浏览器继续使用缓存；如果有修改，则返回200，同时返回最新的文件。

Cache-Control 通常与 Last-Modified 一起使用。一个用于控制缓存有效时间，一个在缓存失效后，向服务查询是否有更新。

Expires 是 HTTP1.0 标准中的字段，Cache-Control 是 HTTP1.1 标准中新加的字段，功能一样，都是控制缓存的有效时间。当这两个字段同时出现时，Cache-Control 是高优化级的。

Etag 也是和 Last-Modified 一样，对文件进行标识的字段。不同的是，Etag 的取值是一个对文件进行标识的特征字串。在向服务器查询文件是否有更新时，浏览器通过 If-None-Match 字段把特征字串发送给服务器，由服务器和文件最新特征字串进行匹配，来判断文件是否有更新。没有更新回包304，有更新回包200。Etag 和 Last-Modified 可根据需求使用一个或两个同时使用。两个同时使用时，只要满足基中一个条件，就认为文件没有更新。

只要是个主流的、合格的浏览器，都应该能够支持HTTP协议层面的这几个字段。这不是我们开发者可以修改的，也不是我们应该修改的配置。在Android上，我们的WebView也支持这几个字段。但是我们可以通过代码去设置WebView的Cache Mode，而使得协议生效或者无效。WebView有下面几个Cache Mode：

- LOAD_CACHE_ONLY: 不使用网络，只读取本地缓存数据；
- LOAD_DEFAULT: 根据cache-control决定是否从网络上取数据；
- LOAD_CACHE_NORMAL: API level 17中已经废弃，从API level 11开始作用同LOAD_DEFAULT模式；
- LOAD_NO_CACHE: 不使用缓存，只从网络获取数据；
- LOAD_CACHE_ELSE_NETWORK，只要本地有，无论是否过期，或者no-cache，都使用缓存中的数据。本地没有缓存时才从网络上获取。

在Android的WebView中修改缓存的mode，要进行如下设置：

```java
WebSettings webSettings = webView.getSettings();
settings.setCacheMode(WebSettings.LOAD_CACHE_ONLY)
```

根据源码中对setCacheMode的注释，缓存的使用方式基于页面的导航类型：如果是常规的load页面，会检查缓存，然后按需重新验证并加载内容；如果是后退操作，页面内容不会被重新验证，WebView会直接从缓存中获取。目前安居客项中并没有显式的设置WebView的cacheMode，不过这种缓存模式是由浏览器默认的提供的，在未显式设置的情况下WebView的缓存模式是LOAD_DEFAULT。

在网络正常时，采用默认缓存策略，在缓存可获取并且没有过期的情况下加载缓存，否则通过网络获取资源以减少页面的网络请求次数。在开启缓存的前提下，WebView在加载页面时检测网络变化，倘若在加载页面时用户的网络突然断掉，我们更改WebView的缓存策略，这就是**离线缓存策略**。

```java
ConnectivityManager connectivityManager = (ConnectivityManager)getSystemService(Context.CONNECTIVITY_SERVICE);
NetworkInfo networkInfo = connectivityManager.getActiveNetworkInfo();
if(networkInfo.isAvailable()) {
    settings.setCacheMode(WebSettings.LOAD_DEFAULT);//网络正常时使用默认缓存策略
} else {
    settings.setCacheMode(WebSettings.LOAD_CACHE_ONLY);//网络不可用时只使用缓存
}
```

### Application Cache

Application Cache（简称 AppCache)似乎是为支持 Web App 离线使用而开发的缓存机制。它的缓存机制类似于浏览器的缓存（Cache-Control 和 Last-Modified）机制，都是以文件为单位进行缓存，且文件有一定更新机制。但 AppCache 是对浏览器缓存机制的补充，不是替代。

AppCache 原理有两个关键点：manifest 属性和 manifest 文件。当一个设置了manifest文件的html页面被加载时，CACHE MANIFEST指定的文件就会被缓存到浏览器的App Cache目录下面。当下次加载这个页面时，会首先应用通过manifest已经缓存过的文件，然后发起一个加载xxx.appcache文件的请求到服务器，如果xxx.appcache文件没有被修改过，那么服务器会返回304 Not Modified给到浏览器，如果xxx.appcache文件被修改过，那么服务器会返回200 OK，并返回新的xxx.appcache文件的内容给浏览器，浏览器收到之后，再把新的xxx.appcache文件中指定的内容加载过来进行缓存。

在Android 内嵌 Webview中，需要通过 Webview 设置接口启用 AppCache，同时还要设置缓存文件的存储路径，另外还可以设置缓存的空间大小。

```java
WebSettings webSettings = webView.getSettings();
webSettings.setAppCacheEnabled(true);
String cachePath = getApplicationContext().getCacheDir().getPath(); // 把内部私有缓存目录'/data/data/包名/cache/'作为WebView的AppCache的存储路径
webSettings.setAppCachePath(cachePath);
webSettings.setAppCacheMaxSize(5 * 1024 * 1024);
```

不过根据官方文档，AppCache 已经不推荐使用了，HTML标准后续也不会再支持。这里仅做介绍，并不会在项目中引入该缓存机制。

### DOM Storage

Dom Storage 给 Web 提供了一种更录活的数据存储方式，存储空间更大（相对 Cookies)，用法也比较简单，方便存储服务器或本地的一些简单的临时数据，类似于 Android 的 SharedPreference 机制。在 Android 内嵌 Webview 中，需要通过 Webview 设置接口启用 Dom Storage，目前安居客项目中已启用。

```java
WebSettings webSettings = myWebView.getSettings();
webSettings.setDomStorageEnabled(true);
```

### Indexed Database

IndexedDB 是一种灵活且功能强大的数据存储机制，它集合了 Dom Storage 和 Web SQL Database 的优点，用于存储大块或复杂结构的数据，提供更大的存储空间，使用起来也比较简单。可以作为 Web SQL Database 的替代，不太适合静态文件的缓存。Android 在4.4开始加入对 IndexedDB 的支持，只需打开允许 JS 执行的开关就好了，目前安居客项目中已启用。

```java
WebSettings webSettings = myWebView.getSettings();
webSettings.setJavaScriptEnabled(true);
```

## 参考

- [WebView性能、体验分析与优化](https://tech.meituan.com/WebViewPerf.html)
- [H5 缓存机制浅析 移动端 Web 加载性能优化](https://segmentfault.com/a/1190000004132566#articleHeader3)
