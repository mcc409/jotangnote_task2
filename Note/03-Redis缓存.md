# Redis 查询缓存

## 一、功能说明

给「按 ID 查询笔记」接口加 Redis 缓存，减轻 MySQL 压力。

**查询流程：**
1. 先查 Redis
2. Redis 有 → 直接返回
3. Redis 没有 → 查 MySQL
4. MySQL 查到 → 写入 Redis → 返回

## 二、核心代码

### application.properties 配置

```properties
spring.data.redis.host=localhost
spring.data.redis.port=6379
```


- NoteController 的 get 方法

```java
@GetMapping("/{id}")
public Note get(@PathVariable Integer id) throws Exception {
    String key = "note:" + id;

    // 1. 先查 Redis
    String cached = redisTemplate.opsForValue().get(key);
    if (cached != null) {
        System.out.println("【缓存命中】从 Redis 返回");
        return objectMapper.readValue(cached, Note.class);
    }

    // 2. Redis 没有，查 MySQL
    System.out.println("【缓存未命中】从 MySQL 查询");
    Note note = noteMapper.findById(id);
    if (note == null) {
        return null;
    }

    // 3. 写入 Redis
    redisTemplate.opsForValue().set(key, objectMapper.writeValueAsString(note));
    return note;
}
```

- 缓存一致性：修改/删除时删缓存

```java
// 修改笔记后
redisTemplate.delete("note:" + id);

// 删除笔记后
redisTemplate.delete("note:" + id);
```

为什么要删缓存？ 如果 MySQL 改了，Redis 还是旧数据，用户就会查到过期内容。删掉缓存后，下次查询会重新从 MySQL 加载。

## 三、测试流程

1. 第一次查 id=1 → 控制台打印 【缓存未命中】从 MySQL 查询，耗时 2212ms
2. 第二次查 id=1 → 控制台打印 【缓存命中】从 Redis 返回，耗时 21ms

./imgs/redis-hit.png

## 四、效果对比

查询 耗时 数据来源
第一次 2212ms MySQL
第二次 21ms Redis

速度快了 100 倍。