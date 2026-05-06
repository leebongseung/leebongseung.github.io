---
layout: post
title: "비밀번호 정책 다루기 — Keycloak 기본 정책과 Custom SPI"
date: 2026-05-07
categories: [개발, Keycloak]
tags: [keycloak, 비밀번호정책, password-policy, spi, 보안, 인증]
---

지난 글들에서 권한(인가) 영역을 다뤘다면, 이번 글은 인증 쪽으로 시선을 옮겨 봅니다. 그중에서도 **비밀번호 정책** — 사내 보안지침을 만족하면서 사용자 경험도 깨뜨리지 않으려면 어디까지가 Keycloak으로 되고, 어디부터가 직접 구현이었는지를 정리해 보려고 합니다.

## Keycloak이 기본으로 제공하는 비밀번호 정책

먼저 출발점부터. Keycloak은 Realm 설정의 **Authentication → Policies → Password Policy** 에서 다음 항목들을 기본 제공합니다.

| 정책 | 설명 | 입력값 예시 |
|---|---|---|
| **Minimum Length** | 최소 길이 | `9` → 9자 이상 |
| **Maximum Length** | 최대 길이 | `64` |
| **Digits** | 숫자 포함 최소 개수 | `1` → 숫자 1개 이상 |
| **Lowercase Characters** | 소문자 포함 최소 개수 | `1` |
| **Uppercase Characters** | 대문자 포함 최소 개수 | `1` |
| **Special Characters** | 특수문자 포함 최소 개수 | `1` |
| **Not Username** | 사용자명을 비밀번호로 사용 금지 | (토글) |
| **Not Email** | 이메일을 비밀번호로 사용 금지 | (토글) |
| **Regular Expression** | 정규식 기반 패턴 검증 | `^(?=.*[A-Z])(?=.*\d).+$` |
| **Expire Password** | 비밀번호 만료 일수 | `90` → 90일 후 만료 |
| **Not Recently Used** | 최근 N개 비밀번호 재사용 금지 | `5` → 최근 5개 재사용 불가 |
| **Not Recently Used (In Days)** | 최근 N일 내 사용한 비밀번호 재사용 금지 | `90` → 90일 내 재사용 불가 |
| **Password Blacklist** | 사전 등록된 비밀번호 차단 | 파일명 입력 |
| **Hashing Iterations** | 해시 반복 횟수 | `27500` (기본값) |
| **HashAlgorithm** | 해시 알고리즘 | `pbkdf2-sha256` (기본값) |

이 정도만 봐도 **상당수의 보안지침은 Keycloak 기본 기능으로 커버 가능** 합니다. 길이·복잡도·만료·이력 관리·해시 강도까지 모두 박혀 있죠.

## 우리 사내 보안지침이 요구한 것

문제는 그다음이었습니다. 사내 정보보호 규정이 요구하는 비밀번호 요건을 정리해 보면 다음과 같았습니다.

| 정책 항목 | 요구사항 |
|---|---|
| **길이** | 10자리 이상 |
| **복잡도** | 영문자(대/소), 숫자, 특수문자 **3종류 조합 8자 이상** |
| **변경 주기** | 주기적 변경 (3개월) |
| **재사용 금지** | 최근 사용한 N개의 비밀번호 재사용 금지 |
| **식별자 유사성 차단** | 사용자 ID와 비슷한 비밀번호 금지 |
| **개인정보 포함 차단** | 연속된 숫자, 생일·전화번호 등 **추측하기 쉬운 개인정보 사용 금지** |
| **로그인 실패 잠금** | 10회 실패 시 즉시 계정 차단 |
| **초기/임시 비밀번호** | 첫 로그인 시 지체 없이 변경 |
| **마스킹** | 입력/변경 시 화면 마스킹 처리 |
| **저장 암호화** | 비밀번호는 **복호화 불가능한 일방향 해시** 로 저장 |
| **유출 시 처리** | 유출 발생 시 즉시 변경 |

## Keycloak 기본 정책으로 어디까지 커버되나

위 요구사항을 Keycloak 기본 정책에 매핑해 보면 다음과 같이 갈립니다.

| 요구사항 | Keycloak 기본 기능 | 커버 여부 |
|---|---|---|
| 길이 10자 이상 | `Minimum Length: 10` | ✅ |
| 3종류 조합 8자 이상 | `Digits / Lowercase / Uppercase / Special: 1` | ✅ |
| 변경 주기 (3개월) | `Expire Password: 90` | ✅ |
| 재사용 금지 (개수/기간) | `Not Recently Used` / `Not Recently Used (In Days)` | ✅ |
| 일방향 해시 저장 | `HashAlgorithm: pbkdf2-sha256` | ✅ |
| ID 유사성 차단 | `Not Username` (완전 일치만 차단) | ⚠️ 부분 |
| **연속/반복 문자, 개인정보 포함 차단** | — | ❌ |
| **로그인 실패 시 계정 잠금** | Authentication → Brute Force Detection (별도 영역) | ⚠️ 분리 |
| **첫 로그인 시 비밀번호 변경 강제** | Required Action: `UPDATE_PASSWORD` (별도 영역) | ⚠️ 분리 |
| 입력 마스킹 | (클라이언트 UI 책임) | ⚠️ 외부 |
| 유출 시 즉시 변경 | (운영 절차) | — |

정리하면, **Password Policy 탭만으로는 보안지침을 100% 만족시킬 수 없었습니다**. 핵심 공백은 다음과 같았습니다.

- **연속/반복 문자, 전화번호 패턴 차단** — 기본 정책에 없음
- **ID와의 유사성 검사** — Keycloak은 *완전 일치* 만 막아줌. "ID + 1234" 같은 변형은 통과
- **영문자 합산 카운트** — 기본은 대/소문자가 분리되어 있어, "영문자 N개 이상" 같은 합산 룰 표현 불가

## 우리가 추가로 만든 것 — Custom Password Policy SPI

남은 공백은 Keycloak이 제공하는 **PasswordPolicyProvider SPI** 로 풀었습니다. 이 SPI는 정확히 *"기본 정책 외에 우리가 원하는 검증 규칙을 끼워 넣는 자리"* 입니다.

### 왜 Regular Expression 정책으로 안 풀었는가

Keycloak에는 `Regular Expression` 정책이 이미 있어서, 이론적으로는 정규식 하나로 어느 정도 표현이 가능합니다. 하지만 우리 케이스에서는 다음 이유로 부적합했습니다.

1. **사용자 컨텍스트가 필요했다** — "ID와 비슷한지", "사용자명·생일·사번이 포함됐는지" 는 비밀번호 문자열만 봐서는 판단할 수 없습니다. 검증 시점에 **사용자 정보를 함께 봐야** 했습니다.
2. **에러 메시지를 규칙별로 분리해야 했다** — "연속된 숫자입니다", "ID와 유사합니다" 처럼 어느 규칙에 걸렸는지 사용자에게 명확히 알려줘야 했는데, 정규식 하나로 묶으면 *"비밀번호 형식이 올바르지 않습니다"* 외에는 안내가 어렵습니다.
3. **복합 조건 표현의 한계** — 연속 문자(`1234`, `abcd`), 반복 문자(`aaaa`), 키보드 인접 문자(`qwer`)를 한 정규식에 모두 담는 건 가독성·유지보수 모두 끔찍해집니다.

### SPI 구조

`PasswordPolicyProvider` 와 `PasswordPolicyProviderFactory` 를 구현하고, `META-INF/services` 에 등록해 Keycloak이 부팅 시 정책 목록에 우리 정책을 노출하도록 했습니다.

```
src/main/java/.../password/
 ├─ AlphabetPolicyProvider.java                     // 영문자 합산 카운트
 ├─ NoPhoneNumberPatternsPolicyProvider.java        // 전화번호 패턴 차단
 ├─ NotContainsUsernamePolicyProvider.java          // username 부분 포함 차단
 ├─ NoRepeatedCharactersPolicyProvider.java         // 동일 문자 반복 차단
 ├─ NoSequentialCharactersPolicyProvider.java       // 연속 문자/숫자 차단
 └─ ...각 PolicyProviderFactory

src/main/resources/META-INF/services/
 └─ org.keycloak.policy.PasswordPolicyProviderFactory
```

핵심은 `validate(RealmModel realm, UserModel user, String password)` 메서드입니다. 여기서 **사용자 컨텍스트(username, attributes 등)에 자유롭게 접근**할 수 있기 때문에, ID 유사성 검사 같은 검증을 자연스럽게 풀 수 있었습니다.

### 우리가 추가한 정책 다섯 가지

| 정책명 | 무엇을 막는가 | Keycloak 기본의 어떤 한계를 메웠나 |
|---|---|---|
| **Alphabet** | 영문자 합산 N개 이상 포함 | 기본은 `Lowercase` / `Uppercase` 가 분리되어 있어 합산 룰 표현 불가 |
| **No Phone Number Patterns** | 전화번호 형태(`010-xxxx-xxxx` 등) 차단 | 기본 정책 자체가 없음 |
| **Not Contains Username** | username **부분 포함** 차단 (`bongseung123` 같은 변형 차단) | 기본 `Not Username` 은 *완전 일치* 만 차단 |
| **No Repeated Characters** | 동일 문자 N글자 이상 반복(`aaaa`, `1111`) | 기본 정책에 없음 |
| **No Sequential Characters** | 연속 문자/숫자(`1234`, `abcd`) | 기본 정책에 없음 |

특히 **Not Contains Username** 은, *Keycloak이 비슷한 이름의 정책을 이미 제공하지만 우리 보안지침을 만족시키기에는 동작이 약간 다른 케이스* 였습니다. 기본 `Not Username` 만 켜두면 *형식상*으로는 정책을 적용한 듯 보이지만, `myid1234` 같은 변형은 그대로 통과합니다. 이런 사례에서 SPI의 가치가 가장 분명하게 드러났습니다.

### 등록 후 운영 화면

빌드한 jar를 `providers/` 디렉토리에 넣고 Keycloak을 재기동하면, **Realm → Authentication → Password Policy** 의 *Add policy* 드롭다운에 우리가 만든 항목이 그대로 노출됩니다. 운영자는 코드를 보지 않고도 GUI에서 ON/OFF·임계값을 조정할 수 있게 됩니다.

이 방식의 장점은 명확했습니다.

- **Keycloak 표준 흐름 그대로** — 비밀번호 변경/등록 모든 경로에서 자동으로 검증됨
- **운영자가 GUI에서 끄고 켤 수 있음** — 코드 배포 없이 정책 적용
- **에러 메시지를 규칙별로 분리** — 사용자에게 어떤 규칙에 걸렸는지 정확히 안내

## 정리

이번 글에서 다룬 흐름을 다시 정리하면 다음과 같습니다.

1. Keycloak 기본 정책으로 길이·복잡도·만료·재사용·해시는 대부분 커버
2. 사내 보안지침의 **연속·반복 문자 차단, 전화번호 패턴 차단, username 부분 포함 차단, 영문자 합산 카운트** 는 기본 정책만으로 부족
3. `PasswordPolicyProvider` SPI 로 **다섯 가지** 커스텀 정책을 만들어 공백을 메움
4. 그 외 *로그인 실패 잠금* / *첫 로그인 시 강제 변경* 은 Password Policy 가 아닌 별도 영역(Brute Force Detection / Required Actions)으로 풀었음

---

다음 글에서는 시야를 한 단계 위로 올려, **인증 서버의 가용성을 어떻게 확보했는지** — 즉 Keycloak 자체가 단일 장애점이 되지 않도록 어떻게 운영했는지를 다뤄 보려고 합니다.
