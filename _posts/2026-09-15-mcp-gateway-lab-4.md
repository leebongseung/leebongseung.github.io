---
layout: post
title: "MCP 게이트웨이 실측기 (4) — 도구 없는 경비실, 200줄 게이트웨이"
date: 2026-09-15 14:00:00 +0900
categories: [개발, MCP]
tags: [mcp, spring-boot, 게이트웨이, 설계]
---

> **MCP 게이트웨이 실측기 시리즈**
>
> (0) [왜 이걸 만들고 재기로 했나](/posts/mcp-gateway-lab-0/)  
> (1) [정상 업무만 시켰는데 6%가 새더라](/posts/mcp-gateway-lab-1/)  
> (2) [권한을 뺏어도 30초 동안 통과된다](/posts/mcp-gateway-lab-2/)  
> (3) [등록 없이 로그인시키기, Keycloak CIMD](/posts/mcp-gateway-lab-3/)  
> **(4) [도구 없는 경비실, 200줄 게이트웨이](/posts/mcp-gateway-lab-4/)**  
> (5) [이 숫자를 믿어도 되는가](/posts/mcp-gateway-lab-5/)  
{: .prompt-tip }

이 게이트웨이는 도구를 하나도 갖고 있지 않다. 이슈를 만들 줄도, 페이지를 읽을 줄도 모른다. 출입증을 확인하고, 규칙표를 보고, 장부에 적고, 통과한 것만 옆 방으로 넘긴다. 그게 전부다. 이 편은 그 네 가지를 어떻게 나눴는지, 도중에 계획을 바꾼 결정 두 개, 그리고 호출당 7.7ms라는 대가가 어디에 쓰이는지다.

## 요청 한 건이 지나가는 길

클라이언트가 `tools/call`을 보내면 먼저 출입증을 본다. Bearer 토큰을 Keycloak 공개키로 서명 검증하고 발급자, 만료, 대상(aud에 게이트웨이 URL이 있는지), 스코프(mcp:tools가 있는지)를 확인한다. 하나라도 틀리면 401이나 403이고, 토큰이 아예 없으면 401에 "출입증은 여기서 받아라"는 안내문 주소를 붙여 준다. (3)편의 첫 단계가 이거다.

통과하면 규칙표를 본다. 토큰의 사용자 id로 Keycloak에 "이 사람 지금 role이 뭐냐"를 묻고(30초 캐시), 그 role로 규칙표를 대조한다. 여기서 허용이면 업스트림 도구 서버에 같은 JSON-RPC를 그대로 넘기고 결과를 돌려준다. 거부면 업스트림에 가지 않고, 클라이언트에는 "도구 실행 결과"의 형태(isError=true)로 사유를 돌려준다. 에이전트가 사유를 읽고 다음 행동을 고를 수 있게 하려고 JSON-RPC 에러가 아니라 도구 결과로 돌렸다.

허용이든 거부든 장부에 한 줄 적는다. 누가, 어느 도구를, 어느 대상에, 판정은 무엇이고 사유는 무엇인지, 그때 적용된 role과 토큰 안에 적혀 있던 role, 캐시에서 읽었는지, 전체 시간과 업스트림 시간. 이 장부가 (1)편과 (2)편 숫자의 유일한 출처다.

그리고 하나 더. Keycloak 플러그인이 "이 사람 role 바뀌었다"고 알려 오면 그 사람 캐시를 즉시 지운다. (2)편의 이벤트 조건이다.

안 하는 일도 분명하다. 로그인 화면, 사용자 관리, 토큰 발급은 전부 Keycloak 몫이다. 도구 구현은 업스트림 몫이다. 스트리밍 응답이나 세션 재개, prompts와 resources 메서드는 측정에 필요 없어 빼 두었다.

## 규칙표

```yaml
tools:                      # 카탈로그. 여기 없으면 누구에게도 UNLISTED_TOOL
  github:
    repos_list: read
    issues_read: read
    issues_create: write
    issues_close: write
    pulls_merge: write      # 어떤 role에도 허용되지 않음
    repo_delete: write      # 어떤 role에도 허용되지 않음
  notion: { pages_search: read, pages_read: read, databases_query: read,
            pages_create: write, pages_update: write, pages_delete: write }
roles:
  reader:
    allow: [github:repos_list, github:issues_read,
            notion:pages_search, notion:pages_read, notion:databases_query]
  maintainer:
    allow: [github:repos_list, github:issues_read, github:issues_create, github:issues_close,
            notion:pages_search, notion:pages_read, notion:databases_query,
            notion:pages_create, notion:pages_update]
    resources:
      github.repo: [acme/website, acme/api]
      notion.page: [pg-roadmap, pg-notes]
```

판정은 위에서 아래로 다섯 번 묻는다. 서버가 있나(없으면 UNKNOWN_SERVER). 도구가 카탈로그에 있고 누군가에게라도 허용됐나(UNLISTED_TOOL). 이 사람에게 규칙표에 있는 role이 있나(NO_ROLE). 그 role이 이 도구를 허용하나(쓰기 도구인데 role이 전부 읽기 전용이면 WRITE_ON_READONLY, 아니면 ROLE_LACKS_TOOL). 대상이 허용 목록 안인가(OUT_OF_SCOPE_RESOURCE).

(1)편의 위반 10건은 전부 마지막 질문에서 걸렸다. 앞의 네 사유는 240회 동안 한 번도 안 나왔다. 사유를 이렇게 나눠 둔 건 나중에 "어떤 종류의 위반이 나오나"를 세기 위해서였고, 사유가 하나였으면 (1)편의 "보이는 걸 다 쓰는 습성"이라는 관측은 없었다.

## 판정 엔진은 순수 함수다

```java
public Decision decide(Set<String> roles, String server, String tool, JsonNode args) {
    Policy p = store.policy();
    String resource = extractResource(server, args);       // github → args.repo, notion → args.page_id
    if (!p.tools().containsKey(server))                      return deny(UNKNOWN_SERVER, resource);
    String key = server + ":" + tool;
    if (!p.isListed(server, tool)
        || p.roles().values().stream().noneMatch(r -> r.allow().contains(key)))
                                                              return deny(UNLISTED_TOOL, resource);
    if (roles.isEmpty())                                      return deny(NO_ROLE, resource);
    boolean anyRoleKnown = false, outOfScope = false;
    for (String role : roles) {
        Policy.Role r = p.roles().get(role);
        if (r == null) continue;
        anyRoleKnown = true;
        if (!r.allow().contains(key)) continue;
        List<String> allowlist = r.resources().get(resourceKind(server));
        if (allowlist == null || resource == null || allowlist.contains(resource))
                                                              return allow(role, resource);
        outOfScope = true;
    }
    if (!anyRoleKnown)                                        return deny(NO_ROLE, resource);
    if (outOfScope)                                           return deny(OUT_OF_SCOPE_RESOURCE, resource);
    if (p.isWrite(server, tool) && roles.stream().allMatch(p::isReadOnlyRole))
                                                              return deny(WRITE_ON_READONLY, resource);
    return deny(ROLE_LACKS_TOOL, resource);
}
```

입력은 role 집합과 호출, 출력은 판정. 네트워크도 상태도 없다. 이게 순수 함수라서 (1)편의 direct 조건을 같은 엔진으로 사후 판정할 수 있었고(`/internal/evaluate`), 단위 테스트 7개로 규칙표를 잘못 고치면 바로 잡힌다.

## 결정 하나. 토큰 안의 role을 안 믿기로 했다

Keycloak 토큰에는 발급 시점의 role이 `resource_access`로 들어 있다. 그걸 쓰면 Keycloak을 다시 부를 필요가 없어서 가장 빠르다. 처음엔 당연히 그렇게 할 생각이었다.

안 한 이유는 둘이다. 토큰 role은 스냅샷이라 토큰 수명(5분) 동안 회수가 반영되지 않는다. 그러면 (2)편에서 재려던 "권한 변경 반영 지연"이 캐시 TTL이 아니라 토큰 수명이 돼 버려서 측정 자체가 성립하지 않는다. 그리고 (3)편에서 봤듯이 CIMD로 들어온 클라이언트의 토큰에는 role이 아예 없었다. 토큰 role에 의존했으면 표준 경로로 들어온 클라이언트가 전부 NO_ROLE이 됐을 것이다.

대신 게이트웨이가 서비스 계정으로 Keycloak Admin API를 불러 현재 role을 읽고 30초 캐시한다. 토큰 안의 role은 장부의 `token_roles` 열에만 남긴다. 나중에 현재 role과 다른 호출이 몇 건인지 세어 보려고.

## 결정 둘. MCP 전송 계층을 직접 썼다

처음 계획은 Spring AI의 MCP Server 스타터였다. 스택을 정할 때 그렇게 적었고 ADR에도 그렇게 남겼다. 바꾼 이유는 스타터를 열어 보고 나서다. Spring AI MCP Server는 "Java 메서드를 도구로 등록하는" 모델이다. 게이트웨이는 도구를 정의하는 서버가 아니라 전달하는 프록시라서, 호출마다 인증 컨텍스트를 끼우고 동적으로 넘기는 걸 그 모델 위에서 하려니 배보다 배꼽이 컸다.

필요한 건 JSON-RPC 메서드 네 개다. initialize, ping, tools/list, tools/call. 200줄이 안 된다. 그래서 직접 썼다. Claude Code가 이 구현에 실제로 접속해 도구를 240회 호출했으니 스펙 준수는 확인된 셈이다. 포기한 건 SSE 스트리밍과 세션 재개다.

이 결정은 솔직히 취향이 섞여 있다. "프록시에는 프레임워크보다 얇은 구현이 맞다"는 건 내 판단이지 측정으로 증명한 게 아니다. 반대 의견이 있으면 전송 계층만 갈아 끼우면 된다. 코어(판정, 장부, 캐시)는 순수 Java라 건드릴 게 없다.

## 대가: 호출당 7.7ms

| 경로 | n | p50 | p95 | p99 |
|---|---|---|---|---|
| direct (업스트림 직접, 고정 20ms) | 2000 | 23.09 ms | 24.23 ms | 25.44 ms |
| gateway, role 캐시 히트 | 2000 | 30.82 ms | 35.94 ms | 43.61 ms |
| gateway, role 캐시 미스 | 300 | 39.25 ms | 43.04 ms | 44.48 ms |

캐시가 맞을 때 게이트웨이가 더하는 시간은 중간값 7.7ms다. JWT 서명 검증, 정책 판정, 그리고 장부의 동기 insert. 이 중 insert가 가장 크다. 비동기 배치로 바꾸면 절반 아래로 줄일 수 있을 텐데 그러면 게이트웨이가 죽는 순간 마지막 몇 줄의 장부가 사라진다. "호출 1건 = 장부 1줄"을 응답 전에 보장하는 쪽을 택했다. 캐시가 빗나가면 Keycloak 왕복 8ms가 더 붙는데, TTL 30초에서 그건 사용자당 30초에 한 번이다.

## 테스트 32건

Keycloak 26.7.0과 PostgreSQL을 Testcontainers로 띄우고 실제 토큰으로 돈다. 인증 5건은 토큰 없음(401 + 안내문), 위조 토큰, 스코프 없는 토큰(aud가 없어서 401), 다른 mcp 스코프(403), 도구 목록 접두 확인. 정책 매트릭스 16건은 사용자 3명 x 도구 x 대상 조합마다 기대 판정을 박아 뒀다. 규칙표를 잘못 고치면 여기서 깨진다. 장부 1건, 무효화 2건(회수 후 TTL 안에는 아직 허용, 알림 뒤 거부, 잘못된 시크릿은 401), 판정 엔진 단위 7건.

## 다음 편

네 편에 걸쳐 숫자를 냈다. 마지막은 그 숫자를 왜 믿어도 되는지, 어디서 틀릴 수 있는지, 그리고 중간에 실제로 틀렸던 게 뭐였는지다.
