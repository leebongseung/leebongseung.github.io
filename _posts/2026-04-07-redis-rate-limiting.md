---
layout: post
title: "Redis Rate Limiting — Fixed Window부터 Token Bucket까지"
date: 2026-04-07
categories: [개발]
tags: [redis, rate-limiting, lua]
---

## 알고리즘 비교

### Fixed Window Counter

시간을 고정 윈도우로 분할 (예: 1분), 윈도우마다 카운터 유지.

```
key = "rate:" + clientId + ":" + (currentTime / windowSize)
count = INCR key
if count == 1 then EXPIRE key windowSize
if count > limit then REJECT
```

**경계 시점 문제:** 1분 윈도우/100건 제한 시, 00:00:59에 100건 + 00:01:00에 100건 → 2초간 200건 허용

### Sliding Window Log

Sorted Set에 각 요청 타임스탬프를 기록:

```
ZREMRANGEBYSCORE key 0 (now - windowSize)   -- 오래된 기록 삭제
ZCARD key                                    -- 현재 윈도우 내 요청 수
ZADD key now requestId                       -- 요청 추가
```

정확하지만 **모든 요청 기록 저장 → 메모리 비용 큼**.

### Sliding Window Counter (하이브리드) — 실무 가장 많이 사용

이전 윈도우 + 현재 윈도우 카운터를 가중 평균으로 추정:

```
추정 = (이전 카운터 × 이전 윈도우 겹치는 비율) + 현재 카운터
예: 80 × 0.67 + 30 = 83.6 → 84건
```

메모리 효율적 + Fixed Window보다 정확. Cloudflare가 사용하는 방식.

### Token Bucket — 버스트 허용

토큰이 일정 속도로 보충되고, 요청마다 토큰 1개 소비. 버킷에 토큰이 누적되면 **일시적 버스트 허용**.

AWS API Gateway, Stripe API 등 대부분의 상용 API가 사용.

## 알고리즘 비교 요약

| 알고리즘 | 정확성 | 메모리 | 버스트 | 복잡도 |
|----------|--------|--------|--------|--------|
| Fixed Window | 낮음 (경계 문제) | 매우 높음 | 의도치 않은 | 매우 단순 |
| Sliding Window Log | 매우 높음 | 낮음 | 없음 | 보통 |
| Sliding Window Counter | 높음 | 높음 | 없음 | 보통 |
| Token Bucket | 높음 | 높음 | 의도적 허용 | 보통 |

## Lua 스크립트가 필요한 이유

**Race Condition 문제:**
```
Client A: GET counter → 99
Client B: GET counter → 99
Client A: INCR → 100 (허용)
Client B: INCR → 101 (허용 — 제한 초과!)
```

Redis는 단일 명령은 원자적이지만, **여러 명령의 조합은 원자적이지 않다.** Lua 스크립트는 단일 명령처럼 원자적으로 실행되어 읽기-판단-쓰기를 안전하게 처리.

```lua
-- Sliding Window Rate Limit (Lua)
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

redis.call('ZREMRANGEBYSCORE', key, 0, now - window)
local count = redis.call('ZCARD', key)

if count < limit then
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window / 1000)
    return 1  -- 허용
else
    return 0  -- 차단
end
```

**MULTI/EXEC 대비 장점:** 트랜잭션은 읽은 값 기반 조건부 로직 불가 (WATCH 필요 + 충돌 시 재시도), Lua는 자연스럽게 조건부 로직 가능.

## Spring 생태계 라이브러리

| 관점 | Bucket4j | Resilience4j | SCG Rate Limiter |
|------|----------|-------------|-----------------|
| 알고리즘 | Token Bucket | Semaphore 기반 | Token Bucket |
| 분산 지원 | Redis 등 지원 | **단일 인스턴스만** | Redis 지원 |
| 적용 위치 | 필터/인터셉터/메서드 | 메서드 (AOP) | Gateway 라우트 |
| 적합 시나리오 | API 서버 Rate Limit | 서비스 간 호출 보호 | API Gateway 레벨 |
