---
layout: post
title: "bootloader实现规范"
categories: 开发
tags: Android bootloader
---

# BCB信息格式

* command[32] - 指令，可以为：
	* boot-recovery - 引导进入recovery系统
	* boot-fastboot － 引导进入fastboot模式
	* boot-main － 无条件引导进入主系统，oneshot
	* update-hboot － 更新bootloader firmware
* status[32] - 状态
	* 如果为空（内容全部为0或者全部为0xFF）则表示状态正常
	* 如果不为空则设置为出错的信息
* recovery[1024] - 传递给recovery系统的参数，每行一个参数，第一行必须是recovery，其它如下：
	* --send_intent=anystring 要求recovery将anystring写入/cache/recovery/intent文件
	* --update_package=path 指定升级包的路径
	* --wipe_data 重新格式化/data分区和/cache分区
	* --wipe_cache 格式化/cache分区
	* --set_encrypted_filesystem=on|off 是否支持加密文件系统
	* --just_exit 退出recovery而不做任何事情

# bootloader引导过程

1. 上电启动，进行必要的初始化
2. 读取misc分区的信息，将头32个字节存入bcb.command
3. 如果bcb.command内容为boot-recovery，则引导进入recovery
4. 如果bcb.command内容为boot-main，则**强制**引导进入主系统，并将misc分区清空
5. 如果bcb.command内容为boot-fastboot，则进入fastboot状态，等待客户端接入
6. 如果bcb.command内容为update-hboot，则读取misc分区的数据，将其覆盖掉bootloader，成功或者失败后将command内容写入boot-recovery，并将错误信息写入bcb.status，引导进入recovery系统
7. 如果command内容为空或者其它无效信息，则检查U盘，如存在recovery.command文件，并且其内容第一行为recovery，则将其内容写入bcb.recovery域，引导进入recovery系统，否则引导进入主系统

# 引导参数（bootargs）的处理

1. androidboot.serialno保存于环境变量
2. 在引导系统之前，必须将这个环境变量取出来附加在bootargs环境变量之后
3. 这样在主系统或者recovery里面，就可以通过读取属性ro.serialno来获得本设备的设备号

# 从USB升级

1. 从USB升级有两种方式，一是升级过程完全由bootloader实现，不同的bootloader可以有不同的实现方式，这里不做讨论，二是bootloader只是检测是否需要USB升级，然后引导进入recovery进行具体的升级动作，这个是我们要规范的方式
2. U盘的根目录下必须存在recovery.command文本文件，并且其内容必须和bcb.recovery规定的内容一致
3. 对于U盘的检查必须在BCB的检查之后，以确保BCB的高优先级
4. 升级完成后，recovery系统会写入boot-main到bcb.command，表示强制bootloader引导至主系统，同时擦除bcb信息
5. 这样USB升级完成后第一次重启系统不会再检查U盘中的文件，给用户足够的时间拔出U盘
