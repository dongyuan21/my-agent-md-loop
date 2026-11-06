# 限流器：从固定窗口到滑动窗口

限流器只做一件事：一段时间内只放行有限次请求。接口很薄，差别在时间怎么切、记录怎么维护。下面 Java 用我自己写的版本，Python 跟着同一套逻辑。

## 框架

来一个请求，先按当前时间更新窗口（丢掉过期记录，或者补令牌），再看还有没有额度。有就记账并放行，没有就拒绝。

只记放行成功的请求。被拒绝的如果也写进队列，窗口会被自己污染。单机给记账加把锁就行；多机再换成 Redis 的原子操作。换算法时，改的是「怎么更新窗口」和「怎么判断额度」。

## 固定窗口

时间切成等长的格子。格子没变就往上加计数，格子一换就清零。

```java
public class FixedWindowRateLimiter {
    private final int maxRequests;
    private final long windowSizeInMillis;
    private long windowStart;
    private int requestCount;

    public FixedWindowRateLimiter(int maxRequests, long windowSizeInMillis) {
        this.maxRequests = maxRequests;
        this.windowSizeInMillis = windowSizeInMillis;
        this.windowStart = System.currentTimeMillis();
        this.requestCount = 0;
    }

    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        if (now - windowStart >= windowSizeInMillis) {
            // Reset the window
            windowStart = now;
            requestCount = 0;
        }

        if (requestCount < maxRequests) {
            requestCount++;
            return true; // Allow the request
        } else {
            return false; // Reject the request
        }
    }
}
```

```python
import time

class FixedWindowRateLimiter:
    def __init__(self, max_requests, window_size_in_millis):
        self.max_requests = max_requests
        self.window_size_in_millis = window_size_in_millis
        self.window_start = time.time() * 1000
        self.request_count = 0

    def allow_request(self):
        now = time.time() * 1000
        if now - self.window_start >= self.window_size_in_millis:
            self.window_start = now
            self.request_count = 0
        if self.request_count < self.max_requests:
            self.request_count += 1
            return True
        return False
```

实现便宜，一个计数器就够。问题在格子交界：上一格快结束时打满，下一格一开始又能立刻打满，短时间里能放到接近两倍。只做粗配额没问题；如果要求任意一段时间都不能超过上限，它就不够。

## 滑动窗口

任意时刻往回看一个窗口，这段里放行次数都不能超过上限。队列里记下每次放行的时间戳，新请求进来时先把过期的弹掉，再看长度。

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class SlidingWindowRateLimiter {
    private final int maxRequests;
    private final long windowSizeInMillis;
    private final Deque<Long> hits = new ArrayDeque<>();

    public SlidingWindowRateLimiter(int maxRequests, long windowSizeInMillis) {
        this.maxRequests = maxRequests;
        this.windowSizeInMillis = windowSizeInMillis;
    }

    public synchronized boolean allowRequest() {
        long currentTime = System.currentTimeMillis();
        // 移除过期的请求
        while (!hits.isEmpty() && currentTime - hits.peekFirst() > windowSizeInMillis) {
            hits.removeFirst(); // 移除最早的请求
        }
        // 检查请求计数是否超过限制
        if (hits.size() < maxRequests) {
            hits.addLast(currentTime);
            return true;
        } else {
            return false;
        }
    }
}
```

```python
import time
from collections import deque

class SlidingWindowRateLimiter:
    def __init__(self, max_requests, window_size_in_millis):
        self.max_requests = max_requests
        self.window_size_in_millis = window_size_in_millis
        self.hits = deque()

    def allow_request(self):
        now = time.time() * 1000
        while self.hits and now - self.hits[0] > self.window_size_in_millis:
            self.hits.popleft()
        if len(self.hits) < self.max_requests:
            self.hits.append(now)
            return True
        return False
```

过期判断用的是 `>`：刚好满窗口的那次请求还留在队列里。如果改成 `>=`，刚好满窗口就会被弹出。两种都行，差在区间开闭。

精度比固定窗口好，代价是每次放行都要存一个时间戳。量一大，内存会涨。

固定窗口看的是「当前这一格」，滑动窗口看的是「任意一段等长的过去」。

## 还可以怎么拓展

时间戳队列太占空间时，可以把窗口切成 N 个小格，每格只存计数。当前格加上上一格按时间比例加权，得到一个近似滑动窗口。生产里这种更常见。

如果目标不是「任意窗口严格不超过 N」，而是控制平均速率、同时允许一点突发，用令牌桶：按速率往桶里补令牌，请求来了先取令牌，桶容量就是突发上限。

如果更在意下游被匀速喂请求，用漏桶：请求进桶，按固定速率漏出。突发会被桶容量截住，或者改成排队。

单机锁换成 Redis 之后，固定窗口一般是 `INCR + EXPIRE`，滑动窗口用有序集合存时间戳再 `ZREMRANGEBYSCORE`，令牌桶要用 Lua 把补令和扣令做成一次原子操作。还得想清楚 key 是全局一份，还是按用户、按 API key 分开，以及多台机器的时钟别差太远。

`allowRequest` 本身只回答放不放行。接到网关或 LLM 代理时，前面还有鉴权，后面还有排队、超时、重试、降级。限住了是直接拒绝还是入队，流式响应怎么做背压，是产品策略，不是限流算法。

Java 和 Python 只是写法不同。选哪种算法，看你要卡死配额、允许突发，还是把出口抹平。
