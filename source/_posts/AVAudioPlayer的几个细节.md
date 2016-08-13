title: AVAudioPlayer的几个细节
date: 2016-08-13 09:24:52
categories: iOS菜鸟心得
author: Sergio Chan

tags: [AVAudioPlayer , AVFoundation]
---

昨天在做 iOS 上的声波传输的时候，倒是遇到了几个和  AVAudioPlayer 有关的有趣问题，这种问题一般情况下我们都注意不到，只要踩过了才知道。


### 关于 PCM Data

`AVAudioPlayer` 有一个初始化方法 `initWithData:error:`，这个方法的 API 说明是

> /* all data must be in the form of an audio file understood by CoreAudio */

在苹果的文档里，我们看到 AVAudioPlayer并不能支持 Stream 播放，它支持的文件格式有下面这些：

| Format name           | Format filename extensions |
| --------------------- | -------------------------- |
| AIFF                  | `.aif`, `.aiff`            |
| CAF                   | `.caf`                     |
| MPEG-1, layer 3       | `.mp3`                     |
| MPEG-2 or MPEG-4 ADTS | `.aac`                     |
| MPEG-4                | `.m4a`, `.mp4`             |
| WAV                   | `.wav`                     |



Stream 类型的音乐流只能被 AudioQueue 或者 AudioUnit 支持。因此要用 `AVAudioPlayer` 来播放 PCM 数据的话，注意要为这个 PCM 包加上 WAV 的 HEADER，然后将完整的 NSData 传给它。



### 关于 Play

`AVAudioPlayer` 还有个有趣的现象，我暂时没有找到官方文档的证据，那就是它的 `play` 不会对自身有一个引用来保持自己是活着的。**只要它的父类之上有一个对象被释放了，那它也就被一起释放掉了**。因此无论你是在第一层直接声明 self.audioPlayer play 还是 self.A.audioPlayer.play ，它的最上层父类必须有一个和 VC 相关或者全局相关的强引用，否则就会在 play 的时候就已经被释放掉了。

