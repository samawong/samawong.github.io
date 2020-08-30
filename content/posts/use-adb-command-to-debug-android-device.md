---
title: "Use Adb Command to Debug Android Device"
date: 2020-08-30T18:16:27+08:00
description: 使用adb命令控制一台没有触摸功能的Android设备
Tags:
- Android
categories:
- 旁通杂粮
draft: false
---
![demo](https://res.cloudinary.com/samawong/image/upload/v1598792229/notes/adb_android/0_zscnv5.jpg "demo")
​手中有一台空气净化器的控制平板，屏幕触摸无外接件，所以无触摸功能，接口少，手边也没有OTG线，不然可以外接鼠标或键盘来进行操作，因此，请上Android下的adb命令来进行操作。

具体步骤如下：

1. 将设备通过USB线连接电脑；
2. 在命令窗口输入adb devices查看是否连接设备,如下图所示，会出现一长串字符串，连接多个设备的话会出现多条类似数据；
```
sama@w ~ % adb devices
List of devices attached
86589cb8c78700000000device
```
3. 使用adb shell 查看系统内已安装的软件包：
```
sama@w ~ % adb shell pm list packages
package:com.android.soundrecorder
package:com.android.launcher
package:com.android.defcontainer
package:com.android.providers.partnerbookmarks
package:com.android.contacts
package:com.android.phone
package:com.google.android.onetimeinitializer
package:com.google.android.partnersetup
package:com.android.calculator2
package:com.android.proxyhandler
package:com.android.htmlviewer
... ...
```
4. 查看已安装的软件包哪个不属于系统自带的（非android，Google的），将他们删除掉：
```
sama@w ~ % adb uninstall com.jinmaigao.shutdown
```
5. 重启查看系统是否已恢复android原本界面；
6. 因为没有触摸功能，所以后续的所有操作还得使用adb命令来进行操作；
7. 连接Wi-Fi：
主要使用以下几条命令：

```
adb shell input keyevent 66 //确定键

adb shell input keyevent 19 //向上

adb shell input keyevent 20 //向下

adb shell input keyevent 21 //向左

adb shell input keyevent 22 //向右

adb shell input text @@@@@@@@ //输入字符串“@@@@@@@@”
```
以上几个命令足以开启Wi-Fi，且大部分设置操作均可使用这些命令来执行操作；

8. 安装软件，因打算旧物利用，来制作个桌面时钟，所以下载了“新机表”这个软件；

新机表：酷安市场搜索下载

安装命令：


`
sama@w ~ % adb install path-to-app.apk
`



安装成功后会有Success字样；

9. 启动软件：

这里有两种方式，一一说明：

- 使用上面的上、下、左、右、确定命令直接打开软件；

- 查看新机表软件包名称，然后查询他的的Activity Resolver Table，然后使用adb命令直接操作该Activity:

```
sama@w ~ % adb shell pm list packages//查看安装的所有软件包

sama@w ~ % adb shell dumpsys package app.xunxun.homeclock
Activity Resolver Table:
  Non-Data Actions:
      android.intent.action.MAIN:
        41b96a78 app.xunxun.homeclock/.activity.MainActivity filter 41b92d00
          Action: "android.intent.action.MAIN"
          Category: "android.intent.category.LAUNCHER"


app.xunxun.homeclock/.activity.MainActivity即我们下一步要操作的字符串
```
- 在命令窗口接着输入以下命令：

` adb shell am start app.xunxun.homeclock/.activity.MainActivity`
可以在平板的屏幕上看到软件已打开。



其他的一些adb使用命令：

1、调整亮度：



`adb shell settings put system screen_brightness 150 //更改亮度值（亮度值在0—255之间）`
2、旋转屏幕：



`adb shell content insert --uri content://settings/system --bind name:s:user_rotation --bind value:i:1`


还有好多命令，本文不再赘述，有兴趣者请参阅文末参考文章链接。

然后使用旧台历做来个支架，不完美，效果还不错。



【参考文章】

1:https://mazhuang.org/awesome-adb/

2:https://blog.csdn.net/luckywang1103/article/details/76804856

3:https://blog.csdn.net/xajhsunei/article/details/78049762

4:https://blog.csdn.net/jimbo_lee/article/details/52168189
