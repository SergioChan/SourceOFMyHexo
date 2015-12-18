title: '[UIColor colorWithRed: green: blue: alpha:] 的正确用法'
date: 2015-2-2 21:37:10
categories: iOS菜鸟心得
author: Sergio Chan
tags: [UIColor]
---

[UIColor colorWithRed: green: blue: alpha:] 颜色值范围都是在0.0~1.0之间的，并不是我们误认为的0~255。

正确用法：

```
[UIColor colorWithRed:240.0/255 green:240.0/255 blue:240.0/255 alpha:1.0];
```

> colorWithRed:green:blue:alpha:
> Creates and returns a color object using the specified opacity and RGB component values.
> 
> Declaration
> OBJECTIVE-C
> + (UIColor *)colorWithRed:(CGFloat)red
> green:(CGFloat)green
> blue:(CGFloat)blue
> alpha:(CGFloat)alpha
> 
> Parameters
> 
> red
> 
> The red component of the color object, specified as a value from 0.0 to 1.0.
> 
> green
> 
> The green component of the color object, specified as a value from 0.0 to 1.0.
> 
> blue
> 
> The blue component of the color object, specified as a value from 0.0 to 1.0.
> 
> alpha
> 
> The opacity value of the color object, specified as a value from 0.0 to 1.0.
> 
> Return Value
> 
> The color object. The color information represented by this object is in the device RGB colorspace.
> 
> Discussion
> Values below 0.0 are interpreted as 0.0, and values above 1.0 are interpreted as 1.0.
> 
> Import Statement
> Availability
> Available in iOS 2.0 and later.

