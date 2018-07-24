title: 做 Custom Keyboard 的时候遇到的几个问题
date: 2016-12-07 15:05:19
tags: [Custom Keyboard Extension, iOS]
---

最近又在搞个大新闻。记录了以下几个问题和窍门。

## Customize your keyboard height

在 **ViewDidLoad** 加入这段代码，在任意位置修改这个 constraint 的值然后调用 **updateViewConstraint** :

```
CGFloat _expandedHeight = 300;
_heightConstraint = [NSLayoutConstraint constraintWithItem: self.view
                             attribute: NSLayoutAttributeHeight
                             relatedBy: NSLayoutRelationEqual
                                toItem: nil
                             attribute: NSLayoutAttributeNotAnAttribute
                            multiplier: 0.0
                              constant: _expandedHeight];
[self.view addConstraint: _heightConstraint];
```

## Network Access and other accesses

一般情况下 Keyboard Extension 是没有访问网络的权限的，因为键盘是比较涉及用户隐私的插件，虽然其实真正能获得的信息很少的 =。= 都被沙盒保护起来了。但是毕竟能获得用户输入的文本信息，有可能你就是个坏人来收集用户输入的用户名密码什么的，对吧。所以需要在取得用户信任的基础上，让用户手动去设置里为键盘插件打开 **完全访问** ，你的插件才可以访问网络，位置信息这些。

在获得权限之前，任何对网络的访问都会报一大串错误，找不到 DNS，服务未启动什么的。你只需要在 **info.plist** 中设置 **NSExtensionAttributes** 中的 **RequestsOpenAccess** 为 YES， 用户就可以在设置页面中找到打开完全访问的开关了。

## OpenURL in Keyboard Extension

由于 **[UIApplication sharedApplication]** 在 Extension 中不可用，openURL 只能通过 **extensionContext** :

```
[self.extensionContext openURL:completionHandler:];
```

然而尴尬的是在 Today Extension 中是可以用的，在 Keyboard Extension 中却没有用。于是找到了下面这个更新的解决方案，直接通过 **responder chain** 找到 UIApplication = =

```
UIResponder* responder = self;
while ((responder = [responder nextResponder]) != nil) {
    if ([responder respondsToSelector:@selector(openURL:)] == YES) {
        [responder performSelector:@selector(openURL:)
                        withObject:[NSURL URLWithString:@"sergio.chan.xxx://"]];
    }
}
```

