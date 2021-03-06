# Redis中的LRU淘汰策略分析

`Redis`作为缓存使用时，一些场景下要考虑内存的空间消耗问题。`Redis`会删除过期键以释放空间，过期键的删除策略有两种：

- 惰性删除：每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话，就删除该键；如果没有过期，就返回该键。
- 定期删除：每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。

另外，`Redis`也可以开启`LRU`功能来自动淘汰一些键值对。

## LRU算法

当需要从缓存中淘汰数据时，我们希望能淘汰那些将来不可能再被使用的数据，保留那些将来还会频繁访问的数据，但最大的问题是缓存并不能预言未来。一个解决方法就是通过`LRU`进行预测：最近被频繁访问的数据将来被访问的可能性也越大。缓存中的数据一般会有这样的访问分布：一部分数据拥有绝大部分的访问量。当访问模式很少改变时，可以记录每个数据的最后一次访问时间，拥有最少空闲时间的数据可以被认为将来最有可能被访问到。

举例如下的访问模式，A每5s访问一次，B每2s访问一次，C与D每10s访问一次，`|`代表计算空闲时间的截止点：

```
​~~~~~A~~~~~A~~~~~A~~~~A~~~~~A~~~~~A~~|
~~B~~B~~B~~B~~B~~B~~B~~B~~B~~B~~B~~B~|
​~~~~~~~~~~C~~~~~~~~~C~~~~~~~~~C~~~~~~|
​~~~~~D~~~~~~~~~~D~~~~~~~~~D~~~~~~~~~D|

```

可以看到，`LRU`对于A、B、C工作的很好，完美预测了将来被访问到的概率B>A>C，但对于D却预测了最少的空闲时间。

但是，总体来说，`LRU`算法已经是一个性能足够好的算法了

## LRU配置参数

`Redis`配置中和`LRU`有关的有三个：

- `maxmemory`: 配置`Redis`存储数据时指定限制的内存大小，比如`100m`。当缓存消耗的内存超过这个数值时, 将触发数据淘汰。该数据配置为0时，表示缓存的数据量没有限制, 即LRU功能不生效。64位的系统默认值为0，32位的系统默认内存限制为3GB
- `maxmemory_policy`: 触发数据淘汰后的淘汰策略
- `maxmemory_samples`: 随机采样的精度，也就是随即取出key的数目。该数值配置越大, 越接近于真实的LRU算法，但是数值越大，相应消耗也变高，对性能有一定影响，样本值默认为5。

## 淘汰策略

淘汰策略即`maxmemory_policy`的赋值有以下几种：

- `noeviction`:如果缓存数据超过了`maxmemory`限定值，并且客户端正在执行的命令(大部分的写入指令，但DEL和几个指令例外)会导致内存分配，则向客户端返回错误响应
- `allkeys-lru`: 对所有的键都采取`LRU`淘汰
- `volatile-lru`: 仅对设置了过期时间的键采取`LRU`淘汰
- `allkeys-random`: 随机回收所有的键
- `volatile-random`: 随机回收设置过期时间的键
- `volatile-ttl`: 仅淘汰设置了过期时间的键---淘汰生存时间`TTL(Time To Live)`更小的键

`volatile-lru`, `volatile-random`和`volatile-ttl`这三个淘汰策略使用的不是全量数据，有可能无法淘汰出足够的内存空间。在没有过期键或者没有设置超时属性的键的情况下，这三种策略和`noeviction`差不多。

一般的经验规则:

- 使用`allkeys-lru`策略：当预期请求符合一个幂次分布(二八法则等)，比如一部分的子集元素比其它其它元素被访问的更多时，可以选择这个策略。
- 使用`allkeys-random`：循环连续的访问所有的键时，或者预期请求分布平均（所有元素被访问的概率都差不多）
- 使用`volatile-ttl`：要采取这个策略，缓存对象的`TTL`值最好有差异

`volatile-lru` 和 `volatile-random`策略，当你想要使用单一的`Redis`实例来同时实现缓存淘汰和持久化一些经常使用的键集合时很有用。未设置过期时间的键进行持久化保存，设置了过期时间的键参与缓存淘汰。不过一般运行两个实例是解决这个问题的更好方法。

为键设置过期时间也是需要消耗内存的，所以使用`allkeys-lru`这种策略更加节省空间，因为这种策略下可以不为键设置过期时间。

## 近似LRU算法

我们知道，`LRU`算法需要一个双向链表来记录数据的最近被访问顺序，但是出于节省内存的考虑，`Redis`的`LRU`算法并非完整的实现。`Redis`并不会选择最久未被访问的键进行回收，相反它会尝试运行一个近似`LRU`的算法，通过对少量键进行取样，然后回收其中的最久未被访问的键。通过调整每次回收时的采样数量`maxmemory-samples`，可以实现调整算法的精度。

根据`Redis`作者的说法，每个`Redis Object`可以挤出24 bits的空间，但24 bits是不够存储两个指针的，而存储一个低位时间戳是足够的，`Redis Object`以秒为单位存储了对象新建或者更新时的`unix time`，也就是`LRU clock`，24 bits数据要溢出的话需要194天，而缓存的数据更新非常频繁，已经足够了。

`Redis`的键空间是放在一个哈希表中的，要从所有的键中选出一个最久未被访问的键，需要另外一个数据结构存储这些源信息，这显然不划算。最初，`Redis`只是随机的选3个key，然后从中淘汰，后来算法改进到了`N个key`的策略，默认是5个。

`Redis`3.0之后又改善了算法的性能，会提供一个待淘汰候选key的`pool`，里面默认有16个key，按照空闲时间排好序。更新时从`Redis`键空间随机选择N个key，分别计算它们的空闲时间`idle`，key只会在`pool`不满或者空闲时间大于`pool`里最小的时，才会进入`pool`，然后从`pool`中选择空闲时间最大的key淘汰掉。

真实`LRU`算法与近似`LRU`的算法可以通过下面的图像对比： 



浅灰色带是已经被淘汰的对象，灰色带是没有被淘汰的对象，绿色带是新添加的对象。可以看出，`maxmemory-samples`值为5时`Redis 3.0`效果比`Redis 2.8`要好。使用10个采样大小的`Redis 3.0`的近似`LRU`算法已经非常接近理论的性能了。

数据访问模式非常接近幂次分布时，也就是大部分的访问集中于部分键时，`LRU`近似算法会处理得很好。

在模拟实验的过程中，我们发现如果使用幂次分布的访问模式，真实`LRU`算法和近似`LRU`算法几乎没有差别。

![](https://i.loli.net/2019/12/11/HPg5FcWa2whvXmS.png)