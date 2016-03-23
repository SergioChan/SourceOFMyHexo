title: 我是怎么踩过在 OSX 上录屏的坑的
date: 2016-03-23 23:55:44
categories: Cocoa入门？

tags: [Cocoa , AVFoundation]
---

昨天开始在研究 OSX 上的屏幕录制并且实时获取视频流或图像帧的实现。遇到了非常大的阻力，各种问题，昨晚纠结了一整晚，终于在小萌的启发下慢慢找到了解决办法，把谷歌和 stackoverflow 都翻了个底朝天，最后的解决有点意外，中间还是有一些细节需要求证，然而除了 Apple Doc 已经没有任何参考文献了，而有些机制 Apple Doc 中都不会涉及。所以此刻迫不及待的想要写一篇博客，来纪念万里长征的第一步。

要实现录屏，有两种途径，一种是通过 `Core Graphic`， 一种是通过 `AVFoundation`。 `Core Graphic` 的话，你可以找到苹果官方的一份 [SampleCode](https://developer.apple.com/library/mac/samplecode/SonOfGrab/Introduction/Intro.html)，如果使用了 

```
CGImageRef screenShot = CGWindowListCreateImage(CGRectMake(0.0f, 0.0f, [self screenRect].size.width, [self screenRect].size.height), kCGWindowListOptionOnScreenOnly, kCGNullWindowID, kCGWindowImageDefault |kCGWindowImageNominalResolution);
```

它的优点在于你可以根据 `WindowID` 来获取**指定窗口**的图像，并且可以通过 `ListOption` 来设定各种包括桌面图标，去除桌面图标，去除桌面，这些七七八八的设置，所以微信 Mac 端的截屏功能应该就是使用了上面这行代码。**所以我们也可以设置一个 NSTimer， 来按照六十分之一秒一帧的速度来获取截图，并且形成一个流。** 实践表明性能还不错，对于录屏这种事情烧一烧 CPU 是常有的事情，毕竟你需要按帧来计算像素，而且对于 Mac 而言，CPU 并不是什么特别大的问题 =。= 因此这种办法是**可行的**，然而我觉得不够优雅。

同样，Core Graphic 中还有一种实现办法：`CGDisplayCreateImage`：

```
CGImageRef Ref = CGDisplayCreateImage(display);
//NSData *data = (NSData *)CFBridgingRelease(CGDataProviderCopyData(CGImageGetDataProvider(Ref)));
screenImg = [[NSImage alloc] initWithCGImage:Ref size:CGDisplayScreenSize(display)];
//screenImg = [image mutableCopy];
CGImageRelease(Ref);
CGDisplayRelease (display);
```

这种实现的机制和上述的是一致的，实现出来的效果和性能也都不错，但是同样的还是觉得不够优雅。

所以此刻就要转向 `AVFoundation` 了。在 `AVFoundation` 中，有一个 input 类叫做 `AVCaptureScreenInput` 这个 input 直接可以获得到当前屏幕的视频输入。这时候我想起两年前我做过视频追踪人脸的 sdk，简单地说就是通过 `AVDeviceCapture` 来获取相机的 input 然后打开一个 `AVSession`， 然后再将 input 里面的 buffer 读出来，对每一帧进行人脸检测的运算。然后我按照苹果官方的一个录屏的例子和一个 Github 上存在不多的这方面的仓库实现了简单的录屏，使用了 `AVCaptureMovieFileOutput` 作为 output。到这里的时候，一切都很顺利，输出到 mov 文件的录屏都是正常的。然后我开始了从缓冲区读取 buffer 的工作，简单来说，从缓冲区读帧是根据 `AVCaptureFileOutputDelegate` 里面的一个回调 

```
- (void)captureOutput:(AVCaptureFileOutput *)captureOutput didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection;

```


来实现的。这里的 `CMSampleBuffers` 是一个 `Core Foundation` 的对象，它包含了零个或多个压缩或未压缩过的特定媒体类型的抽样，通常被用来传递媒体数据。一个 `CMSampleBuffers` 可以包含：

* `CMBlockBuffer`, 可能包含一个或多个的 sample (话说 sample 可以翻译为帧么？还是取样的意思……)
* `CVImageBuffer` 包含了 buffer 层级的附件和 sample 层级的附件，还包括了包含的所有 sample 的格式，大小和时间信息

按照 Apple Doc， 一个 `CMSampleBuffers` 就是这两种 buffer 之一的一个 wrapper， 因此每一个 `CMSampleBuffers` 只会包含其中之一。你需要用不同的方法来取出里面的数据。所以我就很正常的按照最正常的写法来取 buffer 了：

```
CVImageBufferRef imageBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
CVPixelBufferLockBaseAddress(imageBuffer,0);        // Lock the image buffer
        
uint8_t *baseAddress = (uint8_t *)CVPixelBufferGetBaseAddressOfPlane(imageBuffer, 0);   // Get information of the image
size_t bytesPerRow = CVPixelBufferGetBytesPerRow(imageBuffer);
size_t width = CVPixelBufferGetWidth(imageBuffer);
size_t height = CVPixelBufferGetHeight(imageBuffer);
CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();

CGContextRef newContext = CGBitmapContextCreate(baseAddress, width, height, 8, bytesPerRow, colorSpace, kCGBitmapByteOrder32Little | kCGImageAlphaPremultipliedFirst);
CGImageRef newImage = CGBitmapContextCreateImage(newContext);
CGContextRelease(newContext);

CGColorSpaceRelease(colorSpace);
CVPixelBufferUnlockBaseAddress(imageBuffer,0);
```

然而这个时候出了个小岔子，这里获取的 `CMSampleBuffers` 里面包的是 `CMBlockBuffer`！于是我开始查各种 stackoverflow， 无解， 一开始以为是视频格式的问题，需要按照 H264 的编码来解析，但是怎么可能呢…… 百思不得其解，即使我将 `CMBlockBuffer` 里面的 Data 读取了出来，也无法转换成 `NSImage`， 说明这个 Data 不是正常的 data。 那么有没有可能一帧被拆成多个 samples 来传输了呢…… 有可能，然而我尝试了仍然无果。

这时候我回头看看，发现我这里并没有将视频导出到文件的需求，有没有其他 output 来替代。偏巧我在 stackoverflow 上看到了[这个问题](http://stackoverflow.com/questions/15916808/capturing-blank-stills-from-a-avcapturescreeninput)，于是就用 `AVCaptureVideoDataOutput` 来尝试。尝试之前我已经有强烈预感了 - - 毕竟上一个 output 是直接输出到文件，而这个 output 明显是直接输出成 data。于是你只要这样给一个 output 就可以恢复正常了：

```
self.output  = [[AVCaptureVideoDataOutput alloc] init];
[((AVCaptureVideoDataOutput *)self.output) setVideoSettings:[NSDictionary dictionaryWithObjectsAndKeys:@(kCVPixelFormatType_32BGRA),kCVPixelBufferPixelFormatTypeKey, nil]];
dispatch_queue_t queue = dispatch_queue_create("com.sergio.chan", 0);
[(AVCaptureVideoDataOutput *)self.output setSampleBufferDelegate:self queue:queue];
```

这时候的 sampleBuffer 已经可以正常按帧解析出来了，这里有两个问题，一个是在上面那段代码获取到一个 `CGImageRef` 的 `newImage` 对象后需要每一次都对 newImage 进行一次release，否则内存溢出就要爆炸了，一个是线程安全问题，在上面的代码里可以看出这个新的 `AVCaptureVideoDataOutputSampleBufferDelegate` 其实是在一个独立的线程上接收回调的，因此如果你要在这个 delegate 中进行 UI 操作的话，记得回到主线程操作 =。=

```
@try {
    CVImageBufferRef imageBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);
    CVPixelBufferLockBaseAddress(imageBuffer,0);        // Lock the image buffer
    
    uint8_t *baseAddress = (uint8_t *)CVPixelBufferGetBaseAddressOfPlane(imageBuffer, 0);   // Get information of the image
    size_t bytesPerRow = CVPixelBufferGetBytesPerRow(imageBuffer);
    size_t width = CVPixelBufferGetWidth(imageBuffer);
    size_t height = CVPixelBufferGetHeight(imageBuffer);
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    
    CGContextRef newContext = CGBitmapContextCreate(baseAddress, width, height, 8, bytesPerRow, colorSpace, kCGBitmapByteOrder32Little | kCGImageAlphaPremultipliedFirst);
    CGImageRef newImage = CGBitmapContextCreateImage(newContext);
    CGContextRelease(newContext);
    
    CGColorSpaceRelease(colorSpace);
    CVPixelBufferUnlockBaseAddress(imageBuffer,0);
    
    NSImage *image = [[NSImage alloc] initWithCGImage:newImage size:[self screenRect].size];
    CGImageRelease(newImage);
    
    dispatch_async(dispatch_get_main_queue(), ^{
        if(self.imageView) {
            self.imageView.image = image;
        }
    });
}
@catch (NSException *exception) {
    NSLog(@"Error at %@",exception.debugDescription);
}
@finally {
    return;
}
```

PS. Cocoa 中获取 ScreenRect 的方法如下：

```
- (NSRect)screenRect
{
    NSRect screenRect;
    NSArray *screenArray = [NSScreen screens];
    NSScreen *screen = [screenArray objectAtIndex: 0];
    screenRect = [screen frame];//[screen visibleFrame];
    
    return screenRect;
}

```
> 这里后来又遇到一个小坑。如果使用的是 visibleFrame， 那么如果你的窗口处于全屏模式，获取 visibleFrame 的时候其实会把上面状态栏的那部分区域给省略了，因为计算 visibleFrame 的时候估计不考虑状态栏是否隐藏吧，所以这里用 frame 更好。

这里从 delegate 中获取到每一帧的数据之后就可以对每一帧进行压缩，并且以 Data 的形式进行传输了。差点忘记最后介绍一下 `AVCaptureScreenInput` 的一些特性了：

```
self.input.capturesMouseClicks = YES;
self.input.minFrameDuration = CMTimeMake(1, 60);
self.input.scaleFactor = 0.5f;
self.input.cropRect = [self screenRect];
```

首先 `AVCaptureScreenInput` 可以记录下鼠标移动的轨迹，还可以记录鼠标的点击事件（自行体验），第二个属性设置的是最大帧率，也就是60帧一秒。第三个和第四个属性顾名思义分别是缩放的比例和最后输出的裁剪区域，设置这两个属性可以减少每一帧的大小，也就是说在输入的时候就已经限制过大小了，然后你再可以进行一些压缩什么的。最后其实 `AVCaptureScreenInput` 还有一个关键的属性，但是现在已经被废弃了，因为苹果已经把这个属性内置成系统默认了😂 **重复帧会被自动取消**，这在以前的版本是可以通过一个属性设置的，现在已经被默认采用了。

多余的说几点：

* 其实 Core Media 那层有很多知识点，但是苦于文档太少，研究的人也太少，因此实在是举步维艰，感兴趣的朋友可以参考一下[苹果的 Reference ](https://developer.apple.com/library/mac/documentation/CoreMedia/Reference/CMSampleBuffer/)看下这块的内容。
* 其实可能有些人知道在 `AVFoundation` 下面，`Core Media`之上还有一层叫做 `Video ToolBox`，这在2012年那会儿都是只有越狱的设备才能调用到的 Private API，但是2014年的 WWDC 苹果将这一层开放出来了，因此你可以在 `AVFoundation` 更深入的层次去做视频编码解码和流处理，这块的知识我这次只看了个大概，留下了一些资料出处：[Github](https://github.com/McZonk/VideoToolboxPlus)  [WWDC](https://developer.apple.com/videos/play/wwdc2014/513/)

最后，最重要的是！代码已经整理成开源库放在 [Github](https://github.com/RavenTech-GrowthHacker/RTScreenRecorder) 上了！