# flv.js源码解读笔记

在没有 MSE（[Media Source Extensions](https://developer.mozilla.org/zh-CN/docs/Web/API/Media_Source_Extensions_API)） 出现之前，javascript对 video 的操作，仅仅局限在对视频文件的操作，而并不能对视频流做任何相关的操作。现在 MSE 提供了一系列的[接口](https://developer.mozilla.org/en-US/docs/Web/API/MediaSource)，使开发者可以直接提供 media stream。

如果没有 `Media Source Extensions` 的帮助，flv.js不复存在。其实flv.js有挺多参考了[hls.js](https://github.com/video-dev/hls.js)，两个的实现原理都是一样的。

在进行源码解读前，还需要了解一些基础，如视频编码、视频格式、flv的文件格式定义等等。

**本文有参考网上的技术文章，有些还是直接采取，未经修改！**。

## 视频格式？编码？

为了更好理解后续的源码，我们需要有一定视频基础知识，需要理解[视频文件格式](https://zh.wikipedia.org/wiki/%E8%A7%86%E9%A2%91%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F)（容器格式），[视频编解码器](https://zh.wikipedia.org/wiki/%E8%A7%86%E9%A2%91%E7%BC%96%E8%A7%A3%E7%A0%81%E5%99%A8)（视频编码格式）。

如何创建一个视频？简单说，需要将视频/音频数据流会分别进行相关处理，简而言之就是，将视频/音频流，通过一定的手段转换为比特流。最终，将这里比特流以一定顺序放到一个盒子里进行存放，从而生成我们最终所看到的视频文件，比如，mp4，mp3，flv 等等音视频格式的文件。

### 视频文件格式

简单直观`视频文件格式`，就是指加了.flv、.mp4的后缀名文件名（当然没那么简单）。视频文件格式实际上，我们常常称作为容器格式，也就是，我们一般生活中最经常谈到的格式，flv，mp4，ogg 格式等。**它就可以理解为将比特流按照一定顺序放进特定的盒子里。**

> 服务端把音、视频流按照指定的视频容器（flv，mp4等）封装好视频（英文叫remux），然后播放器从服务器端中拉去视频流播放，这时我们需要对指定的视频容器进行解码（英文叫demux）,这样才能取到视频流进行播放。

那选用不同格式来装视频有什么问题吗？ 答案是，没有任何问题，但是你需要知道如何将该盒子解开，并且能够找到对应的解码器进行解码。

### 编码格式

视频编码格式就是我们上面提到的第一步，将物理流转换为比特流，并且进行压缩。同样，它的压缩编码格式会决定它的视频文件格式。所以，第一步很重要。针对于 HTML5 中的 video/audio，它实际上是支持多种编码格式的，但局限于各浏览器厂家的普及度，目前视频格式支持度最高的是 MPEG-4/H.264，音频则是 MP3/AC3。（下面就主要说下视频的，音频就先不谈了。）

目前市面上，主流浏览器支持的几个有：

- H.264（MEPG-4 第 10 部分第）
- MEPG-4 第 2 部分
- Ogg
- WebM（免费）

### 常用的视频格式和编码格式

这里是列举主页的格式，可能还有其他的没列出来。

| 视频格式  | 容器格式   | 视频编码           | 音频编码    |
| ----- | ------ | -------------- | ------- |
| .mp4  | MPEG-4 | AVC/H.264,xvid | AAC,MP3 |
| .ogv  | OGM    | Theora         | Verbis  |
| .webm | WebM   | Theora         | Verbis  |
| .flv  | F4V    | AVC/H.264      | AAC/MP3 |

### 小结

html5 video只支持mp4,webm等解码，并不支持flv的解码，所以需要先将flv视频解码，然后再封装成mp4容器，然后通过`MSE` 传递给html5 video就可以播放了。

由上面的视频编码格式可知flv的音视频编码格式完全可以符合mp4的音视频编码格式，所以容器格式转换后的视频理论上是可以正常播放的。webm的编码格式转MP4理论上就播不了的，因为MP4不支持这些编码格式。

## 二进制

### 运算

`视频容器`mp4、flv等都是二进制数据，源码会涉及到很多的二进制运算。

二进制的左边全0是可以省略的，但是我们一般都是按照4位的倍数表示二进制。

- `&`（与）

  **（x&y）两二进制上下比较只有位值都为1时才取1，否则取0。**

  > 14 & 15 = 14;
  >
  > 14的二进制：1110
  >
  > 15的二进制：1111
  >
  > 14  &  15    ：1110
  > 转为10进制：14

- `|`（或）

  **（x|y）两二进制上下比较只有位值都为0时才取0，否则取1。**

  >14 | 15 = 15;
  >
  >14的二进制：1110
  >
  >15的二进制：1111
  >
  >14  | 15      ：1111
  >转为10进制：15

- `^`（异或）

  **（x^y）两二进制上下比较只有位不相等时才取1，否则取零。**

  >14 ^ 15 = 15;
  >
  >14的二进制：1110
  >
  >15的二进制：1111
  >
  >14  ^ 15      ：0001
  >转为10进制：1

- `~`（非）

  **（~x）二进制取反。**

  > ~5 = -6
  >
  > 二进制原码：0000 0000 0000 0000 0000 0000 0000 0101 
  > 取反操作后：1111 1111 1111 1111 1111 1111 1111 1010 
  > 有符号**整数**都是用补码来表示，而补码=反码+1 
  > **可以简单理解为数字是有符号，非数字是无符号。**
  >
  > 1.先求反码：1000 0000 0000 0000 0000 0000 0000 0101 
  > 2.再求补码：1000 0000 0000 0000 0000 0000 0000 0110
  >
  > 当我们指定一个数量是无符号类型时，那么其最高位的1或0，和其它位一样，用来表示该数的大小。 
  > 当我们指定一个数量是有符号类型时，此时，最高数称为“符号位”。为1时，表示该数为负值，为0时表示为正值。
  > 最高位代表符号位 1 表示负数，0 表示正数 ，**简单理解为负数是有符号的。**

- `<<`（左移）

  **左移运算法则：将数值向左移动若干位，用0补足 **

  > 5 << 1 = 10
  >
  > 二进制原码：0000 0101
  >
  > 左移一位后：0000 1010
  >
  > 转为10进制：10

- `<<`（左移）

  **右移运算法则：将数值向右移动若干位，右边的被移动后不在0位置的值，直接移除。

  >5 >> 1 = 2
  >
  >二进制原码：0000 0101
  >
  >右移一位后：0000 0010
  >
  >转为10进制：2

### 字节序

计算机硬件有两种储存数据的方式：大端字节序（big endian）和小端字节序（little endian）。详细请参考，[理解字节序](http://www.ruanyifeng.com/blog/2016/11/byte-order.html)。

如果不确定正在使用的计算机的字节序，可以采用下面的判断方式。

```js
var littleEndian = (function() {
  var buffer = new ArrayBuffer(2);
  new DataView(buffer).setInt16(0, 256, true);
  return new Int16Array(buffer)[0] === 256;
})();
```

## FLV的文件格式

FLV的文件格式很有必要了解，说白了就是flv格式定义文档。可以参考[这里](http://akagi201.org/post/http-flv-explained/)，这里就不详说。

**单位说明**

| 说明                                       | 类型              |
| ---------------------------------------- | --------------- |
|                                          | Unit data types |
| Signed 8-bit integer（大小为8比特，相当于一个字节）     | SI8             |
| Signed 16-bit integer                    | SI16            |
| Signed 24-bit integer                    | SI24            |
| Signed 32-bit integer                    | SI32            |
| Signed 64-bit integer                    | SI64            |
| Unsigned 8-bit integer                   | UI8             |
| Unsigned 16-bit integer                  | UI16            |
| Unsigned 24-bit integer                  | UI24            |
| Unsigned 32-bit integer                  | UI32            |
| Unsigned 64-bit integer                  | UI64            |
| Slice of type xxx                        | xxx[]           |
| Array of type xxx                        | xxx[n]          |
| Sequence of Unicode 8-bit characters (UTF-8), terminated with 0x00 | STRING          |

> 从整个文件上看, FLV = FLV File Header + FLV File Body

**FLV File Header字段说明**

FLV文件头总长度为9，`Signature`占3个字节（24bit），`Version`占1个字节（8bit），`TypeFlagsReserved`（5bit）`TypeFlagsAudio`（1bit）、`TypeFlagsReserved`（1bit）、TypeFlagsVideo（1bit）一起占用1个字段、`文件头大小`（8bit）占用4个字段。

| 字段                | 类型   | 说明                             |
| ----------------- | ---- | ------------------------------ |
| Signature         | UI8  | ’F’(0x46)                      |
| Signature         | UI8  | ‘L’(0x4C)                      |
| Signature         | UI8  | ‘V’(0x56)                      |
| Version           | UI8  | FLV的版本。0x01表示FLV 版本是1，目前只有版本1。 |
| TypeFlagsReserved | UB5  | 前五位必须是0                        |
| TypeFlagsAudio    | UB1  | 音频流是否存在标志                      |
| TypeFlagsReserved | UB1  | 必须是0                           |
| TypeFlagsVideo    | UB1  | 视频流是否存在标志                      |
| DataOffset        | UI32 | FLV版本1时填写9，表明的是FLV头的大小         |

**FLV File Body字段说明**

`FLV File Body`是紧接着`FLV File Header`的。

| 字段                 | 类型     | 说明                                       |
| ------------------ | ------ | ---------------------------------------- |
| PreviousTagSize0   | UI32   | 总是 0                                     |
| Tag1               | FLVTAG | 第一个 tag                                  |
| PreviousTagSize1   | UI32   | 前一个 tag 的大小, 包括他的 header, 即: 11 + 前一个 tag 的大小 |
| Tag2               | FLVTAG | 第二个 tag                                  |
| …                  |        |                                          |
| PreviousTagSizeN-1 | UI32   | 前一个 tag 大小                               |
| TagN               | FLVTAG | 最后一个 tag                                 |
| PreviousTagSizeN   | UI32   | 最后一个 tag 大小, 包括他的 header                 |

## demux（容器解码）

源码请参考[src/demux/flv-demuxer.js](https://github.com/Bilibili/flv.js/blob/master/src/demux/flv-demuxer.js)。

**检测FLV文件头`签名`和`版本`字段信息**

```js
static probe(buffer) {
  let data = new Uint8Array(buffer);
  let mismatch = { match: false };
  if (
    //请参考上面的文件头字段说明。
    data[0] !== 0x46 ||
    data[1] !== 0x4c ||
    data[2] !== 0x56 ||
    data[3] !== 0x01
  ) {
    return mismatch;
  }
  //后面代码省略
}
```

**字段其实只是为了说明，可以按照数组的索引来理解，按照1个字节算一个数组来算，这个文件头总共9个字节，9个数组代表了9个字节的二进制数据（8bit）。**

### 校验

**检测是否有音、视频流**

假设音视频都有，`data[4]`的二进制为`00000101`，前5位和低7位为保留位。这里需要判断第6为和第8为是否为1。

```js
static probe(buffer) {
  //前面面代码省略
  let hasAudio = ((data[4] & 4) >>> 2) !== 0;
  //00000101 (data[4])
  //00000100 (4)
  //00000100 (&结果)
  //00000001 (右移两位的结果)
  //转成10进制为1
  let hasVideo = (data[4] & 1) !== 0;
  //00000101 (data[4])
  //00000001 (1)
  //00000001 (&结果)
  //转成10进制为1
  //后面代码省略
}
```

其实`hasAudio`可以这样判断

```js
let hasAudio = data[4] & 4 !== 0;
//这样可以确保这个字段的第6位是否为1。
//00000101 (data[4])
//00000100 (4)
//00000100 (&结果)
//转成10进制为4
```

### 解析

这个probe是被 parseChunks 调用的，当读取了至少13个字节后，就判断下是否是一个flv数据，然后再继续后面的分析。为什么是13，其中flv的文件头是9个字节，加上`The FLV File Body`的`PreviousTagSize0`字段信息（占4个字节）就是13个字节。`PreviousTagSize0`表示前一个tag的大小，但是由于第一个tag是不存在前一个的，所以第一个总是 0。

```js
// function parseChunks(chunk: ArrayBuffer, byteStart: number): number;
parseChunks(chunk, byteStart) {
  //前面代码省略
  if (byteStart === 0) {
    // buffer with FLV header
    if (chunk.byteLength > 13) {
      let probeData = FLVDemuxer.probe(chunk);
      offset = probeData.dataOffset;
    } else {
      return 0;
    }
  }
  //后面代码省略
}
```

parseChunks 后面的代码就是在不断解析 tag，flv把一段媒体数据称为 TAG，每个tag有不同的type，实际上真正用到的只有三种type，8、9、18 分别对应，音频、视频和Script Data。

**FLV Tag**

FLV Tag之前都有一个`PreviousTagSize`。tag的数据长度=11 +  4 + dataSize(整个音频数据或者视频数据或者ScriptData)，这里只有数据长度是不固定的 。

| 字段                | 类型     | 说明                                       |
| ----------------- | ------ | ---------------------------------------- |
| Reserved          | UB[2]  | 保留给FMS, 应为 0                             |
| Filter            | UB[1]  | 0 = unencrypted tags, 1 = encrypted tags |
| TagType           | UB [5] | 类型, 0x08 = audio, 0x09 = video, 0x12 = script data |
| DataSize          | UI24   | message 长度, 从 StreamID 到 tag 结束(len(tag) - 11) |
| Timestamp         | UI24   | 相对于第一个 tag 的时间戳(unit: ms), 第一个 tag 总是 0  |
| TimestampExtended | UI8    | Timestamp 的高 8 位. 扩展 Timestamp 为 SI32 类型 |
| StreamID          | UI24   | 总是 0, 至此为 11 字节                          |
| AudioTagHeader    |        | IF TagType == 0x08                       |
| VideoTagHeader    |        | IF TagType == 0x09                       |
| EncryptionHeader  |        | IF Filter == 1                           |
| FilterParams      |        | IF Filter == 1                           |
| Data              |        | AUDIODATA 或者 VIDEODATA 或者 SCRIPTDATA     |

下面的代码是过滤data数据不符合的情况。

```js
parseChunks(chunk,byteStart) {
  //前面代码省略
  let tagType = v.getUint8(0);
  let dataSize = v.getUint32(0, !le) & 0x00ffffff;
  //4为PreviousTagSizeN占用的字节数。
  if (offset + 11 + dataSize + 4 > chunk.byteLength) {
    // data not enough for parsing actual data body
    break;
  }
  if (tagType !== 8 && tagType !== 9 && tagType !== 18) {
    Log.w(this.TAG, `Unsupported tag type ${tagType}, skipped`);
    // consume the whole tag (skip it)
    offset += 11 + dataSize + 4;
    continue;
  }
  //后面代码省略
}
```

`offset`是用来计算当前arrayBuffer已经读取的位置，按字节来算。

`dataSize`的计算原理：

`0x00ffffff`的二进制值为32位，最高8位为0，其余为1，综合`&`的运算规则，可以知道结果的最高8位为0，剩余24位与左边操作数的低24位值相同。
于是`v.getUint32(0, !le) & 0x00ffffff`就是取`v.getUint32(0, !le)`的低24位，即低3字节的值，而这个值正是dataSize的值。

adobe为了节约流量，能用24bit表示的绝不用32bit，但是还是给timestamp设置了一个 扩展位存放最高位的字节，这个设计很蛋疼，于是导致了下面这段奇葩代码，先取三个字节按照big-endian转换成整数再在高位放上第四个字节。

```js
parseChunks(chunk,byteStart) {
  //前面代码省略
  let ts2 = v.getUint8(4);
  let ts1 = v.getUint8(5);
  let ts0 = v.getUint8(6);
  let ts3 = v.getUint8(7);
  let timestamp = ts0 | (ts1 << 8) | (ts2 << 16) | (ts3 << 24);//这里还没弄明白
  //后面代码省略
}
```

解析完了 tag header，后面分别按照不同的 tag type调用不同的解析函数。

```js
parseChunks(chunk,byteStart) {
  //前面代码省略
  switch (tagType) {
    case 8: // Audio
      this._parseAudioData(...);
      break;
    case 9: // Video
      this._parseVideoData(...);
      break;
    case 18: // ScriptDataObject
      this._parseScriptData(...);
      break;
  }
  //后面代码省略
}
```

#### 解析视频流

需要先了解Video Tags的结构。

**VideoTagHeader**总共占5字节。

| 字段              | 类型    | 说明                                       |
| --------------- | ----- | :--------------------------------------- |
| FrameType       | UB[4] | 1 = key frame, 2 = inter frame           |
| CodecID         | UB[4] | 7 = AVC                                  |
| AVCPacketType   | UI8   | IF CodecID == 7, <br />0 = AVC sequence header(AVCDecoderConfigurationRecord), <br />1 = One or more AVC NALUs (Full frames are required), <br />2 = AVC end of sequence |
| CompositionTime | SI24  | IF AVCPacketType == 1 Composition time offset ELSE 0 |

`_parseVideoData`会对Video Tag进行解析。

```js
_parseVideoData(...) {
  //前面代码省略
  //当前Video Tag的第一个字节，第一个字节包括了frameType和codecID。
  let spec = new Uint8Array(arrayBuffer, dataOffset, dataSize)[0];
  //取前四位的值
  let frameType = (spec & 240) >>> 4;
  //其实let frameType = spec >>> 4就可以了。
  //取后四位的值
  let codecId = spec & 15;
  //00010111(假设的spec二进制值)
  //00001111(值为15)
  //00000111(&运算后，取到spec后四位的值)
  if (codecId !== 7) {
    //这里只考虑AVC(H.264)的情况。
    this._onError(
      DemuxErrors.CODEC_UNSUPPORTED,
      `Flv: Unsupported codec in video frame: ${codecId}`
    );
    return;
  }
  this._parseAVCVideoPacket(...);
}
```

这里取两个重要的值，frameType表示帧类型 1 是关键帧 2 是非关键帧，codeId是编码类型。虽然flv支持 六种视频编码格式，但是实际上互联网点播直播真正在用的基本只有H.264一 种，**flv.js只考虑codecId为7的情况**，即AVC编码。

`_parseAVCVideoPacket`处理`VideoTagHeader`的`AVCPacketType`。

```js
_parseAVCVideoPacket(...) {
  //前面代码省略
}
```





## 参考

- [直播协议 HTTP-FLV 详解](http://akagi201.org/post/http-flv-explained/)
- [全面进阶 H5 直播](https://ivweb.io/topic/58de6dcb4813a86006c9052a)
- [理解字节序](http://www.ruanyifeng.com/blog/2016/11/byte-order.html)























