---
title: 三星NOTE9 折腾root小记
tag: 折腾
date: 2019-08-08
updated: 2019-08-08
---

过程比较曲折，前前后后搞了六七个小时总算root了。一开始想在mac上进行root的过程，但是在使用线刷软件的时候总是出问题，

#### 首先尝试在mac上面刷入TWRP
使用mac版的odin3 jodin的时候，总是提示需要PIT文件。当然刷TWRP没有带这个文件，然后按照提示进行PIT文件的获取总是失败。这个时候想到是不是网上能不能下一个，一番google以后找到一个，PIT文件装载后开始刷入但是仍然一直failed，想想还是用windows刷吧。

##### windows使用odin3刷入
刷入过程其实挺简单的，总结下来就这么几步：
* 开发者工具里oem解锁，不然不让你刷入和启动自定义的recovery，系统。
* 重启到download mode，odin3 AP选项里刷入TWRP recovery。
* 格式化data分区，安装no-verity-opt-encrypt-samsung-1.0.zip去除验证和root管理包Magisk。
* 重启，安装MagiskManager apk。

##### 如何重启到download模式
关机状态下，按住音量下+bixby键，插上usb线，会弹出一个蓝色界面，再按一次音量上即可。

##### 如何开机进入TWRP
线刷TWRP的时候，关闭auto reboot，刷完以后先按bixby+音量减+电源键退出download mode，在黑屏的一瞬间马上按bixby+音量上+电源键进入TWRP，如果没有成功进入，那么需要重新执行一遍线刷rec的过程。

<!-- more -->

##### 在刷入过程中遇到的问题：
* 线刷出现红色FAILED，提示**only official released binaries are allowed to be flashed**：我在oem解锁了以后恢复完出厂设置以后没有完成设置直接关机，然后打开odin3准备线刷TWRP就开始就出现了FAILED，并且手机上出现了这行红色英文。后来一检查KG State出现的是prenomal，解锁的状态应该是checking，这就表明并没有OEM解锁成功，重启以后进入开发者菜单发现找不到oem解锁的选项。后来网上经过查找，三星系统oem解锁系统重设后重启后需要先通过rmm.samsung.com验证oem解锁状态，然后在rmm相关存储里标记为解锁才是真正解锁。如果没联网，开发者选项里是看不到oem解锁这个选项的。解决方法嘛，再解锁一次oem，记得进行联网，在开发者选项里检查一下是否是已解锁状态，再重启到download模式就好了。

*  **TWRP刷完no-verity和magisk后进系统一直提示系统验证失败**：这个是data加密分区校验失败导致的，在TWRP刷zip的时候也一直提示无法正确加载data分区。解决方法直接在swipe选项里面格式化data分区，再刷一遍no-verity和magisk即可。要注意的是这个时候又恢复了出厂设置，要再重新设置一遍。

* **TWRP刷完no-verity和magisk进系统重启了以后再开机屏幕就一行红字：only official released binaries are allowed to be flashed**：当时看到这个提示的心想估计又是oem锁的问题，然后回忆了一下刚才的一波操作，嗯，重设完进系统后没联网去看oem解锁状态，又被锁上了。网上查了一下没啥好办法，只能下载完整原版rom重新刷一遍再重新root一遍的过程了。

* **odin3刷入卡在set partition**：刷原版rom过程中试了几次一直卡在set partition，后来检查了一下自己的odin版本用的是3.12，网上教程都是3.13，升级了一下版本解决了。
* **刷的时候提示can't open the specified file.** ：这个原因是勾上了re partition，又没有选择PIT分区文件导致一直报错，不勾选就行了。

在这过程中总结一下碰到最多的问题是oem锁的问题，进行相关操作的时候**一定要确保oem锁是在解锁状态**，特别是刷完TWRP，patch完系统后，记得联下网开发者菜单里查看一下oem解锁，不然重启直接进不去系统了。

在使用TWRP刷入no-verity和magisk的时候，有个小技巧，相关教程都推荐把需要刷入的zip实现拷到手机里，但是由于各种重设操作很容易导致内部存储里的数据消失或者无法加载，这个时候最方便访问相关刷机文件的时候是使用`adb push`命令推送到手机里：
`adb push xxx.zip /sdcard/xxx.zip`

相关下载：
[TWRP地址（google drive）](https://drive.google.com/file/d/1V2eUzokBdgIvzPubx6sD937DDhIJotd2/view)
[no-verity解除分区验证地址（google drive）](https://drive.google.com/file/d/1kUO-D10DyWKVaXiYvqqpEwaOSbCXfz6t/view)