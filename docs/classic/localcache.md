### 1、缓存常见问题
#### 1.1 缓存穿透：请求的数据在缓存和数据库中都不存在
a. 空值缓存<br>
b. 布隆过滤器<br>

```java
import com.google.common.base.Charsets;
import com.google.common.hash.BloomFilter;
import com.google.common.hash.Funnels;

/**
 * Guava 布隆过滤器示例（解决缓存穿透：过滤不存在的商品ID）
 */
public class GuavaBloomFilterDemo {
    public static void main(String[] args) {
        // 1. 配置参数
        int expectedInsertions = 100000; // 预期插入的元素数量（如10万商品ID）
        double fpp = 0.01; // 误判率（1%）

        // 2. 创建布隆过滤器（String类型元素，字符集UTF-8）
        BloomFilter<String> bloomFilter = BloomFilter.create(
                Funnels.stringFunnel(Charsets.UTF_8),
                expectedInsertions,
                fpp
        );

        // 3. 向过滤器中添加元素（模拟加载所有有效商品ID）
        for (int i = 1; i <= expectedInsertions; i++) {
            bloomFilter.put("product_" + i); // 商品ID格式：product_1、product_2...
        }

        // 4. 测试查询（验证存在性）
        // 4.1 存在的元素（大概率命中，误判率1%）
        String existKey = "product_50000";
        boolean exist = bloomFilter.mightContain(existKey);
        System.out.println("元素 " + existKey + " 是否存在：" + exist); // 输出 true

        // 4.2 不存在的元素（一定返回false）
        String notExistKey = "product_100001";
        boolean notExist = bloomFilter.mightContain(notExistKey);
        System.out.println("元素 " + notExistKey + " 是否存在：" + notExist); // 输出 false

        // 4.3 统计误判率（可选）
        int wrongCount = 0;
        int testCount = 10000;
        for (int i = expectedInsertions + 1; i <= expectedInsertions + testCount; i++) {
            if (bloomFilter.mightContain("product_" + i)) {
                wrongCount++;
            }
        }
        System.out.println("实际误判率：" + (double) wrongCount / testCount); // 接近 1%
    }
}
```
注意：布隆过滤器不支持删除，否则可能引起误判率提升。解决方案上，业务规避删除场景，或者定时重建过滤器，或者采用变种布隆过滤器。

#### 缓存击穿：某个热点key突然过期
a. 热点key永不过期<br>
b. 互斥锁(分布式锁)<br>
c. 缓存预热<br>

#### 缓存雪崩：大量缓存key在同一时间段集中失效
a. 随机设置过期时间
b. 分层缓存


### 2、Guava Cache

```java
// 1. 初始化 LoadingCache
LoadingCache<String, User> loadingCache = CacheBuilder.newBuilder()
        .maximumSize(10000) // 最大容量
        .expireAfterWrite(10, TimeUnit.MINUTES) // 写入后10分钟过期
        .refreshAfterWrite(5, TimeUnit.MINUTES) // 写入后5分钟刷新
        .concurrencyLevel(8) // 并发级别（分段锁数量）
        .recordStats() // 开启统计
        .build(new CacheLoader<String, User>() {
            // 缓存未命中时自动加载
            @Override
            public User load(String key) throws Exception {
                // 模拟从数据库加载
                User user = userMapper.getById(key);
                // 解决缓存穿透：空值处理
                if (user == null) {
                    return new User(); // 返回空对象，而非null
                }
                return user;
            }
        });

// 2. 使用方式
// 2.1 自动加载（CacheLoader）
User user1 = loadingCache.get("123"); 
// 2.2 自定义加载（Callable）
User user2 = loadingCache.get("456", () -> {
    return userMapper.getById("456");
});

// 3. 统计信息
CacheStats stats = loadingCache.stats();
System.out.println("命中率：" + stats.hitRate()); // 核心优化指标
System.out.println("平均加载时间：" + stats.averageLoadPenalty());
```

#### 2.1 refresh与expire
**过期**：get数据的时候，如果链表上找不到entry或者value已经过期，就会调用lockedGetOrLoad方法，这个方法会锁住整个segment，直接从数据源加载数据，更新缓存。如果并发量比较大又遇到很多key失效就会很容易导致线程阻塞，可以考虑采用refresh机制规避该问题。

**刷新**：缓存项指定时间间隔被访问，会调用reload方法加载新值，在新值加载期间，旧值仍然会返回给任何请求它的调用者。reload方法应该返回一个ListenableFuture对象，这样刷新操作就可以异步执行，而不会阻塞其他缓存或线程操作。如果reload方法没有被重写，会使用load方法进行同步刷新。

#### 2.3 淘汰策略
基于时间的淘汰、给予容量的淘汰、弱引用淘汰、显示淘汰(Cache.invalidate)

底层结构是分段的哈希表+双向链表，整体借鉴ConsurrentHashMap（JDK7版本）的分段锁思路，同时结合近似LRU算法实现缓存淘汰。

两个核心双向链表(accessQueue/writeQueue)，前者核心关联expireAfterAccess和LRU容量淘汰，后者关联expireAfterWrite和refreshAfterWrite策略。


为了性能，采用近似LRU和懒加载过期检查，而非严格的试试淘汰，这是高性能的关键设计。

#### 2.4 常见问题
内存溢出：必须设置容量和过期时间，否则会导致缓存无限增长；<br>
并发性能低：分段锁数量设置不合理，根据CPU核心数调整<br>
命中率低：key设置不合理或过期时间短，优化key力度、调整过期时间、开启统计分析热点key<br>



### 3、Caffeine
#### 3.1 Caffeine为什么比Guava Cache快
a. 淘汰策略更高效：W-TinyLFU命中率高10%-20%，减少无效加载<br>
b. 并发模型优化：读操作无锁(CAS)，写操作分段锁，Guava Cache读写都需要分段锁
c. 内存布局优化：Caffeine的Entry结构更紧凑，减少内存碎片，GC压力更小
d. 异步加载：Caffeine支持异步加载，避免阻塞读请求
f. 过期清理优化：Caffeine结合惰性清理和定时任务清理

#### 3.2 W-TinyLFU对比LRU优势
**LRU问题**：只关注最近访问时间，突发流量会把热点数据挤出

**W-TinyLFU**：
1、结合访问频率和时间衰减：给每个key记录访问频率，频率随时间衰减(旧数据权重降低)<br>
2、引入Window Cache：缓存最近的访问记录，避免突发流量污染主存<br>

最终效果：更精准的保留真正的热点数据，命中率比LRU高10%到20%

#### 3.3 W-TinyLFU核心数据结构
1、FrequencySketch(频率草图)：低内存开销统计每个key的访问频率，使用位图+哈希函数，每个位置用4bit记录访问频率，频率随时间衰减。<br>
2、WindowCache(窗口缓存)：新key先进入WindowCache，只有达到一定频率后才进入主缓存，避免突发冷数据污染主缓存<br>
3、AdmissionWindow(准入窗口)：结合频率草图和窗口缓存，淘汰频率最低的key<br>
#### 3.4 如何选用
1、优先用Caffeine
a. 性能远超Guava，尤其是高并发场景
b. 兼容Guava Cache的API，迁移成本低
c. Spring5原生支持，适配高并发非阻塞场景

2、仅兼容老项目选Guava
a. 项目已深度依赖Guava，且无性能瓶颈
b. 团队对Guava更熟悉，无需额外学习成本

```java
// Guava 代码
LoadingCache<String, User> guavaCache = CacheBuilder.newBuilder()
        .maximumSize(10000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build(new CacheLoader<String, User>() {
            @Override
            public User load(String key) throws Exception {
                return userMapper.getById(key);
            }
        });

// Caffeine 等价代码（几乎无改动）
LoadingCache<String, User> caffeineCache = Caffeine.newBuilder()
        .maximumSize(10000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build(new CacheLoader<String, User>() {
            @Override
            public User load(String key) throws Exception {
                return userMapper.getById(key);
            }
        });

// Caffeine 新增异步加载
AsyncLoadingCache<String, User> asyncCache = Caffeine.newBuilder()
        .maximumSize(10000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .buildAsync((key) -> CompletableFuture.supplyAsync(() -> userMapper.getById(key)));
```
