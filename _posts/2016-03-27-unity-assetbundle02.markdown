---
layout:     post
title:      "Unity5.x新的AssetBundle机制02——压缩"
date:       2016-03-27 09:29:00
---

## 前言

![unity_cover.png](/assets/images/in-post/unity_cover.png)

本节来讲讲Unity5新的AssetBundle机制中的压缩。

你可以不对你的AssetBundles进行压缩，这样虽然文件很大，但是一旦下载好之后访问是最快的。
通常都会进行压缩。并且Unity的AssetBundles支持LZMA和LZ4两种格式的压缩。

## LZMA（**Ziv–Markov chain algorithm**）格式。

这是Unity打包成AssetBundle时的默认格式，会将序列化数据压缩成一个LZMA流，使用时需要整体解包。优点是打包后体积小，缺点是解包时间长导致加载时间长。

## LZ4格式

相较LZMA会生成更大的压缩文件，但优点是使用时不需要整体解压。LZ4是一种基于chunk的算法，加载对象时只有相应的chunk会被解压。该格式需要unity5.3以上的版本，以前的版本并不支持。
如果你对使用不同的压缩格式和不同的加载API对内存开销及性能的影响感兴趣，可以参考这张表：

![compression_table.png](/assets/images/in-post/compression_table.png)

如果你不敢兴趣，那遵循以下规则好了：

1.将AssetBundles作为StreamAssets中的内容一同发布。
打包AssetBundles时使用BuildAssetBundleOptions.ChunkBasedCompression选项，加载资源时使用AssetBundle.LoadFromFileAsync接口。这样做可以达到数据压缩的效果又能使加载资源的效率如同读取缓存一般。

2.将AssetBundles作为DLCs下载。
打包AssetBundles时使用默认的压缩格式（LZMA），使用 LoadFromCacheOrDownload/WebRequest来下载和缓存。这样你将有最大的压缩率。

3.加密
打包AssetBundles时使用BuildAssetBundleOptions.ChunkBasedCompression选项，加载资源时使用LoadFromMemoryAsync接口（这可能是该接口及其同步版本唯一应该使用到的地方）。

4.自定义压缩
打包AssetBundles时使用BuildAssetBundleOptions.UncompressedAssetBundle选项，使用自己的算法压缩，加载资源时用相应算法解压后使用LoadFromFileAsync接口。

你应该尽量使用异步的接口。

---
参考链接：

http://docs.unity3d.com/Manual/AssetBundleCompression.html