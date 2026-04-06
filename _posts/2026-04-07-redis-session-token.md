---
layout: post
title: "Redis Session & Token 관리 — Spring Session + JWT Blacklist"
date: 2026-04-07
categories: [개발]
tags: [redis, session, jwt, keycloak]
---

## 왜 굳이 Redis 세션인가?

Tomcat에 내장 세션이 있는데 **왜 Redis까지 끌어오는가?** 이 질문에 답할 수 없으면 기술 선택의 근거가 없는 것이다.

### Tomcat 내장 세션으로 충분한 경우

- **단일 인스턴스** 배포 (WAS 1대)
- 세션 데이터가 소량이고 메모리 여유가 있음
- 배포/재시작 시 세션 유실을 허용할 수 있음 (로그인 다시 하면 됨)

이 조건이면 Redis 세션은 **오버엔지니어링**이다.

### Redis 세션이 필요해지는 시점

| 상황 | 문제 | Redis 세션이 해결하는 것 |
|------|------|----------------------|
| **다중 인스턴스 배포** (2대 이상) | 사용자가 서버 A에 로그인 → 로드밸런서가 서버 B로 라우팅 → 세션 없음 → 로그아웃됨 | 모든 인스턴스가 동일 Redis를 바라보므로 **세션 공유** |
| **무중단 배포 (Rolling/Blue-Green)** | 구 인스턴스 종료 시 해당 인스턴스의 세션 소멸 | Redis에 세션이 있으므로 인스턴스 교체와 무관하게 세션 유지 |
| **Sticky Session의 한계** | 로드밸런서에서 같은 서버로 고정 라우팅 → 특정 서버에 부하 집중, 서버 장애 시 세션 유실 | Sticky 없이도 세션 공유 가능 → 균등 분배 |

### keycloak-spring-security에서의 판단

현재 상황:
- 보안 라이브러리로 **여러 프로젝트에 배포**됨
- 사용처마다 단일/다중 인스턴스 환경이 다름
- 라이브러리이므로 **사용자가 선택**할 수 있어야 함

**결정적 이유 — 백채널 로그아웃:**
- Keycloak의 Back-Channel Logout은 **Keycloak → 애플리케이션 서버로 로그아웃 요청**을 보내는 구조
- 서버가 이 요청을 받으면 **해당 사용자의 세션을 찾아서 무효화**해야 한다
- Tomcat 내장 세션이면? → 인스턴스 재시작/교체 시 세션 소실 → **백채널 로그아웃 요청이 와도 매칭할 세션이 없음** → 로그아웃 안 됨
- Redis 세션이면? → 인스턴스와 무관하게 세션이 유지 → 백채널 로그아웃 정상 동작

이것은 "편의" 수준이 아니라 **보안 기능이 동작하지 않는** 문제이므로, 다중 인스턴스 환경에서 Redis 세션은 사실상 필수.

따라서:
- 기본값은 Tomcat 세션 (Redis 없이 동작)
- Redis 의존성을 추가하면 자동으로 Redis 세션으로 전환
- Redis 없이 쓰는 사람에게 에러를 던지지 않도록 **의존성 누락 시 가이드** 필요

> **면접 포인트:** "왜 Redis 세션을 도입했나요?" → "Keycloak 백채널 로그아웃이 서버 측 세션을 찾아 무효화하는 구조입니다. Tomcat 세션은 인스턴스 재시작 시 소실되어 로그아웃이 동작하지 않습니다. 보안 기능이 깨지는 문제이므로 다중 인스턴스 환경에서는 Redis 세션이 필수였고, 라이브러리 특성상 Redis 유무에 따라 자동 전환되도록 설계했습니다."

## Spring Session + Redis 연동 원리

`SessionRepositoryFilter`가 `HttpServletRequest`를 래핑하여, `request.getSession()` 호출 시 Tomcat 내장 세션 대신 **Redis 기반 `RedisIndexedSessionRepository`**에서 세션을 조회/생성한다.

**Redis 키 구조:**
- `spring:session:sessions:<id>` — 세션 데이터 (Hash)
- `spring:session:sessions:expires:<id>` — 만료 추적용 (String)
- `spring:session:expirations:<rounded-timestamp>` — 특정 시각 만료 세션 목록 (Set)

## @EnableRedisHttpSession

자동 구성 항목:
1. `RedisIndexedSessionRepository` 빈
2. `SessionRepositoryFilter` 빈 (서블릿 필터 체인 최상단)

| 속성 | 설명 |
|------|------|
| `maxInactiveIntervalInSeconds` | 세션 유효시간 (기본 1800초 = 30분) |
| `redisNamespace` | Redis 키 접두사 (기본 `spring:session`) |
| `flushMode` | `ON_SAVE`(요청 끝에 저장) / `IMMEDIATE`(즉시 저장) |
| `cleanupCron` | 만료 세션 정리 크론 |

> Spring Boot에서는 `spring.session.store-type=redis`만 설정하면 자동 구성.

## JWT Refresh Token Redis 저장

| Key 패턴 | 용도 |
|----------|------|
| `refresh_token:{userId}` | 1인 1토큰, 다른 기기 로그인 시 기존 무효화 |
| `refresh_token:{tokenValue}` | 토큰 값으로 빠른 검증 |
| `refresh_token:{userId}:{deviceId}` | 멀티 디바이스 지원 |

- TTL: `SET key value EX <seconds>` (7~30일)
- **Refresh Token Rotation**: 재발급마다 새 토큰 발급 + 이전 토큰 삭제 → 탈취 시 Reuse Detection

## Access Token Blacklist

로그아웃 시 Access Token을 Redis에 블랙리스트 등록:

```
Key:   blacklist:{jti}
Value: "logout"
TTL:   token.exp - 현재시각 (초)   ← 만료된 토큰은 어차피 검증 실패
```

- Access Token 수명이 짧을수록(5~15분) 블랙리스트 부담 감소
- 조회 O(1), 성능 부담 거의 없음

## Session Clustering vs JWT

| 관점 | Session (Redis) | JWT (Stateless) |
|------|----------------|-----------------|
| 상태 저장 | 서버 측 (Redis) | 클라이언트 측 |
| 즉시 무효화 | 세션 삭제로 즉시 | 불가 (블랙리스트 필요) |
| 확장성 | Redis가 SPOF 가능 | 수평 확장 용이 |
| 네트워크 비용 | 매 요청 Redis 조회 | 서명 검증만 |
| 토큰 크기 | Session ID (수십 바이트) | JWT (수백~수 KB) |

**실무:** Access Token(JWT, 단수명) + Refresh Token(Redis 저장) **하이브리드**가 가장 흔한 패턴
