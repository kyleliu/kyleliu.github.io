---
layout: post
title: "如何获取系统调用的调用栈"
categories: 开发
tags: Android hook
---


# 如何获取系统调用的调用栈

#### 问题描述

机顶盒在进行老化测试的时候发现，放置一段长时间后，盒子会因为分配不到文件描述符(file descriptor)而重启，可以肯定有fd的泄漏问题。通过lsof发现泄漏的是一个pipe（包含两个fd），但是查找发现系统中使用pipe的地方很多，从review源代码的角度无从下手，只能用一些非常规的方式了。

要定位这个bug，需要解决两个问题：一是要知道是谁调用的pipe，也就是调用pipe的调用栈；二是要拦截pipe的系统调用，添加我们自己的调试代码。

下面就这两个问题详细说明一下。

#### 如何获取调用栈

我们系统是一个深度定制的android系统，使用android系统的好处是有很多现有的功能可以使用，其中就包括调用栈的获取。android提供了一个CallStack的类（见system/core/include/utils/CallStack.h）来收集和打印调用栈，使用方式如下：

	#include <utils/CallStack.h>
	...
	void foo(void) {
	...
	    android::CallStack stack;
	    stack.update();
	    stack.log("XXX");
	...
	}

由于并不是每一次调用都需要打印出来，也可以将其保存在一个buffer中，使用如下方法：

	String8 toString(const char* prefix = 0) const;

对于泄漏的pipe调用来说，我们期望做到的是只打印那些还没有释放的pipe调用，如何做到呢？可以将每次pipe调用的调用栈保存在一个hashmap中，以创建的fd为key保存，当fd关闭的时候再从hashmap中移除，这样内存中永远只保留有尚未关闭的pipe调用栈。hashmap可以使用android提供的system/core/include/cutils/hashmap.h来实现。

#### 如何拦截系统调用

系统调用pipe()的地方很多很多，再没有地方添加代码显然是不现实的。

linux(android)的动态库loader有一种机制，可以通过指定环境变量LD_PRELOAD来预先加载一个动态库，这个动态库中导出的符号可以覆盖其它动态链接库的相同符号，也就是说我们可以写一个自己的动态链接库，用此动态链接库中的函数来覆盖其它动态链接库中的同名函数。

我们的pipe()函数可以这样来实现：

	#define _GNU_SOURCE
	
	#include <stdio.h>
	#include <dlfcn.h>
	#include <utils/CallStack.h>
	
	extern "C"{
	
	int pipe(int fds[2]) {
		// 打印调用栈
		android::CallStack stack;
		stack.update();
		stack.log("Pipe");
		
		// 最后调用原始的pipe(...)接口
	    int (*original_pipe)(int*);
	    original_pipe = dlsym(RTLD_NEXT, "pipe");
	    return (*original_pipe)(fds);
	}
	
	} // extern "C"

注意_GNU_SOURCE必须定义，不然找不到RTLD_NEXT宏；RTLD_NEXT表示在后来加载的动态链接库中查找符号；链接时候需要加入-ldl参数，因为我们用到了dlsym。

编译链接成功后，得到一个动态库，假设保存在目标板的/system/lib/libdbg.so，使用时可以在init.rc中需要调试的进程前加上LD_PRELOAD=/system/lib/libdbg.so，如：

	service zygote LD_PRELOAD=/system/lib/libdbg.so /system/bin/app_process ...   		...

当然，由于zygote服务进程负责派生所有其他应用进程，因此所有这些进程对pipe的调用都会打印出来，会有很多，因此需要我们在pipe的跟踪函数中过滤那些我们不需要的进程，可以通过进程号或者进程名称等手段过滤。