---
title: Redisson RDelayedQueue 3.21.1升级
date: '2026-01-11 14:14:21'
updated: '2026-01-11 14:16:40'
permalink: /post/redisson-rdelayedqueue-3211-upgrade-z1wnilo.html
comments: true
toc: true
---



# Redisson RDelayedQueue 3.21.1升级

# Redisson 3.21.1 延迟队列报错分析与解决方案

## 问题背景

在使用 Redisson 3.21.1 版本的延迟队列时，我们遇到了一个奇怪的报错：

```
[org.redisson.QueueTransferTask:147] ERR Error running script
(call to f_a3a86b949561b56e14c274ad54368afeff259bc8):
@user_script:1: user_script:1: bad argument #2 to 'unpack'
```

这个错误会大约每 5 秒钟出现一次，严重影响了应用的稳定性。

## 问题分析

### 1. Redisson 延迟队列的实现原理

Redisson 的延迟队列 (`RDelayedQueue`) 使用两个 Redis 数据结构：

- ​`List`​ (`redisson_delay_queue:{name}`)：存储待处理的任务
- ​`ZSet`​ (`redisson_delay_queue_timeout:{name}`)：存储任务的超时时间

当任务到期时，`QueueTransferTask` 后台线程会定期（默认每 5 秒）检查并将到期任务从延迟队列转移到目标阻塞队列中。

### 2. 数据格式的不兼容性

问题的根源在于 **Redisson 3.21.0 到 3.21.1 版本之间延迟队列存储格式的变更**。

在 Redisson 3.21.0 及之前版本，任务数据使用以下格式存储到 Redis 中：

```
struct.pack('dLc0', randomId, valueLength, value)
```

- ​`d`：双精度浮点数（randomId）
- ​`L`：无符号长整型（valueLength）
- ​`c0`：以 null 结尾的字符串（value）

但在 Redisson 3.21.1 版本中，数据格式发生了变化，导致原有的 `struct.unpack('dLc0', v)` 解析失败。

### 3. 错误的根本原因

当 Redisson 3.21.1 尝试处理 3.21.0 版本写入的任务时，`QueueTransferTask`​ 执行 Lua 脚本时会尝试使用新的解析格式去解析旧格式的数据，从而导致 `bad argument #2 to 'unpack'` 错误。

## 解决方案：数据迁移

为了解决这个问题，我们需要将旧版本格式的延迟队列数据进行迁移。

### 1. 迁移方案设计

```java
protected void transferData() throws Exception {
    String queueName = "{test}";

    // 创建自定义编解码器来处理特殊的数据格式
    RScript script = redissonClient.getScript(new BaseCodec() {
        int count = 0;

        @Override
        public Decoder<Object> getValueDecoder() {
            return (buf, state) -> {
                Object result;
                if (count % 2 == 0) {
                    result = LongCodec.INSTANCE.getValueDecoder().decode(buf, state);
                } else {
                    result = new TypedJsonJacksonCodec(TestData.class).getValueDecoder().decode(buf, state);
                }
                count++;
                return result;
            };
        }

        @Override
        public Encoder getValueEncoder() {
            return null;
        }
    });

    // Lua 脚本用于从 Redis 中读取延迟队列数据
    String luaScript =
            "local result = {}; " +
                    "local items = redis.call('lrange', KEYS[1], 0, -1); " +
                    "for i, v in ipairs(items) do " +
                    "    local timeout = redis.call('zscore', KEYS[2], v); " +
                    "    local randomId, value = struct.unpack('dLc0', v); " +
                    "    table.insert(result, timeout); " +
                    "    table.insert(result, value); " +
                    "end; " +
                    "return result; ";

    List<Object> result = script.eval(
            RScript.Mode.READ_ONLY,
            luaScript,
            RScript.ReturnType.MULTI,
            List.of("redisson_delay_queue:" + queueName, "redisson_delay_queue_timeout:" + queueName)
    );

    // 将数据写入新队列
    RDelayedQueue<Object> delayedQueue = redissonClient.getDelayedQueue(
            redissonClient.getBlockingQueue("test_v2", new TypedJsonJacksonCodec(TestData.class))
    );

    for (int i = 0; i < result.size(); i += 2) {
        Long timeout = (Long) result.get(i);
        Object data = result.get(i + 1);
        long delay = timeout - System.currentTimeMillis();
        if (delay < 0) {
            log.warn("{} 数据已过期 : {}", queueName, data);
            continue;
        }
        delayedQueue.offer(data, delay, TimeUnit.MILLISECONDS);
    }

    // 删除旧队列
    redissonClient.getKeys().delete("redisson_delay_queue:" + queueName, "redisson_delay_queue_timeout:" + queueName);
}
```

### 2. 迁移步骤

1. **启动新版本队列消费者**：先启动 3.21.1 版本的队列消费者
2. **迁移旧版本数据**：使用 Lua 脚本读取旧格式数据并转换
3. **启动旧版本队列消费者**：确保没有数据丢失
4. **清理旧队列**：数据迁移完成后删除旧队列

### 3. 完整的迁移流程

```java
@Override
public void run(ApplicationArguments args) throws Exception {
    consumerQueue("test_v2");  // 启动新版本队列消费者
    transferData();            // 迁移旧版本队列数据
    consumerQueue("test");     // 启动旧版本队列消费者
}
```

## 代码优化建议

### 1. 迁移过程的健壮性改进

```java
// 在迁移过程中添加异常处理
try {
    List<Object> result = script.eval(
            RScript.Mode.READ_ONLY,
            luaScript,
            RScript.ReturnType.MULTI,
            List.of("redisson_delay_queue:" + queueName, "redisson_delay_queue_timeout:" + queueName)
    );

    if (result != null && !result.isEmpty()) {
        // 数据迁移逻辑
    }
} catch (Exception e) {
    log.error("迁移延迟队列数据失败: {}", e.getMessage(), e);
    // 可以选择暂停迁移或继续尝试
}
```

### 2. 数据验证和转换

```java
// 在处理数据前进行验证
for (int i = 0; i < result.size(); i += 2) {
    Long timeout = (Long) result.get(i);
    Object data = result.get(i + 1);

    // 验证数据有效性
    if (timeout == null || data == null) {
        log.warn("跳过无效数据: timeout={}, data={}", timeout, data);
        continue;
    }

    long delay = timeout - System.currentTimeMillis();
    if (delay < 0) {
        log.warn("{} 数据已过期 : {}", queueName, data);
        continue;
    }

    delayedQueue.offer(data, delay, TimeUnit.MILLISECONDS);
}
```

## 总结与预防

### 1. 问题总结

- **问题根源**：Redisson 3.21.1 版本修改了延迟队列的存储格式，导致与 3.21.0 及之前版本的数据不兼容
- **报错机制**：QueueTransferTask 后台线程每 5 秒执行一次任务转移操作，尝试解析旧格式数据时失败
- **影响范围**：使用 Redisson 延迟队列功能的应用在升级到 3.21.1 版本时会受到影响

### 2. 预防措施

1. **版本升级前备份数据**：在升级前对 Redis 中的延迟队列数据进行备份
2. **使用统一版本**：确保所有使用延迟队列的应用节点都使用相同版本的 Redisson
3. **测试迁移方案**：在生产环境部署前，先在测试环境验证迁移方案的正确性
4. **监控迁移过程**：在数据迁移期间密切监控应用的运行状态和错误日志

### 3. 最佳实践

- 对于生产环境的 Redisson 版本升级，建议先进行充分的测试
- 考虑使用数据迁移工具或脚本自动化处理格式转换
- 监控 Redis 中的数据结构变化，及时发现潜在的兼容性问题
- 保持对 Redisson 官方文档和发布说明的关注，及时了解重要变更

通过以上方案，我们成功解决了 Redisson 3.21.1 延迟队列的报错问题，并建立了一套完整的数据迁移和版本管理策略。

‍

[项目地址](https://github.com/huisunan/redisson-migration-version-3-21-1.git)
