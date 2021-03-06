# Broken RosyWriter(YUV420)

AppleのサンプルコードRosyWriter（とGLCameraRipple）を参考に、ビデオのデータ形式を`kCVPixelFormatType_32BGRA`から`kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange`に変更し、さらに毎フレーム呼び出されるデリゲートメソッド`captureOutput:didOutputSampleBuffer:fromConnection:`にて、サンプルバッファをコピーしようとしているプログラムです。

しかし、現状ではコピーしたサンプルバッファを使おうとしても、画面表示・ビデオ録画のどちらにも使えない、壊れたサンプルバッファしか作れていません。

__(2012/4/10 23:55 @whitedev氏と@norio_nomura氏のご協力により原因判明)__

## 目的
- デリゲートメソッド`captureOutput:didOutputSampleBuffer:fromConnection:`に届くYUV420のサンプルバッファを、__正しいデータとして__コピーすること。
- イメージバッファのコピーは「浅いコピー」ではなく「深いコピー」をすること

上記の処理を、`VideoProcessor.m`の中の`deepCopySampleBuffer:`メソッドで行おうとしています。

### 期待する結果
- 画面に表示した際、きちんと色が付いて見えること
- AVAssetWriterオブジェクトに`appendSampleBuffer:`メソッドでサンプルバッファを書き込むと、ムービーの１フレームとして保存してくれること

![expected result](https://github.com/katokichisoft/Broken-RosyWriter-YUV420/raw/master/expect.png)

### 実際の結果
- 画面に表示すると、緑色に覆われる
- AVAssetWriterオブジェクトに`appendSampleBuffer:`メソッドで書き込むと、不正なイメージデータだとしてエラーが返る

![actual result](https://github.com/katokichisoft/Broken-RosyWriter-YUV420/raw/master/result.png)

## なぜコピーしようとしているか
一般的な使い方だと、デリゲートメソッドに届くサンプルバッファは、描画に使ったり録画するなどして消費します。ですが、これをあえて保持しておくことで、複数フレームからデータを解析／加工することができないかと考えました。

取りあえず到着するサンプルバッファをローカルのバッファに溜めようとしたところ、13フレームぶんのサンプルバッファが到達したところで、デリゲートメソッドが呼ばれなくなってしまいました。どうやらフレームワーク側の処理で、各サンプルバッファが持つピクセルデータ領域に対し、バッファプールから取り出したデータを使い回しているようです。そのために、保持したまま消費しないとプールが枯渇してしまい、デリゲートメソッドが呼べなくなっていることが分かりました。

そこで、到着するサンプルバッファを、ピクセルバッファごとdeep copyしてしまうことで、フレームワークの資源を切り崩さないよう実装することにしました。

## なぜ浅いコピーでは駄目なのか（サンプルバッファのコピーAPIが使えない理由）
サンプルバッファのクラス`CMSampleBufferRef`にはコピー用のAPIとして、`CMSampleBufferCreateCopy()`が用意されています。
しかしこのAPIでは、サンップルバッファ内のイメージバッファ、つまりバッファプールから取り出したイメージバッファのretain countを増やすだけの、「浅いコピー」しか行いません。
この状態でコピーしたサンプルバッファを溜めても、バッファプールが枯渇することには変わりないのです。

===========
# (2012/4/11追記)原因と対策
## 調査により判明した原因
`CVPixelBufferRef`オブジェクトを作成するときに指定するPixelFormatDescriptionについて、__kCVPixelBufferIOSurfacePropertiesKey__キーと適当な辞書を属性として設定すれば良いことが分かりました。

## コード
ピクセルバッファあるいはピクセルバッファプールを作成するときの属性として、`kCVPixelBufferIOSurfacePropertiesKey`と辞書を設定します。

```objc
NSDictionary *mAttrs = [NSDictionary dictionaryWithObject:[NSDictionary dictionary]
                                                   forKey:(id)kCVPixelBufferIOSurfacePropertiesKey];

CVPixelBufferCreate(kCFAllocatorDefault,
                    bufferWidth, bufferHeight, pixelFormatType,
                    (CFDictionaryRef)mAttrs,
                    &copiedPixelBuffer);
```
```objc
NSMutableDictionary *mAttrs = [NSMutableDictionary dictionary];

[mAttrs setObject:[NSNumber numberWithInt:kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange]
           forKey:(NSString*)kCVPixelBufferPixelFormatTypeKey];
[mAttrs setObject:[NSNumber numberWithInt:640] forKey:(NSString*)kCVPixelBufferWidthKey];
[mAttrs setObject:[NSNumber numberWithInt:480] forKey:(NSString*)kCVPixelBufferHeightKey];

[mAttrs setObject:[NSDictionary dictionary] forKey:(NSString*)kCVPixelBufferIOSurfacePropertiesKey];
 
CVPixelBufferPoolCreate(kCFAllocatorDefault,
                        NULL,
                        (CFDictionaryRef)mAttrs,
                        &pool);
```

------
See [Technical Note TN2267 : Video Decode Acceleration Framework Reference](http://developer.apple.com/library/mac/#technotes/tn2267/_index.html)
