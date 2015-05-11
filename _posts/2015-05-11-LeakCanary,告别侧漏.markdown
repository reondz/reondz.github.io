---
layout: post  
title: LeakCanary，~~告别侧漏~~  
subtitle:  
category: 译文   
date: 2015-05-11 14:05:00  
author: Reon  
header-img: "img/post-bg-01.jpg"
---

文章翻译自[戳我](https://corner.squareup.com/2015/05/leak-canary.html)，项目github地址在[戳我](https://github.com/square/leakcanary)

###没人喜欢漏
在Square的注册界面，我们[将用户的签名画在了一个bitmap上](https://corner.squareup.com/2010/07/smooth-signatures.html)。而这张bitmap的大小和屏幕尺寸是一样的，接下来的事情我想你们也应该可以预测到了，在创建这个界面的时候，大量的OOM随之而来。  
<!--more-->
![image](http://7xiegl.com1.z0.glb.clouddn.com/square_signature.png)  

我们尝试过一些手段，但是都不能解决这个问题:  

    1. 使用Bitmao.Config.ALPHA_8（因为签名是黑白图）  
    2. catch住OOM的Error，然后触发GC再进行重试（从[GCUtils](https://android.googlesource.com/platform/packages/inputmethods/LatinIME/+/ics-mr1/java/src/com/android/inputmethod/latin/Utils.java)里得到的灵感）  
    3. 我们尚未思考过脱离Java Heap去申请内存。还好当时[Fresco](https://github.com/facebook/fresco)还不存在（这里他们的意思应该是正是因为没有Fresco所以他们才有灵感和动力做出LeakCanary）  

###我们曾经误入歧途
图片的大小并不是根本问题所在。当内存几乎被吃满的时候，OOM随时随地都会发生。而OOM一般更多的发生字在创建大对象的时候，例如bitmap。实际上OOM是另外一个更深层次问题的症状之一，这个问题便是:```内存泄漏```。   

###内存泄漏是什么？
有一些对象具有有限的生命周期。当它们的任务完成后，它们理应被GC回收掉。但是如果在某个对象的生命周期结束之后有一个引用链持有了该对象，就发生了内存泄漏。当这些泄露不断累积，应用最终必然会耗尽内存而。    

###~~拒绝侧漏~~（捕获内存泄漏）
捕获内存泄漏是有方法论的，在Raizlabs的[扯一扯Dalvik](http://www.raizlabs.com/dev/2014/03/wrangling-dalvik-memory-management-in-android-part-1-of-2/)系列文章中有很好的介绍（粗略翻看了了一下，感觉挺不错的，周末前会翻译学习一下）。  
下面是几个关键步骤:  

    1. 通过一系列手段获取OOM导致的崩溃日志（原文推荐了[Bugsnag](https://bugsnag.com/), [Crashlytics](https://try.crashlytics.com/), or the Developer Console.）  
    2. 尝试复现问题。你可能需要借买，借，偷(原文就是steal＝。＝挺调皮嘛)到那台该死的会出问题的设备（并不是所有设备都会导致内存泄漏，读者加：有这个说法，和版本和rom有关系之类的吗？）。你需要找出可以导致问题的操作路径。  
    3. Dump出内存Heap当OOM发生的时候，方法在[这里](https://gist.github.com/pyricau/4726389fd64f3b7c6f32)。  
    4. 通过[MAT](http://eclipse.org/mat/)和[YourKit](https://www.yourkit.com/)找出那个本应该被垃圾回收的对象。  
    5. 计算出该对象到GC roots的最短的强引用路径。
    6. 找出该路径中本应该不存在的引用（也就是找出那个对象hold住了这个应用链），然后修复内存泄漏。  
    
好了，这个步骤虽然不多，但是做多了也挺麻烦的吧？如果有一个库，可以在OOM前帮你做了这些事，从而让你专注于修复内存泄漏呢？  

###LeakCanary
[LeakCanary](https://github.com/square/leakcanary)是一个开源的Java库，可以帮助你在你的debug包上检测内存泄漏。  

让我们一起看一下下面的例子:  

	class Cat {
	}
	class Box {
  		Cat hiddenCat;
	}
	class Docker {
  		static Box container;
	}

	// ...

	Box box = new Box();
	Cat schrodingerCat = new Cat();
	box.hiddenCat = schrodingerCat;
	Docker.container = box;
	
我们创建了一个RefWatcher的实例，并让它去监视（watch）一个对象。  

	// We expect schrodingerCat to be gone soon (or not), let's watch it.
	refWatcher.watch(schrodingerCat);
	
当泄漏发生并被检测到的时候，会有一个泄漏的trace自动的显示出来。  

	* GC ROOT static Docker.container
	* references Box.hiddenCat
	* leaks Cat instance  
	
我们都知道程序员忙于写各种features，所以我们让LeakCanary的使用变的更加简单。只需要一行代码，LeakCanary就会自动的开始检查Activity的泄漏。  

	public class ExampleApplication extends Application {
  		@Override public void onCreate() {
    		super.onCreate();
    		LeakCanary.install(this);
  		}
	}
	
（这里笔者添加一下完整的攻略）
要使用LeakCanary，只需要在你的gradle项目的配置文件中即build.gradle中添加如下dependencies  

 	dependencies {
   		debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
   		releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
 	}

你会收到一条通知，并且有一个界面非常友好的展示显示出来。  

![image](http://7xiegl.com1.z0.glb.clouddn.com/leaktrace.png)  


###结论
在我们开启了LeakCanary后，我们发现并修复了我们的app中大量的内存泄漏问题。我们甚至还发现了[Android SDK自身的一些内存泄漏问题](https://github.com/square/leakcanary/blob/master/library/leakcanary-android/src/main/java/com/squareup/leakcanary/AndroidExcludedRefs.java)（＝ ＝这个碉）。  

最终的结果是非常惊人的。我们减少了```94％```的OOMCrash！  

![image](http://7xiegl.com1.z0.glb.clouddn.com/oom_rate.png)  

好了，接下来就是广告时间。  
如果你希望限制一下OOM crash，请立即使用[LeakCanary](https://github.com/square/leakcanary)。我们的口号是，告别侧漏，舒爽一整夜，LeakCanary你值得拥有。

