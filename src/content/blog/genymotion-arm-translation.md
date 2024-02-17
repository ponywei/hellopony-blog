---
pubDatetime: 2016-04-01
title: Genymotion 模拟器错误的解决
featured: false
draft: false
tags:
  - Android
description: Genymotion模拟器错误的解决
---

Android模拟器界Genymotion久负盛名，由于以前开发过程中都是真机调试，装的Genymotion也就是体验一下，并没有真正用来开发。这两天Mac的USB接口告急，决定开发调试转Genymotion来进行。但是就在安装的时候就遇到了错误"[INSTALL\_FAILED\_NO\_MATCHING\_ABIS]"导致app无法install，一番折腾之后总算是通过了，总结一下。

<!-- more -->

## 起因

### 错误代码：[INSTALL_FAILED_NO_MATCHING_ABIS]

一番搜索之后，定位到了错误原因：项目中"armeabi/"和"armeabi-v7a/"中引用到了一些.so库，但是Genymotion的System Image是基于Intel x86架构，往一个x86架构的系统中安装含有基于arm架构本地库的APP是注定要失败的。网上已经有了一些很不错的讨论：[StackOverflow](http://stackoverflow.com/questions/7080525/why-use-armeabi-v7a-code-over-armeabi-code)

## 解决方案

在模拟器系统中安装Translation库。

### 4.x

模拟器系统为4.x的解决方案可以参考这篇博客：[4.x方案](http://blog.csdn.net/wjr2012/article/details/16359113)
PS：此博客更新中说可以解决在5.1上的问题，但结合评论的反馈，以及我的实测，此方案在5.x（5.0，5.1）上无效。

### 5.x

以上方案在5.x上并没有效果，所以又一番墙外搜索，找到了这篇博客：[Use ARM Translation on 5.x image](http://23pin.logdown.com/posts/294446-genymotion-use-arm-translation-on-5x-image)，Problem Solved！看作者的文风，应该是港澳台同胞（膜拜大神ing），可以直接去看原文，或者直接看下面的解决步骤：

0. Genymotion中新建5.0或5.1的模拟器
1. 把[ARM_Translation_Lollipop.zip](https://mega.nz/#!Iw1HRLxR!8zVOQ84uk2hpxgsRrsHfp-wsbKOUvupHLJyWqWzPiUg)拖进模拟器窗口，OK，自动安装
2. 安装完成先不要着急关闭/重启模拟器（不同于4.x），执行命令：` adb shell /system/etc/houdini_patcher.sh`
3. 完成后重启模拟器

### 后记

以上两种方案应该能解决绝大部分人的问题了，再次感谢两位博客原作者。顺带安利一下，Genymotion确实是“目前”东西半球加起来最顺滑的模拟器啊没有之一。不过 Android Studio 2.0 已经发布到Preview 4了，最最最重要的是，全新的模拟器很逆天，据说Google自己说很好用，速度提升50倍!!!详细信息看这里：[Android Studio 2.0 Preview](http://android-developers.blogspot.jp/2015/11/android-studio-20-preview.html)。不过因为是工作环境，还没敢升级到Preview去尝试，十分期待稳定版!
