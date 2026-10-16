# 1、Doc Values

倒排索引在搜索包含指定term的doc时非常高效，但是在相反的操作时表现很差：查询一个文档中包含哪些term。具体来说，倒排索引在搜索时最为高效，但在排序、聚合等与指定filed相关的操作时效率低下，需要用**doc_values**。

倒排索引将term映射到包含它们的doc，而doc values将doc映射到它们包含的所有词项，下面是一个示例：

```java
Doc      Terms
-----------------------------------------------------------------
Doc_1 | brown, dog, fox, jumped, lazy, over, quick, the
Doc_2 | brown, dogs, foxes, in, lazy, leap, over, quick, summer
Doc_3 | dog, dogs, fox, jumped, over, quick, the
-----------------------------------------------------------------
```

当数据被逆置之后，想要收集到 Doc_1 和 Doc_2 的唯一 token 会非常容易。获得每个文档行，获取所有的词项，然后求两个集合的并集。

其实，Doc Values本质上是一个序列化了的列式存储结构，非常适合排序、聚合以及字段相关的脚本操作。而且这种存储方式便于压缩，尤其是数字类型。压缩后能够大大减少磁盘空间，提升访问速度。下面是一个数字类型的 Doc Values示例：

```java
Doc      Terms
-----------------------------------------------------------------
Doc_1 | 100
Doc_2 | 1000
Doc_3 | 1500
Doc_4 | 1200
Doc_5 | 300
Doc_6 | 1900
Doc_7 | 4200
-----------------------------------------------------------------
```

列式存储意味着有一个连续的数据块： [100,1000,1500,1200,300,1900,4200] 。因为我们已经知道他们都是数字（而不是像文档或行中看到的异构集合），所以可以使用统一的偏移量来将他们紧紧排列。

而且，针对这样的数字有很多种压缩技巧。你会注意到这里每个数字都是 100 的倍数，Doc Values会检测一个段里面的所有数值，并使用一个最大公约数，方便做进一步的数据压缩。

比如，这个例子中可以用100作为公约数，那么以上数字就变为[1,10,15,12,3,19,42]，可用很少的bit就能存储，节约了磁盘空间。一般来说，Doc Values按顺序来检测以下压缩方案：

- 如果所有值都相同（或缺失），就设置一个标志并记录该值
- 如果少于256个值，就会使用一个简易码表
- 如果值个数大于256，就检查是否存在最大公约数
- 如果没有最大公约数，就以偏移量的方式从最小值开始对所有值编码

String类型使用顺序表，按和数字类型类似的方式编码。String类型去重后排序，然后写入一个表中，并分配一个ID号，然后这些ID号就被当做数字类型的Doc Values。这意味着字符串享有许多与数字相同的压缩特点。

**Doc Values是在字段索引时与倒排索引同时生成，而且生成以后是不可变的。**

Doc Value 默认对除了`analyzed String`外的所有字段启用（因为分词后会生成很多token使得Doc Values效率降低）。但是当你知道某些字段永远不会进行排序、聚合以及脚本操作的时候可以禁用Doc Values以节约磁盘空间提升索引速度，示例如下：

```json
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "session_id": {
          "type":       "string",
          "index":      "not_analyzed",
          "doc_values": false 
        }
      }
    }
  }
}
```

以上配置以后，session_id字段就只能被搜索，不能被用于排序、聚合以及脚本操作了。

还可以通过设定doc_values为true，index为no来让字段不能被搜索但可以用于排序、聚合以及脚本操作：

```json
PUT my_index
{
  "mappings": {
    "my_type": {
      "properties": {
        "customer_token": {
          "type":       "string",
          "index":      "not_analyzed",
          "doc_values": true, 
          "index": "no" 
        }
      }
    }
  }
}
```

Doc Value的特点就是快速、高效、内存友好，使用由linux kernel管理的文件系统缓存弹性存储。doc values在排序、聚合或与字段相关的脚本计算得到了高效的运用，任何需要查找某个文档包含的值的操作都必须使用它。如果你确定某个filed不会做字段相关操作，可以直接关掉doc_values，节约内存，加快访问速度。

# 2、Fielddata

上文说过，在排序、聚合以及在脚本中访问field值时需要一个与倒排索引截然不同的数据访问模式：不同于倒排索引中的查找term->找到对应docs的过程，我们需要直接查找doc然后找到指定某个filed中包含的terms。

大多数field使用索引时、磁盘上的doc_values来支持这种访问模式，但是分词了的String filed不支持Doc Values，而是使用一种叫FieldData的数据结构。

**FieldData主要是针对analyzed String**，它是一种查询时（query-time）的数据结构。

FieldData缓存主要应用场景是在对某一个field排序或者计算类的聚合运算时。它会把这个field列的所有值加载到内存，这样做的目的是提供对这些值的快速文档访问。为field构建FieldData缓存可能会很昂贵，因此建议有足够的内存来分配它，并保持其处于已加载状态。

**FieldData是在第一次将该filed用于聚合，排序或在脚本中访问时按需构建**。FieldData是通过从磁盘读取每个段来读取整个反向索引，然后逆置term->doc的关系，并将结果存储在JVM堆中构建的。

所以，加载FieldData是开销很大的操作，一旦它被加载后，就会在整个段的生命周期中保留在内存中。

这了可以注意下FieldData和Doc Values的区别。较早的版本中，其他数据类型也是用的FieldData，但是目前已经用随文档索引时创建的Doc Values所替代。

JVM堆内存资源是非常宝贵的，能用好它对系统的高效稳定运行至关重要。FieldData是直接放在堆内的，所以必须合理设定用于存放它的堆内存资源数。ES中控制FieldData内存使用的参数是`indices.fielddata.cache.size`，可以用x%表示占该节点堆内存百分比，也可以用如12GB这样的数值。默认状况下，这个设置是无限制的，ES不会从FieldData中驱逐数据。如果生成的fielddata大小超过指定的size，则将驱逐其他值以腾出空间。使用时一定要注意，这个设置只是一个安全策略而并非内存不足的解决方案。因为通过此配置触发数据驱逐，ES会立刻开始从磁盘加载数据，并把其他数据驱逐以保证有足够空间，导致很高的IO以及大量的需要被垃圾回收的内存垃圾。

举个例子来说：每天为日志文件建一个新的索引。一般来说我们只对最近几天数据感兴趣，很少查询老数据。但是，按默认设置FieldData中的老索引数据是不会被驱逐的。这样的话，FieldData就会一直持续增长直到触发**熔断机制**，这个机制会让你再也不能加载更多的FieldData到内存。这样的场景下，你只能对老的索引访问FieldData，但不能加载更多新数据。所以，这个时候就可以通过以上配置来把最近最少使用的FieldData驱逐以够新进来的数据腾空间。

FieldData是在数据被加载后再检查的，那么如果一个查询导致尝试加载超过可用内存的数据就会导致OOM异常。ES中使用了*FieldData Circuit Breaker*来处理上述问题，他可以通过分析一个查询涉及到的字段的类型、基数、大小等来评估所需内存。如果估计的查询大小大于配置的堆内存使用百分比限制，则断路器会跳闸，查询将被中止并返回异常。

断路器是工作是在数据加载前，所以你不用担心遇到FieldData导致的OOM异常。ES拥有多种类型的断路器：

- `indices.breaker.fielddata.limit`
- `indices.breaker.request.limit`
- `indices.breaker.total.limit`

可以根据实际需要进行配置。

FieldData是为分词String而生，它会消耗大量的java 堆空间，特别是加载基数（cardinality）很大的分词String filed时。但是往往对这种类型的分词Field做聚合是没有意义的。

值得注意的是，FieldData和Doc Values的加载时机不同，前者是首次查询时，后者是doc索引时。还有一点，FieldData是按每个段来缓存的。

# 3、Doc values与Fielddata对比

doc_values与fielddata一个很显著的区别是，前者的工作地盘主要在磁盘，而后者的工作地盘在内存。

| 维度     | doc_values     | fielddata                                  |
| -------- | -------------- | ------------------------------------------ |
| 创建时间 | index时创建    | 使用时动态创建                             |
| 创建位置 | 磁盘           | 内存(jvm heap)                             |
| 优点     | 不占用内存空间 | 不占用磁盘空间                             |
| 缺点     | 索引速度稍低   | 文档很多时，动态创建开销比较大，而且占内存 |

索引速度稍低这个是相对于fielddata方案的，其实仔细想想也可以理解。拿排序举例，相对于一个在磁盘排序，一个在内存排序，谁的速度快不言自明。

在ES 1.x版本的官方说法是，

> Doc values are now only about 10–25% slower than in-memory fielddata

虽然速度稍慢，doc_values的优势还是非常明显的。一个很显著的点就是它不会随着文档的增多引起OOM问题。正如前面说的，doc_values在磁盘创建排序和聚合所需的正排索引。这样我们就避免了在生产环境给ES设置一个很大的`HEAP_SIZE`，也使得JVM的GC更加高效，这个又为其它的操作带来了间接的好处。

而且，随着ES版本的升级，对于doc_values的优化越来越好，索引的速度已经很接近fielddata了，而且我们知道硬盘的访问速度也是越来越快（比如SSD）。所以 doc_values 现在可以满足大部分场景，也是ES官方重点维护的对象。

所以我想说的是，doc values相比field data还是有很多优势的。所以 ES2.x 之后，支持聚合的字段属性默认都使用doc_values，而不是fielddata。

# 4、Global Ordinals 全局序号

Global Ordinals是一个在Doc Values和FieldData之上的数据结构，它为每个唯一的term按字典序维护了一个自增的数字序列。每个term都有自己的一个唯一数字，而且字母A的全局序号小于字母B。特别注意，全局序号只支持String类型的field。

请注意，Doc Values和FieldData也有自己的ordinals序号，这个序号是特定segment和field中的唯一编号。通过提供Segment Ordinals和Global Ordinals间的映射关系，全局序号只是在此基础上创建，后者（即全局序号）是在整个shard分片中是唯一的。

一个特定字段的Global Ordinals跟一个分片中的所有段相关，而Doc Values和FieldData的ordinals只跟单个段相关。因此，只要是一个新段要变得可见，那么就必须完全重建全局序号。

也就是说，跟FieldData一样，在默认情况下全局序号也是懒加载的，会在第一个请求FieldData命中一个索引时来构建全局序号。实际上，在为每个段加载FieldData后，ES就会创建一个称为Global Ordinals（全局序号）的数据结构来构建一个由分片内的所有段中的唯一term组成的列表。

全局序号的内存开销小的原因是它由非常高效的压缩机制。提前加载的全局序号可以将加载时间从第一次搜索时转到全局序号刷新时。

全局序号的加载时间依赖于一个字段中的term数量，但是总的来说耗时较低，因为来源的字段数据都已经加载到内存了。

全局序号在用到段序号的时候很有用，比如排序或者terms aggregation，可以提升执行效率。

我们举个简单的例子。比如有十亿级别的doc，每个doc都有一个status字段，但只有pending, published, deleted三个状态数据。如果直接存整个String数据到内存，那么就算每个doc有15字节，那么一共就是差不多14GB的数据。怎么减少占用空间呢？首先想到的就是用数字来进行编码，码表如下：

```java
Ordinal | Term
-------------------
0       | status_deleted
1       | status_pending
2       | status_published
```

这样的话，初始的那三个String就只在码表内被存了一次。FieldData中的doc就可以直接用编码来指向实际值：

```java
Doc     | Ordinal
-------------------------
0       | 1  # pending
1       | 1  # pending
2       | 2  # published
3       | 0  # deleted
```

这样编码以后，直接把数据量压缩了十倍左右。但有个问题是FieldData是按每个段来分别加载、缓存的。那么就会出现一个情况，如果一个段内的doc只有deleted和published两个状态，那么就会导致该FieldData算出来的码表只有0和1，这就和拥有3个状态的段算出的FieldData码表不同。这样的话，聚合的时候就必须一个段一个段的计算，最后再聚合，十分缓慢，开销巨大。

ES的做法是用Global Ordinals这种构建在FieldData之上的小巧数据结构，编码会结合所有段来计算唯一值然后存放为一个序号码表。这样一来，term aggregation可以只在全局序号上进行聚合，而且只会在聚合的最终阶段来计算从序号到真实的String值一次。这个机制可以提升聚合的性能3-4倍。