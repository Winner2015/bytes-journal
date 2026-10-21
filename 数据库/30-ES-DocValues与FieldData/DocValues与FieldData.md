# Elasticsearch DocValues 与 FieldData

# 1 Doc Values

倒排索引在搜索包含指定 term 的 doc 时非常高效，但在相反的操作上表现很差：查询一个文档中包含哪些 term。具体来说，倒排索引在搜索时最为高效，但在排序、聚合等与指定 field 相关的操作上效率低下，这时就需要用到 **Doc Values**。

倒排索引将 term 映射到包含它们的 doc，而 Doc Values 将 doc 映射到它们包含的所有词项，下面是一个示例：

```text
Doc      Terms
-----------------------------------------------------------------
Doc_1 | brown, dog, fox, jumped, lazy, over, quick, the
Doc_2 | brown, dogs, foxes, in, lazy, leap, over, quick, summer
Doc_3 | dog, dogs, fox, jumped, over, quick, the
-----------------------------------------------------------------
```

当数据被逆置之后，想要收集 Doc_1 和 Doc_2 中的唯一 token 就会非常容易：获取每个文档行中的所有词项，然后求两个集合的并集即可。

其实，Doc Values 本质上是一个序列化了的列式存储结构，非常适合排序、聚合以及字段相关的脚本操作。而且这种存储方式便于压缩，尤其是数字类型，压缩后能够大大减少磁盘空间、提升访问速度。下面是一个数字类型的 Doc Values 示例：

```text
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

列式存储意味着有一个连续的数据块：`[100, 1000, 1500, 1200, 300, 1900, 4200]`。因为我们已经知道它们都是数字（而不是像文档或行中看到的异构集合），所以可以使用统一的偏移量将它们紧密排列。

而且，针对这样的数字有很多种压缩技巧。你会注意到这里每个数字都是 100 的倍数，Doc Values 会检测一个段里面的所有数值，并使用最大公约数来进一步压缩数据。

比如，这个例子中可以用 100 作为公约数，那么以上数字就变为 `[1, 10, 15, 12, 3, 19, 42]`，用很少的 bit 就能存储，节约了磁盘空间。一般来说，Doc Values 会按顺序检测以下压缩方案：

- 如果所有值都相同（或缺失），就设置一个标志并记录该值
- 如果少于 256 个值，就使用一个简易码表
- 如果值的个数大于 256，就检查是否存在最大公约数
- 如果没有最大公约数，就以偏移量的方式从最小值开始对所有值编码

字符串类型（如 `keyword`）使用序号表，按与数字类型类似的方式编码：字符串去重后排序，写入一个表中并分配一个 ID 号，然后这些 ID 号就被当作数字类型的 Doc Values。这意味着字符串也享有许多与数字相同的压缩特点。

**Doc Values 是在字段索引时与倒排索引同时生成的，而且生成以后是不可变的。**

从 2.0 开始，Doc Values 默认对除分词字符串（analyzed string）以外的所有字段启用（因为分词后会生成很多 token，使得 Doc Values 效率降低）。5.0 起 `string` 类型被拆分为 `text` 和 `keyword`，`text` 字段不支持 Doc Values，而 `keyword`、数值、日期等其他支持 Doc Values 的字段默认开启。当你确定某些字段永远不会用于排序、聚合以及脚本操作时，可以禁用 Doc Values，以节约磁盘空间、提升索引速度，示例如下：

```json
PUT my_index
{
  "mappings": {
    "properties": {
      "session_id": {
        "type": "keyword",
        "doc_values": false
      }
    }
  }
}
```

这样配置以后，`session_id` 字段就只能被搜索，不能被用于排序、聚合以及脚本操作了。

还可以通过设置 `"index": false`（`doc_values` 默认为 `true`），让字段不建立倒排索引，但仍可以用于排序、聚合以及脚本操作：

```json
PUT my_index
{
  "mappings": {
    "properties": {
      "customer_token": {
        "type": "keyword",
        "index": false
      }
    }
  }
}
```

在 8.x 中，像这样只开启了 Doc Values 的 `keyword`、数值等字段仍然可以被查询，只是速度较慢。

> 早期版本中，上述示例要写成 `"type": "string"` 加 `"index": "not_analyzed"`，并且需要放在 `my_type` 这样的映射类型下。`string` 类型在 5.0 中已被 `text`/`keyword` 取代，映射类型（type）在 7.0 中被废弃、8.0 中被移除，这些写法在当前版本中都已不再可用。

Doc Values 的特点就是快速、高效、内存友好，它使用由 Linux 内核管理的文件系统缓存，可以弹性地利用内存。Doc Values 在排序、聚合以及与字段相关的脚本计算中得到了高效的运用，任何需要查找某个文档包含哪些值的操作都要用到它。如果你确定某个 field 不会进行这些字段相关的操作，可以直接关闭 `doc_values`，以节约磁盘空间、加快索引速度。

# 2 Fielddata

上文说过，在排序、聚合以及在脚本中访问字段值时，需要一种与倒排索引截然不同的数据访问模式：不同于倒排索引中“查找 term -> 找到对应 docs”的过程，我们需要直接查找 doc，然后找到某个 field 中包含的 terms。

大多数 field 使用索引时生成、存放在磁盘上的 Doc Values 来支持这种访问模式，但分词的 `text` 字段不支持 Doc Values，而是使用一种叫 Fielddata 的数据结构。

**Fielddata 主要是针对 `text` 字段的**，它是一种查询时（query-time）构建的数据结构。

需要特别注意的是，从 5.0 开始，`text` 字段默认禁用 Fielddata，直接对其进行排序或聚合会报错。如果确实需要，必须在映射中显式设置 `"fielddata": true`，但官方并不推荐这样做。更好的做法是通过多字段（multi-fields）为它增加一个 `keyword` 子字段，用子字段来排序和聚合：

```json
PUT my_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "raw": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```

这样，全文搜索使用 `title`，排序和聚合使用 `title.raw` 即可。

Fielddata 缓存主要应用于对某个 field 进行排序或计算类的聚合运算。它会把这个 field 列的所有值加载到内存，以便快速地按文档访问这些值。为 field 构建 Fielddata 缓存的代价可能很高，因此建议为它分配足够的内存，并让它保持在已加载状态。

**Fielddata 是在第一次将该 field 用于聚合、排序或在脚本中访问时按需构建的**。构建时会从磁盘读取每个段的整个倒排索引，然后逆置 `term -> doc` 的关系，并将结果存储在 JVM 堆中。

所以，加载 Fielddata 是开销很大的操作，一旦它被加载，就会在整个段的生命周期内保留在内存中。

这里可以注意一下 Fielddata 和 Doc Values 的区别：在较早的版本中，其他数据类型也使用 Fielddata，但现在已经被在文档索引时创建的 Doc Values 所取代。

JVM 堆内存资源非常宝贵，用好它对系统的高效稳定运行至关重要。Fielddata 是直接放在堆内的，所以必须合理设定用于存放它的堆内存大小。ES 中控制 Fielddata 内存使用的参数是 `indices.fielddata.cache.size`，可以用百分比表示占该节点堆内存的比例，也可以用 `12GB` 这样的具体数值。默认情况下，这个设置是不限制的，ES 不会从 Fielddata 中驱逐数据。设置之后，如果生成的 Fielddata 大小超过指定的 size，就会驱逐其他值以腾出空间。使用时一定要注意，这个设置只是一个安全策略，而不是内存不足的解决方案。因为一旦通过此配置触发数据驱逐，ES 就会立刻开始从磁盘加载数据，并驱逐其他数据以保证有足够空间，导致很高的 I/O 以及大量需要被垃圾回收的内存垃圾。

举个例子：每天为日志文件建一个新的索引。一般来说我们只对最近几天的数据感兴趣，很少查询老数据。但是按默认设置，Fielddata 中的老索引数据是不会被驱逐的。这样的话，Fielddata 就会一直持续增长，直到触发**熔断机制**，这个机制会让你再也不能加载更多的 Fielddata 到内存。在这样的场景下，你只能访问老索引的 Fielddata，而不能加载更多新数据。所以，这时就可以通过以上配置，把最近最少使用的 Fielddata 驱逐出去，为新进来的数据腾出空间。

`indices.fielddata.cache.size` 是在数据加载后才检查的，那么如果一个查询尝试加载超过可用内存的数据，就会导致 OOM 异常。ES 使用 Fielddata 断路器（Circuit Breaker）来处理上述问题，它可以通过分析一个查询涉及的字段的类型、基数、大小等来评估所需内存。如果估计的内存大小超过配置的堆内存使用比例限制，断路器就会跳闸，查询将被中止并返回异常。

断路器是在数据加载前工作的，所以可以大大降低因 Fielddata 导致 OOM 的风险。ES 拥有多种类型的断路器：

- `indices.breaker.fielddata.limit`：Fielddata 断路器，默认为 JVM 堆的 40%
- `indices.breaker.request.limit`：请求断路器，默认为 JVM 堆的 60%
- `indices.breaker.total.limit`：父断路器，默认为 JVM 堆的 95%（`indices.breaker.total.use_real_memory` 为 `false` 时为 70%）

可以根据实际需要进行配置。

Fielddata 是为分词字段而生的，它会消耗大量的 Java 堆空间，特别是加载基数（cardinality）很大的分词字段时。而且，对这种分词字段做聚合往往是没有意义的。

值得注意的是，Fielddata 和 Doc Values 的生成时机不同，前者是在首次查询时，后者是在文档索引时。还有一点，Fielddata 是按每个段来缓存的。

# 3 Doc Values 与 Fielddata 对比

Doc Values 与 Fielddata 一个很显著的区别是：前者主要工作在磁盘上，而后者主要工作在内存中。

| 维度     | Doc Values     | Fielddata                                  |
| -------- | -------------- | ------------------------------------------ |
| 创建时间 | 索引时创建     | 使用时动态创建                             |
| 创建位置 | 磁盘           | 内存（JVM 堆）                             |
| 优点     | 不占用 JVM 堆内存 | 不占用磁盘空间                          |
| 缺点     | 索引速度稍低，访问速度略慢于内存 | 文档很多时，动态创建开销较大，而且占用堆内存 |

索引速度稍低是因为 Doc Values 需要在索引时额外构建列式数据；而在查询时，它的访问速度也略慢于 Fielddata，这其实仔细想想也可以理解：拿排序举例，一个在磁盘上排序，一个在内存中排序，谁的速度快不言自明。

ES 1.x 版本的官方说法是：

> Doc values are now only about 10–25% slower than in-memory fielddata

虽然速度稍慢，Doc Values 的优势还是非常明显的。一个很显著的点就是它不会随着文档的增多而引起 OOM 问题。正如前面所说，Doc Values 在磁盘上创建排序和聚合所需的正排索引。这样我们就不必在生产环境中给 ES 设置一个很大的堆内存，也使得 JVM 的 GC 更加高效，这又为其他操作带来了间接的好处。

而且，随着 ES 版本的升级，Doc Values 的优化越来越好，速度已经很接近 Fielddata 了，再加上硬盘的访问速度也越来越快（比如 SSD），所以 Doc Values 现在可以满足绝大部分场景，也是 ES 官方重点维护的对象。

所以我想说的是，Doc Values 相比 Fielddata 还是有很多优势的。因此从 ES 2.x 开始，支持聚合的字段默认都使用 Doc Values，而不是 Fielddata。

# 4 Global Ordinals 全局序号

Global Ordinals 是一个构建在 Doc Values 和 Fielddata 之上的数据结构，它为每个唯一的 term 按字典序维护了一个自增的数字序列。每个 term 都有自己唯一的数字，而且字母 A 的全局序号小于字母 B。特别注意，全局序号只用于 `keyword`、`ip`、`flattened` 等基于序号的字段，以及开启了 Fielddata 的 `text` 字段，数值类型字段不使用全局序号。

请注意，Doc Values 和 Fielddata 也有自己的序号（ordinals），这个序号只在特定的 Segment 和 field 中唯一。全局序号则在此基础上，通过提供段序号（Segment Ordinals）与全局序号之间的映射关系来构建，它在整个分片（Shard）内是唯一的。

一个特定字段的 Global Ordinals 与一个分片中的所有段相关，而 Doc Values 和 Fielddata 的序号只与单个段相关。因此，只要有一个新段变得可见，就必须完全重建全局序号。

也就是说，与 Fielddata 一样，全局序号在默认情况下也是懒加载的，会在第一个需要用到它的请求（如 terms 聚合）命中一个索引时构建。实际上，在为每个段加载好 Doc Values 或 Fielddata 后，ES 就会创建一个称为 Global Ordinals（全局序号）的数据结构，用来构建一个由分片内所有段中的唯一 term 组成的列表。

全局序号的内存开销小，是因为它有非常高效的压缩机制。可以在映射中为字段设置 `"eager_global_ordinals": true` 来提前加载全局序号，把构建开销从第一次搜索时转移到 Refresh 时，代价是 Refresh 会变慢一些。早期版本中对应的写法是 `fielddata.loading: eager_global_ordinals`，该配置在 5.0 中已被移除。

全局序号的加载时间取决于字段中 term 的数量，但总体来说耗时较低，因为构建它所需的序号数据已经由 Doc Values 或 Fielddata 提供了。

全局序号在用到段序号的场景中很有用，比如排序或者 terms 聚合（terms aggregation），可以提升执行效率。

我们举个简单的例子。比如有十亿级别的 doc，每个 doc 都有一个 `status` 字段，但只有 pending、published、deleted 三种状态。如果直接把整个字符串存到内存，就算每个 doc 只有 15 字节，一共也差不多有 14GB 的数据。怎么减少占用的空间呢？首先想到的就是用数字来进行编码，码表如下：

```text
Ordinal | Term
-------------------
0       | status_deleted
1       | status_pending
2       | status_published
```

这样的话，最初的那三个字符串就只在码表内被存了一次。Fielddata 中的 doc 就可以直接用编码来指向实际值：

```text
Doc     | Ordinal
-------------------------
0       | 1  # pending
1       | 1  # pending
2       | 2  # published
3       | 0  # deleted
```

这样编码以后，数据量直接被压缩了十倍左右。但有个问题是 Fielddata 是按每个段分别加载和缓存的，那么就会出现一种情况：如果一个段内的 doc 只有 deleted 和 published 两种状态，那么该段的 Fielddata 算出来的码表就只有 0 和 1，这就和拥有 3 种状态的段算出的码表不同。这样的话，聚合时就必须一个段一个段地计算，最后再汇总，十分缓慢，开销巨大。

ES 的做法是用 Global Ordinals 这种构建在 Fielddata 之上的小巧数据结构，结合所有段来计算唯一值，然后存放为一个序号码表。这样一来，terms 聚合就可以只在全局序号上进行，并且只在聚合的最终阶段把序号转换为真实的字符串值一次。这个机制可以将聚合性能提升 3~4 倍。

# 5 小结

Doc Values 和 Fielddata 解决的都是“由文档找值”的正排访问问题：Doc Values 在索引时构建、存放在磁盘上并依赖 OS Cache，是当前排序、聚合和脚本访问的默认方案；Fielddata 在查询时构建、常驻 JVM 堆，只用于 `text` 字段且默认禁用，实际使用中应优先通过 `keyword` 子字段来实现排序和聚合。全局序号则在两者之上为 `keyword` 等字段提供分片级统一的编码，是 terms 聚合高效执行的关键，可以通过 `eager_global_ordinals` 控制其加载时机。
