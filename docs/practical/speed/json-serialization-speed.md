# 常用 JSON 库性能对比
反序列化是 Android 开发中必备且非常高频的需求，因此一个高性能的反序列化框架就非常重要了，那么在常用的 Gson, Moshi, Jackson, Kotlin Serialization, FastJson 等框架中，到底谁比较快?

## 测试方式
- 通过 Jetpack Microbenchmark 库进行基准测试，以避免 CPU 降频，JIT 优化对测试结果的影响
- 测试用例输入包括 12kb, 78kb, 238kb 大小的三个 json 文件，以测试 json 大小对反序列化速度的影响
- 测试结果分为多次运行充分预热与一次运行无预热两种情况，以测试在冷启动情况下反序列化速度的差异

### 测试代码
测试代码可见：[https://github.com/RicardoJiang/json-benchmark](https://github.com/RicardoJiang/json-benchmark)

## 测试结果
### 多次运行测试结果
|  |small json  |medium json  |large json  |
| --- | --- |--- |--- |
|  Kotlin Serialization| 165,936   ns | 997,228   ns | 2,933,098   ns |
|  JSONReader| 190,902   ns | 1,164,605   ns | 3,412,914   ns |
|  FastJson| 196,860   ns | 1,417,077   ns | 4,218,987   ns |
|  JSONObject| 258,789   ns | 1,690,190   ns | 4,788,937   ns |
|  Moshi| 303,056   ns | 1,411,364   ns | 3,955,789   ns |
|  Gson| 412,421   ns | 1,356,564   ns | 3,557,943   ns |
|  Jackson| 1,073,504   ns | 1,798,989   ns | 3,543,983   ns |

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/october/DeserializationSpeedMultiTimes.png)

从柱状图可以很明显的看出各个框架的速度对比

- Kotlin Serialization 看起来是最适合 Kotlin 的反序列化框架，在各个框架中表现最好
- JSONReader 与 JSONObject 在小数据上表现也不错，在大数据上 JSONReader 与其它框架相差不大，而 JSONObject 由于要将 InputStream 转化成 String，表现较差
- FastJson 在小数据时的确比较快，但是在大数据时相比其它框架没有什么优势，同时在没有无参构造函数时，FastJson 会使用 Kotlin 反射来反序列化，此时速度反序列化速度将大幅下降
- Moshi 与 Gson 在反序列化速度上差距不大，基本上是一个量级
- Jackson 在小数据反序化速度相比其它框架明显较慢，在大数据情况下则相差不大

### 一次运行测试结果
|  |small json  |medium json  |large json  |
| --- | --- |--- |--- |
|  Kotlin Serialization| 4,114,323   ns | 15,739,688   ns | 17,428,906   ns |
|  JSONReader| 630,469   ns | 2,052,501   ns | 5,630,261   ns |
|  FastJson| 61,629,844   ns | 6,756,823   ns | 10,529,791   ns |
|  JSONObject| 580,469   ns | 2,227,290   ns | 6,311,667   ns |
|  Moshi| 4,460,886   ns | 13,854,792   ns | 18,951,198   ns |
|  Gson| 3,319,688   ns | 5,568,906   ns | 10,264,635   ns |
|  Jackson| 15,070,469   ns | 13,625,521   ns | 17,914,687   ns |

![](https://raw.gitmirror.com/RicardoJiang/resource/main/2023/october/DeserializationSpeedRunOnce.png)

可以看出，一次运行测试结果与多次运行测试结果明显不同

- JSONReader 与 JSONObject 相比其它框架优势明显，在冷启动场景使用这些系统原生 API 应该会有一定优势，这也是[ MSON ](https://tech.meituan.com/2018/01/09/mson.html)框架提升反序列化速度的原理
- FastJson 在首次运行时显著慢于其它框架，后续则恢复正常速度，与 Gson 速度差不多，快于 Moshi
- 在冷启动场景，Moshi 与 Kotlin Serialization 速度差不多，相比 Gson 则略慢
- Jackson 首次运行时较慢，后续与 Moshi 速度类似

## 与 Protobuf 比较
Protobuf 全称：Protocol Buffers，是 Google 推出的一种与平台无关、语言无关、可扩展的轻便高效的序列化数据存储格式。

Protobuf 数据以二进制格式存储，相比 json 占用体积要小很多，比如我们将上面的 json 文件转化成 Protobuf 二进制文件之后，体积缩小了 50% 左右。

接下来我们来看下 Protobuf 反序列化的速度相比 json 到底怎么样。

### 多次运行测试结果
|  | small data | medium  data | large data |
| --- | --- |--- |--- |
|  Kotlin Serialization| 171,134   ns | 1,010,621   ns | 2,945,403   ns |
|  JSONReader | 191,611   ns | 1,165,174   ns | 3,426,443   ns |
|  Protobuf | 108,520   ns | 576,859   ns | 1,674,978   ns |

我们将 Protobuf 与之前表现比较优秀的 Kotlin Serialization 与 JSONReader 进行对比，可以看出 Protobuf 的反序列化速度相比之前表现最优秀的 Kotlin Serialization 也有 50% 左右的提升

### 一次运行测试结果
|  | small data | medium  data | large data |
| --- | --- |--- |--- |
|  Kotlin Serialization| 5,033,698   ns | 15,217,709   ns | 17,278,021   ns |
|  JSONReader | 519,167   ns | 1,617,604   ns | 5,623,749   ns |
|  Protobuf | 3,605,313   ns | 6,869,480   ns | 11,330,886   ns |

可以看出在冷启动时，JSONReader 的表现还是遥遥领先，而 Protobuf 的表现比 Kotlin Serialization 还是要好一些。

当然使用 JSONReader 与 JSONObject 这些原生 API 需要写很多模板代码，可以使用类似[ MSON ](https://tech.meituan.com/2018/01/09/mson.html)的方式，通过代码生成的方式减少开发成本。

## 总结
- Kotlin Serialization 是在纯 Kotlin 代码中反序列化 json 的最佳选择，速度最快
- 如果可以牺牲可读性，使用 Protobuf 可以带来 50% 左右的体积与反序列化速度提升
- 在冷启动场景中，使用 JSONReader 与 JSONObject 可以带来很大的性能提升。

当然性能测试的结果可能因为机型和数据的不同有所差异，欢迎使用测试代码进行测试，如有不同意见，欢迎在评论区交流~

### 测试代码
测试代码可见：[https://github.com/RicardoJiang/json-benchmark](https://github.com/RicardoJiang/json-benchmark)