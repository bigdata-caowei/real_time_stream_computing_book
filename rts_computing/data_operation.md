## 数据操作
当数据进入系统后，通常会对数据做一些整理，比如提取感兴趣字段、统一数据格式、过滤不合条件事件、合并不同来源数据流等等。
虽然不同系统处理数据的方式不尽相同，但经过多年的实践和积累，业界已逐渐形成一套有关流式API集合的共识。
在本节中，我们将讨论一些最基础的流数据操作方法。
这些操作方法是流式API的基础套件，几乎所有的流计算平台都会提供这些方法的实现。
而其它功能更丰富的流式API也会构建在这些方法的基础上。


### 过滤filter
过滤filter用于在数据流上筛选出符合指定条件的元素，并将筛选出的元素作为新的流输出。
流的过滤是一个容易理解，也容易实现的操作。
比如从来自全国各地的网页点击事件流里筛选出发生在湖北省的事件，实现如下：

```
streamFiltered = streamAll.filter(e -> "hubei".equals(e.get("province")))
```

上面的lambda表达式`e -> "hubei".equals(e.get("province")`即过滤条件。

### 映射map
映射map用于将数据流中的每个元素转化为新元素，并将新元素输出为数据流。
比如给原始事件流里每个事件增加事件发生时所在省份的信息，实现如下：

```
streamEnhanced = streamAll.map(e -> {
    e.put("province", queryProvinceByIp(e.getString("ip")));
    return e;
})
```
上面示意代码的lambda表达式中，通过原始事件的`IP`查询到相应省份后附加到事件上，最后返回附加了省份信息的事件。
这种对数据流的数据做信息增强的功能，正是映射操作的重要作用之一。

### 聚合reduce
聚合reduce用于将数据流中的元素按照指定方法进行聚合，并将聚合结果作为新的流输出。
由于流数据具有时间序列的特征，所以聚合操作不能像诸如Hadoop等批处理计算框架那样作用在整个数据集上。
流数据的聚合操作必然是指定了窗口（或者说这样做才有更加实际的意义），这些窗口可以基于时间、事件或会话（Session）等。
以时间窗口为例，假设我们要统计每分钟总的交易金额数，可以实现为：
```
streamAmount
    .timeWindow(Time.seconds(60), Time.seconds(1))
    .reduce((amount1, amount2) -> amount1 + amount2);
```
上面的代码片段中，`streamAmount`是由交易金额构成的数据流，
我们使用`timeWindow`指定了每隔1秒钟，对60秒钟窗口内的数据进行一次计算。
而`reduce`方法的输入是一个用于求和的lambda表达式。
在实际执行时，这个求和lambda表达式会依次将每条数据与前一次计算的结果相加，最终完成对窗口内全部流数据的求和计算。
如果将求和操作换成其它"二合一"的计算，则可以实现对应功能的聚合运算。
由于使用了窗口，所以聚合后流的输出不再是像map运算那样逐元素地输出，
而是每隔一段时间才会输出窗口内的聚合运算结果。比如前面的实例代码中，就是每隔1秒钟输出60秒钟窗口内的聚合计算结果。


### 关联Join
关联Join用于将两个数据流中满足特定条件的元素对组合起来，按指定规则形成新元素，并将新元素输出为数据流。
在关系型数据库中，关联操作是非常常用的查询手段，这是由关系型数据库的设计理念所决定。
而在流数据领域，由于数据来源的多样性和在时序上的差异性，数据流之间的关联也成为一种非常自然的需求。
以常见场景为例，假设我们收集的事件流，同时被输入两个功能不同的子系统以做处理，它们各自处理的结果同样以数据流的方式输出。
现在需要将这两个子系统的输出流，按照相同事件id合并起来，以汇总两个子系统对同一事件的的处理结果。
以下就是实现这个功能的示意代码。

```
streamAction1
    .join(streamAction2)
    .where(e1 -> e1.get("event_id))
    .equalTo(e2 -> e2.get("event_id))
    .apply((e1, e2) -> (e1 + e2))
```

如果从关系型数据库表间join操作来看，上面的示意代码确实实现了两个流的关联功能。
但是流数据的关联在语义和实现上都会更加复杂些。
由于流的无限性，只有在"一对一映射"等非常受限的使用场景下，这种不限时间窗口的关联设计和实现才有意义。
大多数使用场景下，我们需要引入"窗口"来对关联的流数据进行时间同步。以下为引入"窗口"后的流关联实现。

```
streamAction1
    .join(streamAction2)
    .where(e1 -> e1.get("event_id))
    .equalTo(e2 -> e2.get("event_id))
    .window(TimeWindows.of(Time.seconds(60), Time.seconds(1)))
    .apply((e1, e2) -> (e1 + e2))
```

流的关联是一个我们经常想用，但又容易让人头疼的操作。因为稍不注意，关联操作的性能就会惨不忍睹。
关联操作需要保存大量的状态，尤其是窗口越长，需要保存的数据越多。因此当使用流数据的关联功能时，应尽可能让窗口较短。
做个简单计算，假设N和M分别是两个流在1分钟内的平均事件数。
那么当窗口为5分钟时，关联操作的计算量是`5N * 5M`。
而如果窗口为1分钟，那么同样的数据就只有`5 * N * M`的计算量。

总的来说，如果两个流中属于同一实体的不同元素产生的时间差最大为`T`。
那么为了将这两个流中属于同一实体的元素关联起来，需要设置窗口长度为`T`，并配置一个较小的滑动步长。
比如为了找出点击事件流和激活事件流中，属于同一设备并且发生时间间隔在1分钟内的"点击-激活"事件对，可以设置时间窗口为60秒，滑动步长为1秒。
这样可以保证窗口较小的同时，尽可能无遗漏地找到匹配的事件对。
但这样做同时造成了计算严重重复的问题，毕竟每个滑动步长（1秒）内的数据前前后后总共要被计算60次。
所以对于流的关联，在设计和使用时，一定要谨慎考量资源使用（主要是内存）、计算效率、关联窗口、滑动步长和业务精度及性能要求，
在各个考量因素间作出取舍和权衡。


### 遍历foreach
遍历foreach是对数据流的每个元素执行指定方法的过程。
遍历与映射非常相似又非常不同。

说相似是因为foreach和map都是将一个表达式作用在数据流上，只不过foreach使用是"方法"（没有返回值的函数），而map使用的是"函数"。

说不同是因为foreach和map语义迥异，在具体实现时也很不相同。
从API语义上来讲，map作用是对流进行转换，不应该对系统产生副作用。
但foreach就不同了，它会处理掉流，至于怎样处理则不做任何限定，流在经过foreach后也就终结了。
所以foreach可以对系统，特别是外部IO或存储产生副作用。

当然，我们可以不顾一切地用map实现foreach（毕竟"方法"也是"函数"），但终归不太纯粹和优雅。

下面的示例代码实现了将事件流中的事件保存到MongoDB的功能。

```
streamAll.foreach(e -> {
    saveToMongo(e)
})
```


### 总结
本节讲解了流式API中几种最基础的操作。除了这些基础操作外，还有很多功能丰富流式API，比如flatMap、reduceByKey等等。
虽然本节没有详细讲解它们，但在开发流式应用时十有八九会用到它们。
所有读者还是有必要对它们有所了解，具体可以查阅Apache Flink等开源流计算框架的API文档，在此就不一一展开了。
