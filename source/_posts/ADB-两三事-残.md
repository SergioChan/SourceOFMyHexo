title: ADB 两三事-残
date: 2016-10-24 12:24:50
categories: 安卓菜鸟心得


tags: [Android, ADB, Android Debug Bridge, 残念, 智障]
---



折腾了两天 ADB， 宛若一个智障。遇到如下问题。

## 从电脑 push 多个文件到设备 : wildCard

参考[这篇](http://xmodulo.com/how-to-copy-or-transfer-multiple-files-to-android-devices.html)。

一开始用 \*.\* 的方式将文件夹下的所有文件 push 到设备的目录下，但是始终失败。查无果，只看到了上面这篇说不支持的，于是看了下自己的 ADB 版本号，1.0.32。确认这个版本是不支持 wildCard 匹配文件的。遂将 \*.\* 这种表示方式改成了 shell 中的 for 循环。

```
for i in `ls /Users/sergiochan/xxx/`
do
echo "/Users/sergiochan/xxx/$i"
adb push "/Users/sergiochan/xxx/$i" /system/{destination}/
adb push "/Users/sergiochan/xxx/$i" /system/{destination}/{subDestination}/
done
```

后来试了一下才发现升级到 1.0.36 是可以支持的。遂，卒。

## ADB remount 显示 succeeded 却并没有成功挂载

重启大法好哈哈哈哈哈哈。

## 想要在 Android App 中运行 shell？

可以说很麻烦么 =3=

除非你是以 root 用户运行的 app，否则如果是普通用户的话，连 system 路径下的文件都没法修改，更别说 su 了。不过，如果想要修改 system 路径下的文件，可以在 manifest 中加上 **android:sharedUserId="android.uid.system"** 让你的 app 以 system 用户运行，然后就可以直接通过 **FileOutputStream** 的方式修改 system 路径下的文件，例如控制 CPU 内存什么的。

想要运行 shell ，可以在 ADB 命令行通过 ps 命令查看你的 package 运行的用户是谁，例如用户 test，如果你 `su test` 之后再运行 `su` 显示的是 **Permission Denied**，那就没有办法了。如果 root 过后的设备给你的运行用户配了用户权限什么的，那记得在 Android 中执行 shell 之前，执行一下 `su` :

```
try {
	Process process = Runtime.getRuntime().exec("su");
	process.waitFor();
} catch (IOException e) {
	e.printStackTrace();
} catch (InterruptedException e) {
	e.printStackTrace();
}
```





