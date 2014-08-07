---
layout: post
title: "Recovery相关问题"
categories: 开发
tags: Android Recovery
---


# 进入RECOVERY的方式


#### 恢复出厂设置


1. 主系统下用户选择“恢复出厂设置”
2. 主系统写入命令“--wipe_data”到文件/cache/recovery/command
3. 主系统重新启动，并设置bootloader control block (BCB)为启动到recovery系统，如何设置后面会说到
4. bootloader检查BCB参数，如果是boot-recovery，则引导recovery系统
5. recovery将检查/cache/recovery/command文件，并擦除userdata/cache分区数据
6. 完成动作后擦除BCB信息
7. 重启进入主系统


#### OTA升级


1. 主系统接收升级服务器推送包，存入/cache/xxx.zip
2. 主系统写入命令“--update_package=/cache/xxx.zip”到文件/cache/recovery/command
3. 主系统重启，设置BCB参数为boot-recovery
4. bootloader检查BCB，如果是boot-recovery则进入recovery
5. 尝试安装升级包
6. 升级成功后擦除BCB
7. 如果升级失败，提示用户重启
8. 检查是否更新固件（bootloader）
	1. recovery将boot-recovery和--wipe_cache写入BCB
	2. recovery将固件映像写裸入cache分区
	3. recovery将update-radio/hboot和--wipe_cache写入BCB
	4. 重启进入bootloader
	5. bootloader尝试将/cache的内容烧入到固件分区
	6. bootloader将boot-recovery和--wipe_cache写入BCB
	7. 进入recovery格式化/cache分区
	8. recovery擦除BCB
9. 重启系统


#### USB升级


1. 插入U盘，重启系统
2. bootloader检查U盘根目录是否有文件recovery.command
3. 有则读取文件内容，将内容写入BCB
4. 引导进入recovery
5. 其它同OTA升级


# BCB(bootloader control block)格式说明


1. 总共分为三个部分：command/status/recovery，其中command占用32字节，status占用32字节，recovery占用1024字节
2. BCB保存在misc分区里，对于nand来它占用3个页面大小，但实际数据存在于第2个页面上，读取和写入时要特别注意保持一致，mmc则直接按顺序存储
3. command支持的取值有boot-recovery（进入recovery）、boot-hboot（刷bootloader）、boot-radio（刷电话固件）
4. status由bootloader填入，主要表示刷固件的结果
5. recovery数据是传递给recovery程序的参数，每一行表示一个参数，第一行参数必须为recovery，其它参数有
	* --send_intent=anystring - 要求recovery将anystring存入/cache/recovery/intent文件
	* --update_package=path - 指定OTA包的位置
	* --wipe_data - 重新格式化/data和/cache分区
	* --wipe_cache － 只格式化/cache分区
	* --set_encrypted_filesystem=on|off － 文件系统是否加密
	* --just_exit － 不做任何其它事情


# bootloader的实现


每个平台都有自己的bootloader，基本上都是基于u-boot的某个版本的深度定制版本，理论上我们可以使用u-boot来统一bootloader的实现，但考虑到有的平台私货太多，支持又差，统一起来有很大的压力，所以暂时还是采用各家的私有bootloader，但是为了达到平台的表现一致性，特提出bootloader的一般性功能规范：

* 必须支持android boot image格式的镜像；
* 必须读取BCB信息以确定引导哪个系统；
* 必须支持stbid的保存并将其作为androidboot.serialno的值传递给kernel；
* 支持fastboot协议（这个目前还都没有实现）；
* 支持U盘升级；
* 支持我们的烧写工具；
