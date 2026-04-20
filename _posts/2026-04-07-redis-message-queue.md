---
layout: post
title: "Redis 메시지 큐 — List vs Pub/Sub vs Stream"
date: 2026-04-07
categories: [개발]
tags: [redis, stream, pubsub, kafka]
---

## List (LPUSH / BRPOP)

- `LPUSH queue msg` → `BRPOP queue timeout` (블로킹 FIFO)
- 다수 Consumer가 `BRPOP`하면 Redis가 하나에만 전달 (경쟁 소비)
- **ACK 없음** → Consumer가 처리 중 죽으면 메시지 유실
- **Reliable Queue 패턴**: `BRPOPLPUSH`로 "처리 중" 리스트에 이동 → 완료 후 삭제

## Pub/Sub

- `PUBLISH channel msg` / `SUBSCRIBE channel`
- **Fire-and-Forget**: 구독자 없으면 메시지 소멸, 연결 끊김 시 유실
- At-Most-Once 전달, 영속성 없음
- 실시간 알림, 캐시 무효화 이벤트에 적합

## Redis Stream

Redis 5.0 도입. Kafka의 핵심 개념을 Redis 내에서 구현한 **append-only 로그**.

```
XADD stream * field value             # 메시지 추가
XREADGROUP GROUP g consumer COUNT n    # Consumer Group에서 읽기
XACK stream group id                   # 처리 완료 확인
```

- **Consumer Group**: 같은 그룹 내 경쟁 소비, 다른 그룹은 팬아웃
- **PEL (Pending Entries List)**: 읽었지만 ACK 안 한 메시지 보관
- **XCLAIM**: 장애 Consumer의 미처리 메시지를 다른 Consumer에게 재할당
- 구독자 없어도 메시지 유지 (Pub/Sub와의 결정적 차이)

## 비교 요약

| 관점 | List Queue | Pub/Sub | Stream |
|------|-----------|---------|--------|
| 영속성 | 소비하면 삭제 | 없음 | append-only 보관 |
| Consumer Group | 불가 | 불가 | 지원 |
| ACK / 재처리 | 불가 | 불가 | XACK + XCLAIM |
| 전달 보장 | At-Most-Once | At-Most-Once | At-Least-Once |
| 적합 용도 | 간단한 작업 큐 | 실시간 알림 | 이벤트 소싱, 감사 로그 |

## Kafka vs Redis Stream

| 관점 | Kafka | Redis Stream |
|------|-------|-------------|
| 처리량 | 초당 수백만 | 초당 수십만 |
| 영속성 | 디스크 기반, 장기 보관 | 인메모리, MAXLEN 제한 |
| 파티셔닝 | 토픽 파티션 → 대규모 병렬 | 단일 스트림 |
| 운영 복잡도 | ZooKeeper/KRaft 필요 | Redis 있으면 추가 인프라 불필요 |
| 지연시간 | ms 단위 | sub-ms |
| 에코시스템 | Connect, Streams, KSQL | Redis 내에서만 |

> **실무 경험칙:** "Redis 이미 있고 하루 수백만 건 이하면 Redis Stream, 그 이상이거나 장기 보관 필요하면 Kafka"
