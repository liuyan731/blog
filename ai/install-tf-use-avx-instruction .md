---
title: 安装Tensorflow使用AVX指令集
date: 2018-04-22 22:44:00
categories: Machine Learning
tags: [machine learning,tensorflow]

---
*Blog的Github地址：[https://github.com/liuyan731/blog](https://github.com/liuyan731/blog)*

***
最近上线了两个机器学习Python服务，但是在线上的性能并不是特别好，而且CPU的负载非常高。然后发现是安装的Tensorflow未使用AVX指令集导致的。今天分享一下调试方法和解决方案。

## 未使用AVX指令集warning

最近发现线上Tensorflow python服务CPU消耗过高，服务性能也非常差，然后分析原因，测试服务器上使用timeline工具查看预测操作的耗时如下图：

![timelime-img-1](https://raw.githubusercontent.com/liuyan731/tuchuang/master/blog/tf-avx-1.png)

发现整个计算过程消耗时间过长（30ms）对比本机（使用GPU加速，服务器机器只用于提供服务，无GPU）耗时差别巨大（1ms，如下图）

![timelime-img-2](https://raw.githubusercontent.com/liuyan731/tuchuang/master/blog/tf-avx-2.png)

同时发现两个timeline上面的op的名称不一样，在测试服务器上操作为：_MklConv2DWithBias、_MklRelu等，在本机上为Conv2D、Relu等。

这时想到在运行Tensorflow的时候，import会warning提示没有使用intel avx、sse指令集：
> Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX AVX2

> The TensorFlow library wasn't compiled to use SSE4.1 instructions, but these are available on your machine and could speed up CPU computations.

> The TensorFlow library wasn't compiled to use SSE4.2 instructions, but these are available on your machine and could speed up CPU computations.

> The TensorFlow library wasn't compiled to use AVX instructions, but these are available on your machine and could speed up CPU computations.

因为这个只是warning并不影响程序的运行，所以一直没有在意，同时当时在网上查找解决方案，有很多“建议”让不管或直接把这个warning关掉...

考虑到这个AVX/SSE指令集可能会提高性能，所以着手重新编译安装Tensorflow，编译安装可以参考：[https://stackoverflow.com/questions/41293077/how-to-compile-tensorflow-with-sse4-2-and-avx-instructions](https://stackoverflow.com/questions/41293077/how-to-compile-tensorflow-with-sse4-2-and-avx-instructions) ，过程还是有点复杂，需要使用bazel。

但是我并没有真正去编译安装orz，因为发现我们使用的Tensorflow 1.6.0 已经开始使用AVX指令集进行预编译了，如下图：

![tf-release](https://raw.githubusercontent.com/liuyan731/tuchuang/master/blog/tf-avx-4.png)

但是为何还是会有warning警告呢？

## pip与conda安装的Tensorflow不一致？

与小组的另一位同学讨论，发现他使用的tensorflow1.6.0并不会有avx warning，细问之下发现他是使用pip（阿里云）安装的，而我是使用conda（清华镜像）安装的。

考虑到conda镜像和pip镜像的可能不一致，将conda安装的Tensorflow卸载，重新使用pip安装，avx warning消失。

重新运行服务并打印timeline，如下图：

![timelime-img-3](https://raw.githubusercontent.com/liuyan731/tuchuang/master/blog/tf-avx-3.png)

整体时间下降到2ms，性能大大提升，同时CPU负载大大降低！至此性能优化到合理的水平。

所以至少对于Tensorflow 1.6.0版本pip安装与conda安装是不一致的。

ps

- 使用pip安装完Tensorflow后，重新import tensorflow报错：ImportError: /usr/lib64/libstdc++.so.6: version CXXABI_1.3.7’ not found 的[解决方案](http://libowei.net/ImportError-usr-lib64-libstdc-so-6-version-CXXABI-1-3-7%E2%80%99-not-found.html)
- [AVX](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions)：
Advanced Vector Extensions(AVX) are extensions to the x86 instruction set architecture for microprocessors from Intel and AMD proposed by Intel in March 2008 and first supported by Intel with the Sandy Bridge processor shipping in Q1 2011 and later on by AMD with the Bulldozer processor shipping in Q3 2011. AVX provides new features, new instructions and a new coding scheme.
- [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)：In computing, Streaming SIMD Extensions (SSE) is an SIMD instruction set extension to the x86 architecture, designed by Intel and introduced in 1999 in their Pentium III series of processors shortly after the appearance of AMD's 3DNow!. SSE contains 70 new instructions, most of which work on single precision floating point data. SIMD instructions can greatly increase performance when exactly the same operations are to be performed on multiple data objects. Typical applications are digital signal processing and graphics processing.


## 补充
### TF Timeline模块
参考：[Tensorflow Timeline介绍及简单使用](https://liuyan731.github.io/2018/04/22/tf-timeline-tool/)

### pip使用国内镜像

```shell
mkdir .pip
vi .pip/pip.conf
 
[list]
format=columns
 
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/
 
[install]
trusted-host=mirrors.aliyun.com
```

## 思考
程序中的warning也是需要引起大家足够的重视，说不定里面就有一个大坑。。。

*2018/4/22 done*

*此文章也同步至[个人Github博客](https://liuyan731.github.io/)*