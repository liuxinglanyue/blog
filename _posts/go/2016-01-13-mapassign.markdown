---
layout: post
category: "go"
title:  "疑似Golang下hashmap内存泄露"
tags: [mapassign,hashmap,go,reflect]
---

我们采用的是Open-Falcon来监控几百台机器，同时添加了探针的功能。等于说，做了定制化开发。

在现网机器上，我们发现Judge、Graph消耗的内存都非常高，2G左右。当前量级的情况下，这有些不合理。

这时我们使用了Go提供的分析工具pprof，使用下来发现真心不错，推荐。

```
go tool pprof http://127.0.0.1:6071/debug/pprof/heap

(pprof) top -20
1164.54MB of 1169.95MB total (99.54%)
Dropped 68 nodes (cum <= 5.85MB)
Showing top 10 nodes out of 42 (cum >= 711.16MB)
      flat  flat%   sum%        cum   cum%
  576.16MB 49.25% 49.25%   576.16MB 49.25%  reflect.mapassign
  164.01MB 14.02% 63.26%   164.01MB 14.02%  reflect.unsafe_New
  157.50MB 13.46% 76.73%   157.50MB 13.46%  encoding/gob.decString
  136.36MB 11.66% 88.38%   136.36MB 11.66%  reflect.Value.call
     108MB  9.23% 97.61%      108MB  9.23%  reflect.makemap
      16MB  1.37% 98.98%       16MB  1.37%  encoding/json.(*Decoder).refill
    6.50MB  0.56% 99.54%     6.50MB  0.56%  encoding/json.(*decodeState).literalStore
         0     0% 99.54%   999.68MB 85.45%  encoding/gob.(*Decoder).Decode
         0     0% 99.54%   999.68MB 85.45%  encoding/gob.(*Decoder).DecodeValue
         0     0% 99.54%   711.16MB 60.79%  encoding/gob.(*Decoder).decOpFor.func2

```
发现reflect.mapassign消耗了49.25%的内存，继续输入命令web后，打开浏览器，我们会得到一张svg图，一目了然。

[pprof001.svg](http://javagoo.com/img/go/pprof001.svg)

仔细分析后发现，mapassign是rpc decodeMap导致的，然后查看相关代码，的确需要反序列化map。

到这里，问题定位了。怎么解决呢？

第一，我想到了，golang是不是修复了这个bug呢。

[reflect: mark mapassign as noescape](https://github.com/golang/go/commit/8d31a86a1e7be5f84af9df8aeb36bc1e157d50eb)

```
The lack of this annotation causes Value.SetMapIndex to allocate
when it doesn't need to.

Add comments about why it's safe to do so.

Add a test to make sure we stay allocation-free.
```

这是在golang 1.6beta1中修复的。然后我使用此版本重新编译了Judge，在现网跑了一段时间后，发现 然并卵。

也Google了，没找到相似问题。此路不通。

继续：

既然decodeMap消耗内存严重，我们就放弃使用map。Transfer rpc之前，先将map转成string，然后Judge再将string转成map。

```
go tool pprof http://127.0.0.1:6071/debug/pprof/heap

(pprof) top -20
549.52MB of 557.53MB total (98.56%)
Dropped 95 nodes (cum <= 2.79MB)
Showing top 10 nodes out of 37 (cum >= 103MB)
      flat  flat%   sum%        cum   cum%
  423.61MB 75.98% 75.98%   423.61MB 75.98%  reflect.Value.call
     103MB 18.47% 94.45%      103MB 18.47%  encoding/gob.decString
       8MB  1.43% 95.89%        8MB  1.43%  encoding/json.(*Decoder).refill
    5.50MB  0.99% 96.88%     5.50MB  0.99%  encoding/json.(*decodeState).literalStore
       5MB   0.9% 97.77%        5MB   0.9%  reflect.mapassign
    4.41MB  0.79% 98.56%     4.41MB  0.79%  hawkeye/judge/cron.rebuildStrategyMap
         0     0% 98.56%   103.51MB 18.57%  encoding/gob.(*Decoder).Decode
         0     0% 98.56%   103.51MB 18.57%  encoding/gob.(*Decoder).DecodeValue
         0     0% 98.56%      103MB 18.47%  encoding/gob.(*Decoder).decOpFor.func3
         0     0% 98.56%      103MB 18.47%  encoding/gob.(*Decoder).decOpFor.func4
```

改动后，reflect.mapassign只消耗了5M内存。svg图也贴下

[pprof002.svg](http://javagoo.com/img/go/pprof002.svg)

Graph同样修改下，内存消耗从1.3G降到了20M。

新的问题【待解】：从pprof002.svg中可以发现一个新的内存大户reflect.Value.call， 由于Judge需要定时从hbs拉取Hbs.GetStrategies、Hbs.GetExpressions信息。