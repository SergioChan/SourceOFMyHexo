title: PHP图片处理 透明水印的处理和添加
date: 2014-12-26 9:11:25
categories: PHP大神养成
tags: [PHP图片处理，水印]
---

最近玩了下PHP的图片处理，实现了一个给图片添加水印或者是重叠两张图片的功能，把自己的解决过程记录下来。
首先我需要将图片resize到640*480的尺寸，重绘的代码如下：

```
$thumb = imagecreatetruecolor($newwidth, $newheight);
$source = imagecreatefromjpeg($filename);
imagecopyresampled($thumb, $source, 0, 0, 0, 0, $newwidth, $newheight, $width, $height);
imagejpeg($thumb, $filename);
```

在这段重绘的代码里，包含了几个很基本的东西：

第一个是imagecreatetruecolor和imagecreate
用imagecreatetruecolor(int x,int y)建立的是一幅大小为x和y的黑色图像(默认为黑色)，如想改变背景颜色则需要用填充颜色函数imagefill($img,0,0,$color)，imagecreate 新建一个空白图像资源，用imagecolorAllocate()添加背景色：

```
$img = imagecreatetruecolor(100,100);
$color = imagecolorAllocate($img,200,200,200);
imagefill($img,0,0,$color);
header('content-type:image/jpeg');
imagejpeg($img);
$img = imagecreate(100,100);
$color = imagecolorallocate($img,200,200,200);
header('content-type:image/jpeg');
imagejpeg($img);
```

第二个是imagecreatefromjpeg，这个是从图片文件创建一个新图像。
支持下面这些格式的图片文件。

```
function imagecreatefrompng ($filename) {}
function imagecreatefromgif ($filename) {}
function imagecreatefromjpeg ($filename) {}
function imagecreatefromwbmp ($filename) {}
function imagecreatefromxbm ($filename) {}
function imagecreatefromxpm ($filename) {}
function imagecreatefromgd ($filename) {}
function imagecreatefromgd2 ($filename) {}
```

第三个是imagecopyresampled，这个函数在裁剪，缩放图像时都特别有用：

```
/**
* (PHP 4 &gt;= 4.0.6, PHP 5)
* Copy and resize part of an image with resampling
* @link http://php.net/manual/en/function.imagecopyresampled.php
* @param resource $dst_image
* @param resource $src_image
* @param int $dst_x
* x-coordinate of destination point.
*
* @param int $dst_y
* y-coordinate of destination point.
*
* @param int $src_x
* x-coordinate of source point.
*
* @param int $src_y
* y-coordinate of source point.
*
* @param int $dst_w
* Destination width.
*
* @param int $dst_h
* Destination height.
*
* @param int $src_w
* Source width.
*
* @param int $src_h
* Source height.
*
* @return bool true on success or false on failure.
*/
function imagecopyresampled ($dst_image, $src_image, $dst_x, $dst_y, $src_x, $src_y, $dst_w, $dst_h, $src_w, $src_h) {}
```

第四个是imagejpeg，这个函数如果不提供第二个filename的参数，会将图片文件直接输出出来，加上了这个参数就会按照方法名里的文件格式输出到文件中去。返回的是BOOL值。

接下去就是加上水印的这个函数了，我添加了一个newwidth的参数，是希望在进入这个函数前计算出需要的水印大小，传入函数，这样生成的水印大小就会不一样了，可以根据不同的需要变化。

```
function mark_pic($background, $waterpic, $x, $y, $new_width)
{
$back = imagecreatefromjpeg($background);
$water = imagecreatefrompng($waterpic);
$w_w = imagesx($water);
$w_h = imagesy($water);
imagesavealpha($water, true);
if ($new_width != $w_w)
{
$new_height = $w_h * ($new_width / $w_w);
$thumb = imagecreatetruecolor($new_width, $new_height);
$c = imagecolorallocatealpha($thumb, 0, 0, 0, 127);
//拾取一个完全透明的颜色
imagealphablending($thumb, false);
//关闭混合模式，以便透明颜色能覆盖原画布
imagefill($thumb, 0, 0, $c); //填充
imagesavealpha($thumb, true); //设置保存PNG时保留透明通道信息
imagecopyresampled($thumb, $water, 0, 0, 0, 0, $new_width, $new_height, $w_w, $w_h);
imagepng($thumb, $waterpic);
imagedestroy($water);
$water = imagecreatefrompng($waterpic);
$w_w = imagesx($water);
$w_h = imagesy($water);
}
$b_w = imagesx($back);
$b_h = imagesy($back);
imagecopy($back, $water, $x, $y - $w_h , 0, 0, $w_w, $w_h);
imagejpeg($back, "./files/test.jpg");
imagedestroy($back);
imagedestroy($water);
}
```

在这个函数里，最重要的一个事情其实在于重绘一个具有透明通道的图像。如果使用imagecreate，透明通道就会被消灭掉，因此我最后只能采用了以上的办法，通过imagecreatetruecolor，往上面覆盖一个完全透明的颜色，来保留图片的透明通道，这对于添加水印这个功能应该是至关重要的。
