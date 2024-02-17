---
pubDatetime: 2016-08-24
title: Android WebView 相关姿势
featured: false
draft: false
tags:
  - Android
description: 在Android项目开发中，WebView的使用都是必不可少的。在一个基本功能完备的项目中，都会把WebView提取成一个公用的Activity组件，在需要跳转到H5页面的地方启动这个activity并传入相关参数即可。这样的简便快捷的使用，却容易造成开发者对WebView相关知识的疏忽。遂成此文，整理一二。
---

在Android项目开发中，WebView的使用都是必不可少的。在一个基本功能完备的项目中，都会把WebView提取成一个公用的Activity组件，在需要跳转到H5页面的地方启动这个activity并传入相关参数即可。这样的简便快捷的使用，却容易造成开发者对WebView相关知识的疏忽。遂成此文，整理一二。

<!-- more -->

## 添加WebView

在应用中使用WebView，只需要在layout文件中添加

```xml
<WebView
    android:id="@+id/main_web_view"
    android:layout_width="match_parent"
    android:layout_height="match_parent"/>
```

使用loadUrl方法来加载WebView

```java
WebView webView = (WebView) findViewById(R.id.main_web_view);
webView.loadUrl("http://www.example.com");
```

在此之前，还要确保应用有Internet权限

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

至此，一个基本的WebView页就完成了。

## 在WebView中使用JavaScript

默认情况下，WebView是禁用JavaScript的。通过如下代码来为WebView开启JavaScript支持

```java
WebSettings webSettings = webView.getSettings();
webSettings.setJavaScriptEnabled(true);
```

WebSettings管理着很多很有用的WebView设置项，例如setUserAgentString()等。具体细节可戳 [WebSettings Doc](http://androiddoc.qiniudn.com/reference/android/webkit/WebSettings.html).

### WebView与JavaScript的交互\*

WebView属Android端，所以该交互问题就是Android本地与JS端的交互问题。为了便于梳理，写了如图所示的一个Demo [Source code](https://github.com/ponywei/WebViewJSDemo)，上半部分白底为WebView，加载本地的一个html页面，内含一个button和一个h1；下半部分是Android上的一个Button。

![屏幕截图](http://7u2h4k.com1.z0.glb.clouddn.com/2016-03-31%2017.55.55.png)

### WebView调用JavaScript的方法

WebView调用JavaScript的方法相对比较简单，如果是无参数方法：

```java
webView.loadUrl("javascript:METHOD")
```

其中 METHOD 为javascript方法名；
如果方法需要入参，例如下面这个javascript方法：

```javascript
function testMethod(name) {
    ...
}
```

由于javascript为弱类型语言，所以入参并没有指定任何类型；具体类型的处理一般在javascript方法体内。
在webView中调用方法（注意，如果是字符串类型，必须在用单引号包裹，否则javascript无法识别，会提示name为定义）：

```java
String name = "Android";
webView.loadUrl("javascript:testMethod('" + name + "')");
```

### JavaScript调用Android端的Java方法

#### 1.创建接口

想要在 JavaScript 中调用相关的 Java 方法，需要在JavaScript端与Android客户端之间建立一个接口。

如下名为WebAppInterface的类，该类提供了一个名为showToast的本地方法，调用Android Toast通知消息，需要入参；该方法为等待JavaScript调用的方法，必须指定为 public 。

注意：如果 targetSdkVersion >= 17 ;必须要在为JS设置的方法加上 ** @JavascriptInterface ** 的注解；如果没有该注解，那么该方法在Android 4.2及以上的系统中是无法被JS访问到的。

```java
public class WebAppInterface {
    Context mContext;

    /** Instantiate the interface and set the context */
    WebAppInterface(Context c) {
        mContext = c;
    }

    /** Show a toast from the web page */
    @JavascriptInterface
    public void showToast(String toast) {
        Toast.makeText(mContext, "Hello," + toast, Toast.LENGTH_SHORT).show();
    }
}
```

#### 2.绑定接口

将该接口绑定到javascript，并指定一个名字，用于在javascript中调用。

```java
webView.addJavascriptInterface(new WebAppInterface(this), "Android");
```

这样就为运行在WebView中的JavaScript创建了一个名为Android的接口，

#### 3.调用

在javascript中的调用方法如下代码所示，id为btnCallNative的button在点击时后调用名为showAndroidToast的JS方法，并传入参数字符串；showAndroidToast方法体中通过名为Android的接口调用Android端的showToast方法，同时传入参数（注意： ** 没有必要 ** 再在javascript中对 "Android" 进行初始化，步骤2中webView已自动完成此工作）：

```html
<script type="text/javascript">
  function showAndroidToast(toast) {
    Android.showToast(toast);
  }
</script>

<button id="btnCallNative" onclick="showAndroidToast('javascript')">
  JS call Native
</button>
```

## WebView页面导航处理

当你在WebView中点击其他链接时，系统默认会去启动一个默认浏览器（或弹出选择）去处理这个URL请求。大多数情况下这都不符合需求场景，我们可以通过创建自定义的 ** WebViewClient ** 来修改这个规则，让用户的前进或者后退操作都在这个WebView内进行，或者打开其他程序来处理。

```java
private class MyWebViewClient extends WebViewClient {
    @Override
    public boolean shouldOverrideUrlLoading(WebView view, String url) {
        if (Uri.parse(url).getHost().equals("www.example.com")) {
            // This is my web site, so do not override; let my WebView load the page
            return false;
        }
        // Otherwise, the link is not for a page on my site, so launch another Activity that handles URLs
        Intent intent = new Intent(Intent.ACTION_VIEW, Uri.parse(url));
        startActivity(intent);
        return true;
    }
}
```

以上代码定义了一个名为MyWebViewClient的类，继承字WebViewClient，通过重写shouldOverrideUrlLoading方法来进行url路由操作；为了数据安全，建议在app的WebView中只加载该app自有的主机链接；对于其他不能确定安全性的链接在交给系统中其他程序处理。

然后为WebView初始化一个WebViewClient实例：

```java
webView.setWebViewClient(new MyWebViewClient());
```

## WebView历史导航

如果WebView重新加载了其他的URL，那么它会自动累加保存一个页面历史记录。可以通过 goBack() 和 goForward() 方法来进行历史导航。
如下代码，在activity中重写onKeyDown方法来处理返回按键的逻辑，其中 canGoBack() 用来判断是否还有历史页面可以返回，类似的还有 canGoForward()。如果用户按下返回键且存在可返回的历史页面，WebView回退到上一个历史页面。

```java
@Override
public boolean onKeyDown(int keyCode, KeyEvent event) {
    // Check if the key event was the Back button and if there's history
    if ((keyCode == KeyEvent.KEYCODE_BACK) && myWebView.canGoBack()) {
        myWebView.goBack();
        return true;
    }
    // If it wasn't the Back key or there's no web page history, bubble up to the default
    // system behavior (probably exit the activity)
    return super.onKeyDown(keyCode, event);
}
```

## 参考资料

- [Android WebView 与 JavaScript 交互](http://mthli.github.io/Android-WebView-JavaScript)
- [Building Web Apps in WebView](https://developer.android.com/intl/zh-cn/guide/webapps/webview.html)
