---
title: web视频播放一杆到底
date: 2020-12-09 12:24:22
tags:
---
## 前言
毫无疑问，现在是短视频、直播的时代。视频内容逐渐代替图文形式成为网友们获取新鲜事物以及展现自我的一大媒介。随着5G的到来，2020年属于直播短视频爆发式增长的一年，电商平台也都涌入直播营销的大风口，成为了各自平台引流转化的关键。不管是用户还是开发者，我们处于这个风口中。本文将带你探索浏览器视频播放的奥秘。

## 视频的构成
一个完整可播放的视频文件是由视频和音频两部分构成。视频和音频又有各自的封装格式（容器）和编码格式。
1. **编码格式**：常见的视频编码格式有：MPEG4、H.264、H.265 等。常见的音频编码格式有：MP3、AAC、WAV 等。
2. **封装格式**：常见的视频封装格式有：MP4、FLV、mov、AVI、RMVB 等。

## 先理解几个名词
1. **帧**：就是影像动画中的最小单位的影像画面。一帧就是一张静止的图像。视频中的动画就是由多幅连续的帧画面构成。
2. **帧率**：帧率是以帧为单位的图像在显示器上出现的频率，也叫帧速率，单位：赫兹（Hz）。简单理解为每秒播放图片的数量。
3. **码率**：码率是比特率的俗称，是指每秒传送的比特数。
4. **FFmpeg**：FFmpeg 是可以用来记录、转换数字视频和音频的一套计算机程序。FFmpeg 是在 linux 下开发，所以天生跨平台。它对音视频编码格式的支持比较全面，能对视频的各个组成部分进行编码。
5. **H264**：通常被称之为 H.264/AVC；是由国际标准化组织和国际电信联盟共同提出的继MPEG4之后的新一代数字视频压缩格式。采用 H.264 压缩后的数据具有低码率、高质量图像、容错能力强、网络适应性强等优点。
6. **MP4**：MP4 是一种标准的数字多媒体容器格式；用于音频、视频的压缩编码，也可以存储字幕和静止图像，同时能以流的方式进行网络传输。
7. **fMP4（Fragmented MP4）**：fMP4 是基于 MPEG-4 Part 12 的流媒体格式，与 MP4 很相似。简单来说 fMP4 区别于 MP4 最大的区别就是它能很好地适应流式播放。

## 浏览器播放视频
1. **video标签播放**：在浏览器播放视频，可以使用 HTML5 原生的 video 标签。但其播放的格式使用一定限制的，目前 video 只支持三种格式 WebM、Ogg、MP4。
    - WebM：WebM 文件使用 VP8 视频编解码器和 Vorbis 音频编解码器
    - Ogg：Ogg 文件使用 Theora 视频编解码器和 Vorbis 音频编解码器
    - MP4：MPEG 4 文件使用 H264 视频编解码器和 AAC 音频编解码器
    ```html
        <video id="video-box" src="//cloud.video.taobao.com/play/u/755731755/p/1/e/6/t/1/283631891407.mp4" controls width="400px" heigt="400px"></video>
    ```
    在页面初始化完成后，video 标签会将整个 mp4 文件下载到浏览器，完成后即可播放。但是当 mp4 文件较大时，缓存时间就比较长，播放体验不好。当然也可以使用 **video.js** 来播放，这里就不赘述了。

2. **播放HLS流**：HLS（HTTP Live Streaming）是一个由 Apple 公司提出的基于 HTTP 的流媒体传输协议。视频的封装格式是 TS，编码格式是 H.264/ACC，除了定义 TS 视频文件本身，还定义了用来控制播放的m3u8 文本文件。移动端大部分浏览器都支持，也就是说，你可以在移动端浏览器直接使用 vedio 标签直接加载一个 m3u8 文件播放视频或者直播。但在 PC 端只支持苹果的 safari 浏览器，其他浏览器想播放需要引入第三方库，如：hls.js：
    ```html
        <script src="https://cdn.jsdelivr.net/npm/hls.js"></script>
        <video id="video"></video>
        <script>
        var video = document.getElementById('video');
        var videoSrc = 'https://test-streams.mux.dev/x36xhzz/x36xhzz.m3u8';
        if (Hls.isSupported()) {
            var hls = new Hls();
            hls.loadSource(videoSrc);
            hls.attachMedia(video);
            hls.on(Hls.Events.MANIFEST_PARSED, function() {
                video.play();
                });
        }
        </script>
    ```
播放 HLS 流的逻辑很简单，首先根据提供的 m3u8 地址源通过 HTTP 请求获取到一级 index 文件内容，例如：
    ```nginx
        #EXTM3U
        #EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=64000
        500kbps.m3u8
        #EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=774000
        1000kbps.m3u8
        #EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=887000
        500kbps.m3u8
        #EXT-X-STREAM-INF:PROGRAM-ID=1,BANDWIDTH=7692000
        1000kbps.m3u8
    ```
`bandwidth` 指定视频流的码率，每一个 `#EXT-X-STEAM-INF` 的下一行是二级 index 文件的路径，可以是相对路径或者是绝对路径。请求到的二级文件内容如下：

    ```nginx    
        #EXTM3U
        #EXT-X-PLAYLIST-TYPE:VOD
        #EXT-X-TARGETDURATION:10
        #EXTINF:10,
        1000kbps-00001.ts
        #EXTINF:10,
        1000kbps-00002.ts
        #EXTINF:10,
        1000kbps-00003.ts
        #EXTINF:10,
        1000kbps-00004.ts
        #EXTINF:10,
        ... ...
        #EXT-X-ENDLIST
     ```
可以从二级文件中读取到 ts 文件的路径，同样可以是相对路径或者绝对路径。`#EXTINF` 表示每个 ts 切片的时长。`#EXT-X-ENDLIST` 是视频结束标志，如果有这个标志也表明该流不是一个直播流。

    - **HLS播放的优势**：可以使用 http 协议请求数据流；可以切换不同的码率，实现无缝播放
    - **劣势**：延迟较高，实时性差，一般延迟在 10s 以上，不适合做直播；ts 文件切片小且多，对存储和缓存都有一定的要求

3. **播放FLV流**：FLV（Flash Video）是一种网络视频格式，FLV只能基于 flash 播放，但是由于 flash 存在很多安全问题已经被众多厂商抛弃，现在我们如果要在 H5 中播放 flv 格式的视频流可以使用 Blibli 的开源库：Flv.js，`flv.js`原理是解析视频的flv流并实时转换为 fmp4 格式，再通过 Media Source Extension 喂给浏览器的 `video` 标签。
    ```html
        <script src="flv.min.js"></script>
        <video id="videoElement"></video>
        <script>
        if (flvjs.isSupported()) {
            var videoElement = document.getElementById('videoElement');
            var flvPlayer = flvjs.createPlayer({
                type: 'flv',
                url: 'http://example.com/flv/video.flv'
            });
            flvPlayer.attachMediaElement(videoElement);
            flvPlayer.load();
            flvPlayer.play();
        }
        </script>
    ```
                
4. **基于 Media Source Extensions 播放视频流**：Media Source Extensions（媒体源扩展，缩写 MSE ）是一项 W3C 规范，MSE 允许 Javascript 为 audio 标签和video标签动态地构造媒体源。借助 MSE 的能力，我们可以将接收到的实时流通过 `blob url` 往 video 标签中灌入二进制数据（如 fmp4 格式流），或者使用 `canvas` 来实现直播。
    - **简单实现**：首先，判断浏览器是否支持 MediaSource：
    ```javascript
        const supportMediaSource = window.MediaSource &&
        typeof window.MediaSource.isTypeSupported === 'function' &&
        window.MediaSource.isTypeSupported('video/mp4; codecs="avc1.42c01f,mp4a.40.2"');
    ```
MediaSource支持情况：
    |  | Chrome | Edge | Firefox | Safari | Opera | Android |
    | --- | --- | --- | --- | --- | --- | --- |
    | MediaSource | 31 | 12 | 42 | 8 | 18 | 4.4.3 |
    | MediaSource() Constructor | 31 | 12 | 42 | 8 | 15 | 4.4.3 |
    | activeSourceBuffers | 23 | 12 | 42 | 8 | 15 | 4.4.3 |
    | addSourceBuffer | 23 | 12 | 42 | 8 | 15 | 4.4.3 |

接下来新建 `MediaSource` 实例，并使用生成 blob url 加到 video 标签。并且监听 `sourceOpen` 事件来判断初始化完成。

    ```javascript
        const mediaSource = new MediaSource();
        const video = document.querySelector('#video-box');
        video.src = URL.createObjectURL(mediaSource);

        mediaSource.addEventListener('sourceopen',function(){
            // TODO
        })
    ```
接下来我们通过 websocket 获取原始视频流，处理后通过 `SourceBuffer` 喂给 `mediaSource`

    ```javascript
        const sourceBuffer = mediaSource.addSourceBuffer('video/mp4; codecs="avc1.42E01E, mp4a.40.2"');
        const ws = new WebSocket("wss://xxx.websocket.com");

        ws.onopen = function(evt) { 
        console.log("Connection open..."); 
        ws.send("fetch Data");
        };

        ws.onmessage = function(evt) {
        // 可以在灌入数据前进行转码等操作
        sourceBuffer.appendBuffer(evt.data);
        };

        ws.onclose = function(evt) {
        console.log("Connection closed.");
        };
    ```
通过MSE的方式我们可以将接收到的视频或者音频流进行端处理，配合WebWorker技术实现快速转码、支持多播，给我们无限的想象空间。

5. **H.265 视频播放**：H.265 是 ITU-T VCEG 继 H.264 之后所制定的新的视频编码标准。H.265 标准围绕着现有的视频编码标准 H.264，保留原来的某些技术，同时对一些相关的技术加以改进。新技术使用先进的技术用以改善码流、编码质量、延时和算法复杂度之间的关系，达到最优化设置。但是由于浏览器不支持 H265 格式的流，所以我们无法直接播放。这时候可以使用 MSE 的方式在 `sourceBuffer.appendBuffer(evt.data)` 前将 `evt.data` 使用 libde265.js 等转码库转码后给到 `sourceBuffer`。或者使用业界成熟的播放器进行播放，如淘系的 @ali/videox 播放器。

## 总结
越来越多的厂商更加偏向于H.265的编码格式，但是浏览器对该格式的支持度不友好的前提下我们不得不进行转码。使用MSE方式在浏览器端转码，则能借助GPU提高效率和降低延迟。但还是无法兼容所有的PC或者移动端浏览器，这条路还需要我们去继续探索。5G给互联网带来的福利不仅仅是在视频、直播的爆发，我相信web端图像视频技术也将突破现有的技术瓶颈，WebAssembly、硬件编码等图像渲染技术也将越来越丰富。

## 参考
- baike.baidu.com/item/ffmpeg
- github.com/Bilibili/fl…
- developer.mozilla.org/zh-CN/docs/…

**标签**：JavaScript
