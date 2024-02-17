---
pubDatetime: 2016-07-22
title: SpannableString.setSpan 非常规探索
featured: false
draft: false
tags:
  - Android
description: SpannableString.setSpan 非常规探索
---

SpannableString或SpannableStringBuilder可以通过SetSpan()方法对一个字符串对象的不同部分设置不同的样式信息，这个想必是人人皆知的。但看似简单的用法，其实里面也有一些小小的算不上坑的坑。

<!-- more -->

## SpannableString

SpannableString或SpannableStringBuilder可以通过SetSpan()方法对一个字符串对象的不同部分设置不同的样式信息，这个想必是人人皆知的，like this:

```java
String testString = "0123456789";
SpannableString spannableString = new SpannableString(testString);
spannableString.setSpan(new StyleSpan(Typeface.BOLD), 0, 4, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
spannableString.setSpan(new UnderlineSpan(), 4, spannableString.length(), Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
spanTv.setText(spannableString);
```

这段代码的运行效果就是“0123”加粗显示，“456789”下划线显示，酱紫:

![device-2016-07-22-182058-w193](http://7u2h4k.com1.z0.glb.clouddn.com/device-2016-07-22-182058-1.png)

看似简单的用法，其实里面也有一些小小的算不上坑的坑。

## setSpan方法

SpannableString实现了Spannable接口，该接口定义了两个方法，除了setSpan( )还有removeSpan( )。先看下setSpan( )这个方法：

**void setSpan (Object what, int start, int end, int flags)**

[官方解释](https://developer.android.com/reference/android/text/Spannable.html#setSpan(java.lang.Object, int, int, int))也比较简单，总结为一句话，就是对start~end之间的text附加特定的标记对象（markup object)，这个对象通常是[CharacterStyle](https://developer.android.com/reference/android/text/style/CharacterStyle.html)，[ParagraphStyle](https://developer.android.com/reference/android/text/style/ParagraphStyle.html)，[TextWatcher](https://developer.android.com/reference/android/text/TextWatcher.html)，[SpanWatcher](https://developer.android.com/reference/android/text/SpanWatcher.html)的子类。（PS：本例使用了StyleSpan和UnderlineSpan均为CharacterStyle的子类，还有各种各样的Span及用法可自行发掘，不在本文讨论范围之内）。官方文档中并没有对参数进行一一解释，可能是觉得太过显而易见？只是在方法说明中带了一嘴：去 Spanned查看各个flags的意义。
简单解释一下参数：

- Object what: 对应各种Span
- int start: 指定的Span开始应用的位置索引
- int end: 指定的Span结束应用的位置索引
- int flags: **现在只知道是一个标识**

刚开始使用SpannableString的时候，看到setSpan( )这个方法，会不由自主的理解为，这个里的flags是用来标识要设置Span的起始位置start...end的，很多博客在没有调查清楚的时候也是这么误导读者的，清一色的SPAN_INCLUSIVE_EXCLUSIVE用法。一个偶然的机会，我改了这个参数，发现对于显示效果没有任何的影响，一度以为自己出现了幻觉。

## 关于flags

通过查阅接口[Spanned](https://developer.android.com/reference/android/text/Spanned.html)文档，flags标识对于text处理常用的有以下几种常量：

- Spanned.SPAN_EXCLUSIVE_EXCLUSIVE：前开后开（do not expand to include text inserted at either their starting or ending point）
- Spanned.SPAN_EXCLUSIVE_INCLUSIVE：前开后闭（expand to include text inserted at their ending point but not at their starting point）
- Spanned.SPAN_INCLUSIVE_EXCLUSIVE：前闭后开（expand to include text inserted at their starting point but not at their ending point）
- Spanned.SPAN_INCLUSIVE_INCLUSIVE：前闭后闭（expand to include text inserted at either their starting or ending point）

贴上原文，方便感受一下这些flag的真是含义：对于在起点前或者终点后 **插入的（inserted at)** 文本 扩展/不扩展 应用Span。

恍然大悟，这个flags并不是标识我们在构造SpannableString或者SpannableStringBuilder时传入的String Object，而是针对在setSpan( )之后附加到 start 之前 或者 end 之后的文本，是否沿用Span。

## 场景一

什么情况下可以在setSpan( )之后可以在insert text呢？SpannableString对象和String一样，是不可变对象，那SpannableStringBuilder就和StringBuilder一样是可变的了。新发现一定要试试：

```java
String testString = "0123456789";
SpannableStringBuilder spannableStringBuilder = new SpannableStringBuilder(testString);
spannableStringBuilder.setSpan(new StyleSpan(Typeface.BOLD), 0, 4, Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
spannableStringBuilder.setSpan(new UnderlineSpan(), 4, spannableString.length(), Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
spannableStringBuilder.insert(0, "begin");
spannableStringBuilder.append("end");
spanTv.setText(spannableStringBuilder);
```

基本设置与前例一致，不同在于，在SpannableStringBuilder设置Span之后，分别在头部和尾部插入了两个字符串文本。由于[0,4)前闭后开，“begin”沿用加粗的Span；而[4,9)也是前闭后开，附加在最后的"end"则不沿用任何Span，酱紫：

![device-2016-07-22-181947-w268](http://7u2h4k.com1.z0.glb.clouddn.com/device-2016-07-22-181947.png)

## 场景二

再想，如果在代码中去手动insert、append特定文本到特定的位置，那直接在构造的时候就传进去岂不是更方便？似乎不是一个很通用的应用场景。那什么场景可以灵活的在文本的前后中间随意插入呢？ **EditText**文本输入框。
上栗子：

```java
String testString = "0123456789";
SpannableString spannableString = new SpannableString(testString);
spannableString.setSpan(new ForegroundColorSpan(ContextCompat.getColor(this, R.color.colorGreen)), 0, spannableString.length(), Spanned.SPAN_INCLUSIVE_EXCLUSIVE);
inputEt.setText(spannableString);
```

在“0”之前输入”begin“，在”9“之后输入“end”，酱紫：
![device-2016-07-22-163752-w335](http://7u2h4k.com1.z0.glb.clouddn.com/device-2016-07-22-163752.png)

## 后记

至此终于搞清楚了setSpan( )中flags参数的真实含义，过程略绕，因为SpannableString的官方文档对与setSpan( )并没有做任何说明，接口Spannable也只是对该方法的意义略做解释，而flags参数的真正含义却要挖到Spanned接口中对常量的解释。写此文的过程中，发现在SpannableStringBuilder的文档中对该方法有一句话说明：

> Mark the specified range of text with the specified object. The flags determine how the span will behave when text is inserted at the start or end of the span's range.

其对 flags 进行了解释，就是不知道为什么这句话没有出现在SpannableString的文档中。

日常使用没有 **insert** 需求时，flags只需传 **0** 即可。
