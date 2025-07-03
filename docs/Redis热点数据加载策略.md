# Redis热点数据加载策略详解

## 1. 热点数据概述

热点数据是指访问频率高、对系统性能影响大的数据。在Redis中，合理加载热点数据可以显著提升系统性能和用户体验。

### 1.1 热点数据特征
- **访问频率高**：短时间内被大量请求访问
- **影响范围广**：影响多个业务模块
- **性能敏感**：直接影响用户体验
- **数据量适中**：适合放入内存缓存

### 1.2 热点数据识别方法

```bash
# 使用redis-cli的--hotkeys选项识别热点key
redis-cli --hotkeys

# 使用OBJECT FREQ命令查看key的访问频率
redis-cli OBJECT FREQ key_name

# 使用INFO stats查看整体统计信息
redis-cli INFO stats | grep keyspace
```

## 2. 热点数据加载策略

### 2.1 预加载策略（Preload）

在系统启动或业务低峰期预先加载热点数据。

#### 实现方式：

```java
@Component
public class HotDataPreloader {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private UserService userService;
    
    @PostConstruct
    public void preloadHotData() {
        // 1. 加载热门用户信息
        List<User> hotUsers = userService.getHotUsers();
        for (User user : hotUsers) {
            String key = "user:" + user.getId();
            redisTemplate.opsForValue().set(key, user, Duration.ofHours(24));
        }
        
        // 2. 加载热门商品信息
        List<Product> hotProducts = productService.getHotProducts();
        for (Product product : hotProducts) {
            String key = "product:" + product.getId();
            redisTemplate.opsForValue().set(key, product, Duration.ofHours(12));
        }
        
        // 3. 加载排行榜数据
        loadRankingData();
    }
    
    private void loadRankingData() {
        // 使用ZSET存储排行榜
        List<RankingItem> topItems = rankingService.getTopItems(100);
        for (RankingItem item : topItems) {
            redisTemplate.opsForZSet().add("ranking:hot", item.getId(), item.getScore());
        }
    }
}
```

#### 优点：
- 系统启动后即可提供服务
- 避免冷启动问题
- 用户体验好

#### 缺点：
- 需要预测热点数据
- 可能加载不必要的数据
- 占用额外内存

### 2.2 懒加载策略（Lazy Loading）

在首次访问时加载数据到缓存。

#### 实现方式：

```java
@Service
public class UserService {
    
    @Autowired
    private RedisTemplate<String, User> redisTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    public User getUserById(Long userId) {
        String key = "user:" + userId;
        
        // 1. 尝试从缓存获取
        User user = redisTemplate.opsForValue().get(key);
        if (user != null) {
            return user;
        }
        
        // 2. 缓存未命中，从数据库加载
        user = userRepository.findById(userId).orElse(null);
        if (user != null) {
            // 3. 写入缓存，设置过期时间
            redisTemplate.opsForValue().set(key, user, Duration.ofHours(24));
        }
        
        return user;
    }
}
```

#### 优点：
- 按需加载，节省内存
- 实现简单
- 适合数据量大的场景

#### 缺点：
- 首次访问延迟高
- 可能出现缓存穿透
- 需要处理并发加载问题

### 2.3 定时刷新策略（Scheduled Refresh）

定期更新热点数据，保持数据新鲜度。

#### 实现方式：

```java
@Component
@EnableScheduling
public class HotDataRefresher {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private ProductService productService;
    
    // 每5分钟刷新一次热门商品
    @Scheduled(fixedRate = 300000)
    public void refreshHotProducts() {
        List<Product> hotProducts = productService.getHotProducts();
        
        // 使用管道批量更新
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (Product product : hotProducts) {
                String key = "product:" + product.getId();
                byte[] value = JSON.toJSONString(product).getBytes();
                connection.setEx(key.getBytes(), 3600, value); // 1小时过期
            }
            return null;
        });
    }
    
    // 每小时刷新一次排行榜
    @Scheduled(fixedRate = 3600000)
    public void refreshRanking() {
        List<RankingItem> topItems = rankingService.getTopItems(100);
        
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            connection.del("ranking:hot".getBytes());
            for (RankingItem item : topItems) {
                connection.zAdd("ranking:hot".getBytes(), item.getScore(), 
                              String.valueOf(item.getId()).getBytes());
            }
            return null;
        });
    }
}
```

### 2.4 事件驱动加载策略（Event-Driven Loading）

基于业务事件触发数据加载。

#### 实现方式：

```java
@Component
public class HotDataEventListener {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @EventListener
    public void handleUserLoginEvent(UserLoginEvent event) {
        // 用户登录时，预加载用户相关热点数据
        Long userId = event.getUserId();
        
        // 加载用户基本信息
        User user = userService.getUserById(userId);
        redisTemplate.opsForValue().set("user:" + userId, user, Duration.ofHours(24));
        
        // 加载用户推荐商品
        List<Product> recommendations = recommendationService.getRecommendations(userId);
        redisTemplate.opsForList().rightPushAll("recommendations:" + userId, recommendations);
        redisTemplate.expire("recommendations:" + userId, Duration.ofHours(2));
    }
    
    @EventListener
    public void handleProductViewEvent(ProductViewEvent event) {
        // 商品被查看时，更新热门商品列表
        Long productId = event.getProductId();
        
        // 增加商品热度分数
        redisTemplate.opsForZSet().incrementScore("hot_products", productId, 1);
        
        // 如果热度超过阈值，加入热门商品缓存
        Double score = redisTemplate.opsForZSet().score("hot_products", productId);
        if (score != null && score > 100) {
            Product product = productService.getProductById(productId);
            redisTemplate.opsForValue().set("hot_product:" + productId, product, Duration.ofHours(6));
        }
    }
}
```

## 3. 高级加载策略

### 3.1 分层缓存策略

```java
@Service
public class LayeredCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private CaffeineCacheManager localCache;
    
    public Object getData(String key) {
        // 1. 查询本地缓存（L1）
        Object data = localCache.get(key);
        if (data != null) {
            return data;
        }
        
        // 2. 查询Redis缓存（L2）
        data = redisTemplate.opsForValue().get(key);
        if (data != null) {
            // 回填本地缓存
            localCache.put(key, data);
            return data;
        }
        
        // 3. 查询数据库
        data = loadFromDatabase(key);
        if (data != null) {
            // 同时写入本地缓存和Redis
            localCache.put(key, data);
            redisTemplate.opsForValue().set(key, data, Duration.ofHours(1));
        }
        
        return data;
    }
}
```

### 3.2 智能预加载策略

```java
@Component
public class SmartPreloader {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private AccessLogService accessLogService;
    
    // 基于访问模式智能预加载
    @Scheduled(fixedRate = 60000) // 每分钟执行
    public void smartPreload() {
        // 1. 分析访问模式
        Map<String, Integer> accessPatterns = accessLogService.analyzeAccessPatterns();
        
        // 2. 预测下一个热点数据
        List<String> predictedHotKeys = predictHotKeys(accessPatterns);
        
        // 3. 预加载预测的热点数据
        for (String key : predictedHotKeys) {
            if (!redisTemplate.hasKey(key)) {
                Object data = loadDataFromDatabase(key);
                if (data != null) {
                    redisTemplate.opsForValue().set(key, data, Duration.ofMinutes(30));
                }
            }
        }
    }
    
    private List<String> predictHotKeys(Map<String, Integer> accessPatterns) {
        // 基于时间序列分析预测热点
        return accessPatterns.entrySet().stream()
            .filter(entry -> entry.getValue() > 10) // 访问频率阈值
            .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
            .limit(20) // 预加载前20个
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
    }
}
```

### 3.3 批量加载策略

```java
@Service
public class BatchLoaderService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void batchLoadUsers(List<Long> userIds) {
        // 使用管道批量加载
        redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
            for (Long userId : userIds) {
                String key = "user:" + userId;
                
                // 检查是否已存在
                if (!connection.exists(key.getBytes())) {
                    User user = userRepository.findById(userId).orElse(null);
                    if (user != null) {
                        byte[] value = JSON.toJSONString(user).getBytes();
                        connection.setEx(key.getBytes(), 3600, value);
                    }
                }
            }
            return null;
        });
    }
    
    public void batchLoadWithPriority(List<LoadTask> tasks) {
        // 按优先级分批加载
        Map<Integer, List<LoadTask>> priorityGroups = tasks.stream()
            .collect(Collectors.groupingBy(LoadTask::getPriority));
        
        // 高优先级先加载
        for (int priority = 1; priority <= 3; priority++) {
            List<LoadTask> groupTasks = priorityGroups.get(priority);
            if (groupTasks != null) {
                batchLoadTasks(groupTasks);
            }
        }
    }
}
```

## 4. 性能优化技巧

### 4.1 使用管道（Pipeline）

```java
public void batchSetWithPipeline(Map<String, Object> dataMap) {
    redisTemplate.executePipelined((RedisCallback<Object>) connection -> {
        for (Map.Entry<String, Object> entry : dataMap.entrySet()) {
            String key = entry.getKey();
            Object value = entry.getValue();
            byte[] valueBytes = JSON.toJSONString(value).getBytes();
            connection.setEx(key.getBytes(), 3600, valueBytes);
        }
        return null;
    });
}
```

### 4.2 使用Lua脚本

```lua
-- 批量检查和设置缓存
local function batchCheckAndSet(keys, values, ttl)
    local results = {}
    for i, key in ipairs(keys) do
        local exists = redis.call('EXISTS', key)
        if exists == 0 then
            redis.call('SETEX', key, ttl, values[i])
            results[i] = 1
        else
            results[i] = 0
        end
    end
    return results
end
```

### 4.3 异步加载

```java
@Service
public class AsyncLoaderService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Async
    public CompletableFuture<Void> asyncLoadData(String key) {
        try {
            Object data = loadFromDatabase(key);
            if (data != null) {
                redisTemplate.opsForValue().set(key, data, Duration.ofHours(1));
            }
        } catch (Exception e) {
            log.error("Async load data failed for key: " + key, e);
        }
        return CompletableFuture.completedFuture(null);
    }
    
    public Object getDataWithAsyncLoad(String key) {
        // 先返回缓存数据
        Object data = redisTemplate.opsForValue().get(key);
        if (data != null) {
            return data;
        }
        
        // 异步加载数据
        asyncLoadData(key);
        
        // 返回默认值或null
        return getDefaultValue(key);
    }
}
```

## 5. 监控和调优

### 5.1 缓存命中率监控

```java
@Component
public class CacheMonitor {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Scheduled(fixedRate = 60000)
    public void monitorCacheHitRate() {
        Properties info = redisTemplate.getConnectionFactory()
            .getConnection().info("stats");
        
        long hits = Long.parseLong(info.getProperty("keyspace_hits"));
        long misses = Long.parseLong(info.getProperty("keyspace_misses"));
        
        double hitRate = (double) hits / (hits + misses);
        
        // 记录监控指标
        log.info("Cache hit rate: {:.2%}", hitRate);
        
        // 如果命中率过低，触发优化
        if (hitRate < 0.8) {
            triggerCacheOptimization();
        }
    }
}
```

### 5.2 热点数据识别

```java
@Component
public class HotKeyDetector {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public List<String> detectHotKeys() {
        List<String> hotKeys = new ArrayList<>();
        
        // 使用SCAN命令遍历所有key
        String pattern = "*";
        ScanOptions options = ScanOptions.scanOptions().match(pattern).count(100).build();
        
        Cursor<String> cursor = redisTemplate.scan(options);
        while (cursor.hasNext()) {
            String key = cursor.next();
            
            // 检查key的访问频率
            Long freq = getKeyFrequency(key);
            if (freq != null && freq > 100) { // 阈值可配置
                hotKeys.add(key);
            }
        }
        
        return hotKeys;
    }
    
    private Long getKeyFrequency(String key) {
        try {
            // 使用OBJECT FREQ命令获取访问频率
            Object result = redisTemplate.execute((RedisCallback<Object>) connection -> {
                return connection.objectEncoding(key.getBytes());
            });
            return result != null ? 1L : 0L; // 简化实现
        } catch (Exception e) {
            return null;
        }
    }
}
```

## 6. 最佳实践

### 6.1 数据选择原则

1. **访问频率高**：选择被频繁访问的数据
2. **计算成本高**：选择计算或查询成本高的数据
3. **数据量适中**：避免缓存过大的数据
4. **变化频率低**：优先缓存相对稳定的数据

### 6.2 过期时间设置

```java
public class ExpirationStrategy {
    
    // 根据数据特性设置不同的过期时间
    public Duration getExpirationTime(String key, Object data) {
        if (key.startsWith("user:")) {
            return Duration.ofHours(24); // 用户数据24小时
        } else if (key.startsWith("product:")) {
            return Duration.ofHours(6);  // 商品数据6小时
        } else if (key.startsWith("ranking:")) {
            return Duration.ofHours(1);  // 排行榜1小时
        } else if (key.startsWith("config:")) {
            return Duration.ofDays(7);   // 配置数据7天
        }
        return Duration.ofHours(1); // 默认1小时
    }
}
```

### 6.3 内存管理

```java
@Component
public class MemoryManager {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Scheduled(fixedRate = 300000) // 每5分钟检查
    public void checkMemoryUsage() {
        Properties info = redisTemplate.getConnectionFactory()
            .getConnection().info("memory");
        
        long usedMemory = Long.parseLong(info.getProperty("used_memory"));
        long maxMemory = Long.parseLong(info.getProperty("maxmemory"));
        
        double usageRatio = (double) usedMemory / maxMemory;
        
        if (usageRatio > 0.8) { // 内存使用率超过80%
            // 清理低优先级缓存
            cleanupLowPriorityCache();
        }
    }
    
    private void cleanupLowPriorityCache() {
        // 清理过期时间较短的缓存
        Set<String> keys = redisTemplate.keys("temp:*");
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
}
```

## 7. 总结

热点数据加载是Redis缓存优化的核心策略，需要根据业务特点选择合适的加载方式：

1. **预加载**：适合可预测的热点数据
2. **懒加载**：适合数据量大、访问不规律的场景
3. **定时刷新**：适合数据变化频繁的场景
4. **事件驱动**：适合基于业务事件触发的场景

通过合理的策略组合和性能优化，可以显著提升系统性能和用户体验。同时要注意监控缓存效果，及时调整策略参数。 