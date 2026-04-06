---
layout: post
title: "Redis 분산 락 — SET NX부터 Redisson, Redlock까지"
date: 2026-04-07
categories: [개발]
tags: [redis, 분산락, redisson]
---

## 왜 분산 락이 필요한가

단일 JVM에서는 `synchronized`, `ReentrantLock`으로 충분하지만, **다중 서버(스케일아웃)** 환경에서는:

- JVM 락은 프로세스 내에서만 유효 → 서버 A의 락이 서버 B를 차단하지 못함
- 재고 차감, 포인트 차감, 선착순 쿠폰 등에서 **race condition** 발생
- **모든 서버가 공유할 수 있는 외부 저장소 기반의 락**이 필요 → Redis

**분산 락의 두 가지 목적 (Kleppmann 분류):**

| 목적 | 설명 | 실패 시 결과 |
|------|------|-------------|
| **효율성** | 중복 작업 방지 | 비용 낭비 (이중 이메일 등) |
| **정확성** | 데이터 손상 방지 | 금전 손실, 데이터 불일치 |

## 구현 방법

### SETNX + EXPIRE의 문제 (원자적이지 않음)

```
SETNX lock:order:123 "server-1"   # 키가 없을 때만 SET
EXPIRE lock:order:123 10          # TTL 설정
```

**문제:** `SETNX` 성공 후 `EXPIRE` 전에 서버 크래시 → TTL 없는 영구 락 (Deadlock)

### SET NX PX — 올바른 원자적 락 획득

```
SET lock:order:123 <unique-value> NX PX 10000
```

| 옵션 | 의미 |
|------|------|
| `NX` | 키가 없을 때만 설정 |
| `PX 10000` | 만료 시간 10초 |
| `<unique-value>` | UUID 등 클라이언트 고유값 |

**단일 원자적 연산**으로 키 설정과 TTL 설정이 동시에 처리된다.

### 락 해제 — Lua 스크립트 필수

단순 `DEL`은 **본인이 아닌 다른 클라이언트의 락도 삭제**할 수 있다:

1. 클라이언트 A 락 획득 (TTL 10초)
2. A의 작업이 12초 → 락 만료
3. 클라이언트 B가 새로 락 획득
4. A가 `DEL` 실행 → **B의 락 삭제!**

```lua
-- 올바른 해제: 값이 일치할 때만 삭제
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

Redis는 Lua 스크립트를 **단일 명령처럼 원자적으로** 실행하므로, GET과 DEL 사이에 다른 명령이 끼어들 수 없다.

## Redisson RLock

### Watchdog (자동 갱신)

| 항목 | 값 |
|------|---|
| 기본 lockWatchdogTimeout | 30초 |
| 갱신 주기 | TTL의 1/3 (기본 10초마다) |
| 동작 조건 | leaseTime 미지정 시에만 |
| 중단 조건 | `unlock()`, 클라이언트 종료, 인터럽트 |

**흐름:** `lock()` → Watchdog 활성화 → 10초마다 TTL 30초로 재설정 → 클라이언트 죽으면 갱신 중단 → 30초 후 자동 만료

### tryLock vs lock

| 비교 | `lock()` | `tryLock(waitTime, leaseTime)` |
|------|----------|-------------------------------|
| 블로킹 | 무한 블로킹 | 타임아웃 후 false 반환 |
| 반환값 | void | boolean |
| 데드락 위험 | 높음 | 타임아웃으로 방지 |
| **실무 권장** | 비권장 | **권장** |

```java
RLock lock = redissonClient.getLock("lock:order:123");
try {
    if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {
        // 비즈니스 로직
    }
} finally {
    lock.unlock();
}
```

### 재진입(Reentrant) 락

Redis Hash 구조 `{lock-name} → {thread-id: count}` 사용:

```java
lock.lock();   // count = 1
lock.lock();   // count = 2 (같은 스레드 → 재진입)
lock.unlock(); // count = 1
lock.unlock(); // count = 0 → 실제 해제
```

메서드 A가 락을 잡고 내부에서 메서드 B를 호출하는데, B도 같은 락이 필요한 경우에 유용.

## Redlock 알고리즘

**구성:** N개(보통 5개)의 **독립적인** Redis 마스터 노드 (복제 없음)

**절차:**
1. 현재 시간 기록
2. N개 노드 모두에 `SET key value NX PX ttl` 시도 (짧은 타임아웃)
3. **과반수(N/2+1)** 이상 성공 + 유효 시간 > 0 → 락 획득
4. 실패 시 **모든 노드에서 해제**

### Martin Kleppmann의 비판

| 비판 | 내용 |
|------|------|
| **GC Pause** | 클라이언트 A가 락 획득 후 Full GC → 락 만료 → B 획득 → A의 GC 종료 → A, B 동시 접근 |
| **클럭 의존** | NTP 점프로 TTL이 예상보다 빨리 만료 → 두 클라이언트가 동시에 과반수 획득 |
| **Fencing Token 부재** | 단조 증가 토큰이 없어 만료된 락의 지연된 쓰기를 차단 불가 |

**Kleppmann 결론:** 효율성 목적이면 단일 Redis로 충분(Redlock 과잉), 정확성 목적이면 Redlock 불충분(ZooKeeper + fencing token 권장)

### Antirez(Redis 창시자)의 반박

| 반박 | 내용 |
|------|------|
| GC 문제는 **모든 분산 락**에 해당 | ZooKeeper도 GC 중 heartbeat 불가 → 세션 만료 |
| 클럭은 **관리 가능** | 단조 클럭 사용, NTP slew mode로 점프 방지 |
| Fencing 가능하면 **CAS로 락 없이도 안전** | 순환 논리에 가깝다 |

### 실무 합의

| 상황 | 권장 방식 |
|------|----------|
| 효율성 목적 (중복 방지) | 단일 Redis + Redisson RLock |
| 정확성 목적 (금융, 결제) | **Redisson RLock + DB 방어 병행** (Optimistic Lock, Unique Constraint) |
| 강한 정확성 (절대 불가) | ZooKeeper/etcd + fencing token |

> Redis 분산 락 = "1차 방어선", DB 제약 조건 = "최종 방어선"

## 주의사항

**Fencing Token:** 락 획득 시마다 단조 증가 정수를 발급. 저장소가 이전보다 작은 토큰의 쓰기를 거부. DB의 `@Version` 기반 Optimistic Lock이 사실상 같은 역할.

**TTL 설정 가이드:**
- 너무 짧으면 → 작업 중 만료 → 동시 접근 위험
- 너무 길면 → 장애 시 락 해제 대기 → 서비스 지연
- Redisson 사용 시 → leaseTime 미지정 + Watchdog에 맡기는 것이 가장 안전
- Redisson 미사용 시 → 99 퍼센타일 소요 시간의 3배
