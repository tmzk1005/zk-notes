---
title: "流量限制-令牌桶算法"
date: 2022-07-05T11:29:58+08:00
draft: true
toc: false
tags: []
---

流量限制的目的是为了防止流量过大，超出了系统的承载能力，导致系统不健康甚至崩溃；流量限制的效果即当满足一定的条件后，丢弃多余的
流量，比如对于http服务器来说，可以对短时间内突发的过多请求直接返回拒绝服务的响应。

实现流量限制的常用算法：`令牌桶算法`，其基本的算法逻辑可以这样来理解：有一个容量一定的水桶，初始状态是满的；有一个水管以固定的
速率向水桶中放入水，当桶达到满状态时停止；有一个水管可以从桶中取走水，而每一个请求到来时，都通过此取水管从桶中取固定量的水，如果
能取到水，则表示未触发限流，请求放行，而如果没有足够的水能被取到，则表示达到了限流条件，直接拒绝请求。

![](./img1.svg)

这里面涉及到3个变量：

- burstCapacity: 令牌桶容量
    令牌桶能存放的令牌数，即蓄水池的容量
- replenishRate: 入水速率
    每个时间周期（秒，分钟，或者其它自定义固定周期均可）放入令牌桶的令牌数，即入水速率
- requestedTokens: 
    每个请求消耗的令牌数，即出水速率

以上3个参数即决定了基本的令牌桶算法的限流最终效果。

下面使用Java代码来实现令牌桶算法：

先定义一个暴露给业务使用者的接口:

RateLimiter.java
```java
public interface RateLimiter {

    /**
     * 判断是否允许通过
     *
     * @param key 唯一key标识限流主体
     * @return 时间戳， -1 表示被限制，>=0 表示通过，并返回计算令牌数的时间戳
     */
    long (String key);

}
```

`isAllowed`方法自身不要求`key`有任何特定的业务意义，`key`的实际业务意义有调用放来决定，对于http请求限流来说，如果
是对请求path限流，那么key可以是请求的path，如果是对客户端的ip进行限流，那么key可以是客户端的ip。至于返回结果，其实
返回一个Boolean值，或者把限流器内部的状态都返回回来也是可以的。这里是把时间戳返回回来，已被不时之需。

> 返回时间戳是基于实际碰到的一个业务场景：有一个网关会审计AccessLog，每个请求会有一个时间戳，假设有一个分析程序对AccessLog进行聚合分析，
比如统计每分钟的请求数，那么AccessLog先按当前时间来设置时间，然后进行限流判断，这个期间可能发生“跨分钟”，比如，AccessLog赋时间戳的
时间是“10:10:59.999”,而进入到限流判断时，限流逻辑的发生时间是"10:11:00.003",那么就会导致不同的逻辑把同一个请求算在了不同的时间窗口。

然后实现此接口：

RateLimiterImpl.java

```java
package zk.javalab.ratelimit;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class RateLimiterImpl implements RateLimiter {

    private final long burstCapacity;

    private final long replenishRate;

    private final long requestedTokens;

    private final Map<String, TokenInfo> internalStatus = new ConcurrentHashMap<>();

    public RateLimiterImpl(long burstCapacity, long replenishRate, long requestedTokens) {
        this.burstCapacity = burstCapacity;
        this.replenishRate = replenishRate;
        this.requestedTokens = requestedTokens;
    }

    @Override
    public synchronized long isAllowed(String key) {
        internalStatus.computeIfAbsent(key, s -> new TokenInfo());
        final long currentTimeMillis = System.currentTimeMillis();
        final TokenInfo tokenInfo = internalStatus.get(key);
        tokenInfo.leftTokens = Math.max(burstCapacity, tokenInfo.leftTokens + replenishRate * (currentTimeMillis - tokenInfo.updateTime));
        tokenInfo.updateTime = currentTimeMillis;
        if (tokenInfo.leftTokens < requestedTokens) {
            return -1;
        }
        tokenInfo.leftTokens = tokenInfo.leftTokens - requestedTokens;
        return currentTimeMillis;
    }

    class TokenInfo {

        private long updateTime;

        private long leftTokens;

        public TokenInfo() {
            this.updateTime = System.currentTimeMillis();
            this.leftTokens = burstCapacity;
        }

    }

}
```

> 这个简单的实现假设放令牌的时间周期是每秒钟，如果是每分钟，每10秒或者其他的周期，那么代码要做一定的调整。

上面的Java实现将限流的状态维持在单个JVM进程的内存中，只适合单进程应用来使用，如果是多个同样服务的进程要共享限流状态，
那么限流状态需要通过某种方式共享，并控制并发访问，一般的做法是状态放在redis，用lua脚本来实现令牌桶算法，大致的代码如下：

```lua
redis.replicate_commands()

local tokens_key = KEYS[1]
local timestamp_key = KEYS[2]

local rate = tonumber(ARGV[1])
local capacity = tonumber(ARGV[2])
local now_time = redis.call('TIME')
local now_seconds = tonumber(now_time[1])
local requested = tonumber(ARGV[3])

local fill_time = capacity / rate
local ttl = math.floor(fill_time * 2)

local last_tokens = tonumber(redis.call("get", tokens_key))
if last_tokens == nil then
  last_tokens = capacity
end

local last_refreshed = tonumber(redis.call("get", timestamp_key))
if last_refreshed == nil then
  last_refreshed = 0
end

local delta = math.max(0, now_seconds - last_refreshed)
local filled_tokens = math.min(capacity, last_tokens + (delta * rate))
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens
local allowed_num = 0
if allowed then
  new_tokens = filled_tokens - requested
  allowed_num = 1
end

if ttl > 0 then
  redis.call("setex", tokens_key, ttl, new_tokens)
  redis.call("setex", timestamp_key, ttl, now_seconds)
end

return { allowed_num, new_tokens, now_seconds, tonumber(now_time[2]) }
```