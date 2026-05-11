---
layout: post
title: "Infinispan 기본 개념 정리 — 분산 캐시 모델과 클러스터링"
date: 2026-05-12 00:02:00 +0900
categories: [개발, Keycloak]
tags: [infinispan, 분산캐시, keycloak, jgroups, 클러스터링]
---

[지난 글](/posts/keycloak-high-availability/)에서 Keycloak 클러스터링의 캐시 동기화를 담당하는 라이브러리로 **Infinispan**이 등장했습니다. 그때는 "노드를 3대로 잡은 이유"에 집중했고, 그 핵심에 있던 `num_owners=2` 정도만 짚고 넘어갔는데요. 이번 글에서는 Infinispan 자체를 한 단계 깊이 들여다보려고 합니다.

## Infinispan이란

한 줄로 요약하면 **자바 기반 인메모리 데이터 그리드(In-Memory Data Grid)**입니다. 여러 노드의 메모리를 묶어서 하나의 분산 캐시 공간처럼 다룰 수 있게 해주는 라이브러리이고, 단순 캐시를 넘어 트랜잭션·검색·이벤트 리스닝 같은 기능까지 제공합니다.

Keycloak이 이 Infinispan을 자기 안에 임베드해서, 분산 세션·로그인 실패 카운트·로컬 캐시 무효화 메시지 전파 같은 클러스터링 핵심 동작을 위임하고 있다고 보면 됩니다.

## 캐시 모드 — 4가지

Infinispan은 4가지 캐시 모드를 제공합니다. 각각 어떤 상황에 쓰는지가 다릅니다.

| 모드 | 동작 방식 | 특징 |
| --- | --- | --- |
| **Local** | 클러스터링 없이 단일 노드 내부에만 저장 | 가장 빠름, 노드 간 공유 X |
| **Invalidation** | 변경 시 다른 노드에 "그 데이터 무효" 메시지만 전파 | DB 같은 영속 저장소가 따로 있을 때 사용. 메시지 크기 작음 |
| **Replicated** | 모든 노드가 모든 키를 보유 | 데이터 일관성 강력, 노드 수 증가 시 비용 급증 |
| **Distributed** | `numOwners` 개수만큼만 복제, 나머지 노드는 비-소유자 | 확장성과 내결함성의 균형. 분산 캐시의 본령 |

Keycloak이 클러스터링에 쓰는 분산 캐시(sessions, loginFailures 등)는 **Distributed 모드**, 로컬 캐시 무효화 전파용 `work` 캐시는 **Replicated 모드**로 동작합니다.

## `num_owners`와 데이터 분산 동작

Distributed 모드의 핵심 파라미터가 **`num_owners`**입니다. "같은 데이터를 클러스터 내 몇 개 노드에 복제할 것인가"를 결정합니다. Keycloak 기본값은 **2**.

- `num_owners=1` → 각 데이터를 한 노드만 보유. 그 노드 죽으면 데이터 손실
- **`num_owners=2`** → 두 노드에 복제. 한 노드 죽어도 다른 한쪽에서 조회 가능 (기본값)
- `num_owners=N` (전체 노드 수) → Replicated와 사실상 동일

즉, Distributed 모드는 **전체 복제(Replicated)와 무복제(Local) 사이의 스펙트럼**이고, `num_owners`가 그 위치를 결정합니다. 노드 수가 늘어나도 복제 노드 수가 고정되므로 **선형 확장(linear scalability)** 이 가능합니다.

## 노드 디스커버리 — JGroups

Infinispan은 노드 간 통신·디스커버리에 **JGroups**라는 별도 라이브러리를 사용합니다. "클러스터에 어떤 노드들이 살아있는가"를 추적하는 방식이 여기서 결정되는데, 환경에 따라 여러 프로토콜을 선택할 수 있습니다.

| 디스커버리 프로토콜 | 동작 방식 | 적합 환경 |
| --- | --- | --- |
| `MPING` / `PING` | UDP 멀티캐스트 | 멀티캐스트 허용된 사내망 |
| `JDBC_PING2` | 공용 DB에 노드 정보 기록·조회 | 멀티캐스트가 막힌 환경, 동적 확장 |
| `DNS_PING` | DNS SRV 레코드로 노드 조회 | Kubernetes 등 컨테이너 환경 |
| **`TCPPING`** | 정적 노드 IP 목록 하드코딩 | 노드 IP가 고정된 온프레미스 |

우리 팀은 **`TCPPING`** 을 선택했습니다. 온프레미스 3대 고정 IP 구성이라 가장 단순하고, 추가 의존성(DB 테이블·DNS·K8s)이 없다는 이유였습니다. 동적 확장이 필요한 시점이 오면 `JDBC_PING`으로 갈아탈 여지를 남겨두는 정도로 정리했습니다.

`cache-ispn.xml`의 JGroups 스택은 대략 이런 모양이 됩니다 (IP는 예시).

```xml
<jgroups>
    <stack name="tcpping" extends="tcp">
        <TCPPING initial_hosts="<NODE1_IP>[<PORT>],<NODE2_IP>[<PORT>],<NODE3_IP>[<PORT>]"
                 port_range="0"
                 stack.combine="REPLACE"
                 stack.position="MPING" />
    </stack>
</jgroups>
```

Docker로 띄울 때는 `network_mode: host`를 걸어 컨테이너가 호스트 IP로 바인딩되도록 했습니다 (그래야 다른 노드에서 도달 가능한 IP로 JGroups가 동작합니다).

## Embedded vs Remote (Hot Rod)

Infinispan은 사용 방식이 두 가지로 나뉩니다.

**Embedded**
- 애플리케이션 프로세스 안에 라이브러리로 임베드되어 함께 실행됨
- 프로세스 내 직접 호출이라 **최고 성능**
- Keycloak 기본 구성이 이쪽

**Remote (Hot Rod)**
- 별도의 Infinispan 서버 클러스터를 띄우고, 애플리케이션이 **Hot Rod 프로토콜**로 원격 접근
- 자바 외에 C++, C#, Python, Node.js 등 다양한 언어 클라이언트 지원
- 캐시 인프라를 애플리케이션 라이프사이클과 분리할 수 있음

우리는 **Embedded**를 채택했습니다. 외부 Infinispan 클러스터를 따로 운영하지 않아도 되니 운영 컴포넌트가 줄어든다는 점이 컸습니다. 실제 `keycloak.conf`에는 다음 한 줄만 들어가면 됩니다.

```properties
cache=ispn
```

Custom Docker 이미지를 빌드할 때 `cache-ispn.xml`을 함께 넣어두는 식으로 클러스터 설정을 이미지에 박아 배포했습니다.

## 운영 시 마주칠 수 있는 이슈

분산 캐시 라이브러리답게, 운영하다 보면 분산 시스템 특유의 이슈가 시야에 들어옵니다.

- **Split-brain** : 네트워크 단절로 클러스터가 분리되었을 때, 각 파티션이 자기들끼리 동작하다 다시 합쳐질 때의 데이터 정합성 문제
- **Rebalancing 비용** : 노드 합류·이탈 시 데이터 재분배가 일어남. 그 동안 CPU·네트워크 부하 상승
- **TCPPING의 정적 IP 의존** : 노드 IP가 바뀌면 `cache-ispn.xml` 수정 + 이미지 재빌드 필요. 동적 확장이 필요한 시점이 오면 `JDBC_PING` 등으로 전환 고려
- **동기 vs 비동기 복제** : 동기는 안전하지만 느리고, 비동기는 빠르지만 일시적 불일치 가능 — 워크로드에 맞게 선택 필요
- **순차 기동 권장** : 3대를 동시에 띄우면 클러스터 형성이 불안정할 수 있음. 첫 노드 기동 후 cluster view 메시지 확인 → 다음 노드 기동 순서로 진행

## 정리

Keycloak 가용성 글에서 짚었던 `num_owners=2`나 `TCPPING` 같은 키워드들이 결국 Infinispan의 분산 캐시 모델 위에 올라와 있는 셈입니다. 핵심을 다시 요약하면:

- **Infinispan = 자바 인메모리 데이터 그리드** + JGroups 기반 클러스터링
- **Distributed 모드 + `num_owners`** 가 분산 캐시의 본령
- **Embedded / Remote** 두 사용 모델 중 Keycloak은 Embedded 채택
- 디스커버리는 환경에 따라 `MPING` / `JDBC_PING2` / `DNS_PING` / `TCPPING` 선택

다음에 Keycloak 클러스터링 글을 다시 읽을 때, 캐시 동기화·`num_owners`·`TCPPING`이 한층 더 익숙하게 읽힐 것 같습니다.

## 참고 자료

- [Infinispan Documentation](https://infinispan.org/documentation/)
- [Configuring Infinispan caches](https://infinispan.org/docs/stable/titles/configuring/configuring.html)
- [Keycloak — Configuring distributed caches](https://www.keycloak.org/server/caching)
