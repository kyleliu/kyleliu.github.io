---
layout: post
title: "以Fpanel为例说明如何为Webdroid系统扩充功能"
categories: 开发
tags: webdroid HAL
---

以Fpanel为例说明如何为Webdroid系统扩充功能
=====================================

* ### 写在前面的话

	最近项目中刚好有添加一个前面板功能的要求，就是要求Webapp应用能够控制前面板地显示，别看这是一个小功能需求，但麻雀虽小五脏俱全，这个功能地实现从最底层地Kernel驱动到最顶层地应用接口都有涉及，从这个实现中刚好可以了解一个系统级别的功能是如何添加进Webdroid的，借此机会跟大家分享一下。相信了解这个过程以后，不仅可以将这个功能完整地Merge到其它系统，而且写一个新地功能也再是一个困难的事情。
	
	为了继续后面的描述，我假定我们的代码根目录为$(WEBDROID)。

* ### HAL层接口定义

	HAL是硬件抽象层（Hardware Abstract Layer）的简称，最先是Android系统为了规避GPL的版权问题，在应用层定义的一套硬件访问接口规范，使得很多硬件厂商可以在此基础上开发驱动而不用开放源代码。这一层实际上是一个接口定义规则，它并不包括具体有哪些接口，这就为扩展系统功能提供了无尽可能。
	
	基本上每一个设备对应一个头文件，同时一般来说都会提供一个默认的实现。以fpanel设备为例，其接口头文件存放在$(WEBDROID)/hardware/libhardware/include/hardware下，名称为fpanle.h，同时我们也为它准备了一个默认实现，代码见$(WEBDROID)/hardware/libhardware/modules/fpanel目录，这个默认实现只是加了一些打印，实际不做任何事情。
	
	涉及的文件：
	
	> $(WEBDROID)/hardware/libhardware/include/hardware/fpanel.h
	> $(WEBDROID)/hardware/libhardware/modules/fpanel/fpanel.c
	> $(WEBDROID)/hardware/libhardware/modules/fpanel/Android.mk

* ### HAL层接口的实现

	有了接口和默认实现还不够，每一个平台都要去实现这套接口，以海思系统为例，这个实现涉及到的文件有：
	
	> $(WEBDROID)/hardware/hisilicon/godbox/fpanel/fpanel_godbox.c
	> $(WEBDROID)/hardware/hisilicon/godbox/fpanel/fpanel_test.c
	> $(WEBDROID)/hardware/hisilicon/godbox/fpanel/Android.mk
	> $(WEBDROID)/hardware/hisilicon/godbox/Android.mk
	
	其中特别要注意最后一个Android.mk，因为fpanel为新添加的目录，它不是默认就包含在编译系统之内的，我们必须在这个.mk中添加fpanel目录才能正常编译。
	
* ### 添加系统服务

	Webdroid是一个多进程系统，一般来说每个webapp就是一个进程，要访问像fpanel这样的稀有硬件资源，如果每个进程都直接调用硬件接口的话，可能会产生冲突，最好的办法是找一个中间代理人来管理，所有请求都经过代理人进行序列化操作，是众多的请求按照一定的顺序执行，这个代理人就是Service，就前面板来说就是FpanelService。
	
	在Framework框架中，我们一般会用到一种模式，就是Manager<->Service模式，XXXManager一般是由使用者（应用或者其它代码）调用的接口，每个进程都会产生一个Manager实例，而XXXService是跑在SystemServer进程中的，它只可能产生一个实例。因此就前面板来说，我们会有FpanelManager和FpanelService。
	
	服务定义好以后，应用如何去获得和调用这个服务呢？这就要带出Context这个东西了，Context顾名思义就是上下文的意思，对于程序来说就是一个运行环境，可以通过它的接口获取任何需要的功能，Context.getSystemService()接口是用来获取指定的服务的，因此只要有Context存在我们就能获得Service。同样也意味着，如果我们想提供Fpanel服务，我们就必须修改Context的实现。
	
	同时，由于我们的Service是采用java实现的，而我们的HAL层代码是C/C++的，因此我们必须使用JNI的方式使Java代码和C/C++代码能够互通。
	
	综上所述，这里我们涉及到的文件有：
	
	> $(WEBDROID)/frameworks/base/Android.mk
	> $(WEBDROID)/frameworks/base/core/java/android/app/ContextImpl.java
	> $(WEBDROID)/frameworks/base/core/java/android/content/Context.java
	> $(WEBDROID)/frameworks/base/core/java/android/dvb/FpanelManager.java
	> $(WEBDROID)/frameworks/base/core/java/android/dvb/IFpanelService.aidl
	> $(WEBDROID)/frameworks/base/services/java/com/android/server/SystemServer.java
	> $(WEBDROID)/frameworks/base/services/java/com/android/server/dvb/FpanelService.java
	> $(WEBDROID)/frameworks/base/services/jni/com_android_server_FpanelService.cpp
	> $(WEBDROID)/frameworks/base/services/jni/Android.mk
	> $(WEBDROID)/frameworks/base/services/jni/onload.cpp

* ### 添加JS接口

	服务做好以后，Java应用就可以直接使用了，但我们是webapp，因此还有一步工作要做，就是为webapp提供javascript接口。由于我们的应用框架采用的是cordova，引入接口的方式已经大大简化，在此不做深入展开。列出此步骤涉及到的文件：
	
	> $(WEBDROID)/frameworks/base/core/java/android/app/webapp/dvb/FpanelCore.java
	> $(WEBDROID)/frameworks/base/core/res/assets/webapp/dvb/Fpanel.js
	> $(WEBDROID)/frameworks/base/core/res/assets/webapp/webapp_plugins.json
	> $(WEBDROID)/frameworks/base/core/res/res/xml/webapp_plugins.xml

* ### 应用层面的调用方式

	略

* ### 其它方面的修改

	由于涉及到硬件操作，所以有些硬件配置需要做修改。这里以hisilicon为例，修改的文件如下：
	
	> $(WEBDROID)/device/hisilicon/hisi3716m/BoardConfig.mk
	> $(WEBDROID)/device/hisilicon/hisi3716m/configs/init.godbox.rc
	> $(WEBDROID)/device/hisilicon/hisi3716m/hisi3716m.mk
	
	主要功能是在系统启动的时候自动加载hi_keyled.ko驱动，并将HAL实现模块加入编译系统的默认编译列表。
