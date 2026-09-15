---
layout: post
title: "MCP 게이트웨이 실측기 (3) — 등록 없이 로그인시키기, Keycloak CIMD"
date: 2026-09-15 13:00:00 +0900
categories: [개발, MCP]
tags: [mcp, keycloak, oauth2, cimd, pkce]
---

Keycloak 로그인 화면에 처음 보는 문구가 떴다. "The client's hostname is host.docker.internal." 이 클라이언트를 Keycloak에 등록한 적이 없다. 그런데 Keycloak은 이 클라이언트가 누구인지 알고 있었고, 무슨 권한을 요구하는지도 화면에 적어 놓았다. MCP 인가 스펙이 기존 OAuth 위에 얹은 두 가지 중 하나가 눈앞에서 작동하는 순간이었다.

## OAuth를 왜 건드려야 했나

보통의 OAuth 로그인은 이렇게 시작한다. 개발자가 인가 서버에 앱을 등록하고 client_id를 받는다. 그 값과 인가 서버 주소를 코드에 박아 둔다. 끝. 수십 년 잘 돌아간 방식이다.

MCP 환경에서는 이 전제가 둘 다 깨진다. 클라이언트(Claude Code 같은 에이전트 프로그램)는 사용자 수만큼 많고, 접속할 도구 서버는 수십 개다. 서버마다 개발자가 등록하고 설정하는 방식은 감당이 안 된다. 그리고 클라이언트가 어떤 도구 서버에 붙을지는 실행 시점에 정해진다. 인가 서버 주소를 미리 알 방법이 없다.

그래서 MCP 인가 스펙은 두 가지를 요구한다. 도구 서버가 "내 출입증은 저기서 받아라"를 알려 주는 안내문(Protected Resource Metadata, RFC 9728)을 두는 것. 그리고 클라이언트가 등록된 이름 대신 자기 소개 문서의 URL로 자신을 증명하는 것(Client ID Metadata Document, CIMD). 후자는 2025-11-25 판부터 필수가 됐고, Keycloak은 26.6.0(2026-04)에서 실험 기능으로 넣었다.

## 흐름을 한 번에 따라가 보면

| 단계 | 누가 → 누구 | 무엇 | 일반 OAuth2와 |
|---|---|---|---|
| 1 | 클라이언트 → 게이트웨이 | 토큰 없이 tools/list 요청 | 같음 |
| 2 | 게이트웨이 → 클라이언트 | 401 + `WWW-Authenticate: Bearer resource_metadata="…"` | **다름.** 안내문 위치를 알려 준다 |
| 3 | 클라이언트 → 게이트웨이 | 안내문 읽기. 인가 서버 주소와 요청 가능한 스코프 목록 | **다름.** 스스로 발견 |
| 4 | 클라이언트 → Keycloak | OIDC discovery. `client_id_metadata_document_supported: true` | 같음 |
| 5 | 클라이언트 → Keycloak (브라우저) | `/auth?client_id=http://…/client.json&scope=openid mcp:tools&code_challenge=…` | **다름.** client_id가 URL |
| 6 | Keycloak → 클라이언트 문서 | 그 URL을 GET 해서 이름, 콜백 주소, 공개 클라이언트 여부를 읽는다 | **다름.** DB 조회 대신 fetch |
| 7 | 사용자 | 로그인 + 동의 | 같음. 미등록이라 동의는 항상 뜬다 |
| 8 | 클라이언트 → Keycloak | `/token` (code + code_verifier) → `scope="openid mcp:tools"`, `aud="http://localhost:8090/mcp"` | 같음 |
| 9~10 | 클라이언트 → 게이트웨이 | tools/call + Bearer. 게이트웨이가 서명·iss·exp·aud·scope 검증, role 조회, 정책 판정, 기록 | 같음 |

정리하면 2~3과 5~6만 다르다. 나머지는 표준 Authorization Code + PKCE 그대로다. 바뀐 두 곳의 이유는 같다. 클라이언트가 사람보다 많아졌다는 것.

## 스코프는 어디서 토큰에 들어가나

이 부분이 처음 볼 때 가장 헷갈렸다. 클라이언트는 3단계에서 안내문을 읽고 "요청할 수 있는 스코프가 mcp:tools, mcp:prompts, mcp:resources구나"를 안다. 5단계에서 `scope=openid mcp:tools`로 요청한다. 8단계에서 토큰에 scope 클레임이 들어가고, 동시에 aud에 게이트웨이 URL이 박힌다.

aud가 박히는 건 Keycloak의 mcp:tools 스코프에 Audience 매퍼를 붙여 뒀기 때문이다. 매퍼의 "Included Custom Audience" 값이 게이트웨이 URL이다. 스코프를 요청하지 않으면 매퍼도 안 돌아서 aud가 없고, 게이트웨이는 aud 없는 토큰을 401로 돌려보낸다. 스코프 하나가 "무엇을 할 수 있나"와 "어느 서버용인가"를 동시에 정하는 구조다.

MCP 스펙은 원래 토큰이 특정 서버 전용임을 `resource` 파라미터(RFC 8707)로 표시하라고 한다. Keycloak은 이 파라미터를 아직 지원하지 않는다. Keycloak 가이드가 권하는 우회가 바로 이 스코프 + Audience 매퍼 조합이다.

## 소개 문서는 이렇게 생겼다

```json
{
  "client_id": "http://host.docker.internal:8095/client.json",
  "client_name": "mcp-authz-lab CIMD test client",
  "redirect_uris": ["http://localhost:8095/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none",
  "scope": "openid mcp:tools"
}
```

관리자가 Keycloak 콘솔에 클라이언트를 등록할 때 입력하는 항목이 그대로 들어 있다. 차이는 이걸 Keycloak DB에 넣는 대신 클라이언트가 자기 URL에 올려 두고, URL 자체를 이름표로 쓴다는 것이다. client_id 값이 문서 자기 주소와 같아야 하는 게 규칙이다. 자기 소개서에 자기 주소가 적혀 있어야 위조가 어렵다.

이 URL을 읽는 건 브라우저도 게이트웨이도 아니고 Keycloak 서버다. 여기서 한 번 막혔다. Keycloak은 Docker 컨테이너 안에서 돌고, 소개 문서는 호스트의 8095 포트에서 서비스된다. 컨테이너 안에서 localhost는 자기 자신이라, 문서 주소를 `host.docker.internal`로 써야 했다. 그런데 브라우저 콜백은 `localhost:8095`로 돌아온다. 같은 프로세스가 두 이름으로 불리는 셈이고, Keycloak의 CIMD 정책에는 "client_id와 콜백이 같은 도메인이어야 한다"는 옵션이 있어서 그걸 꺼야 했다. 로그인 화면의 "hostname is host.docker.internal"은 그 흔적이다.

## 실제로 돌렸다

사전 등록 없는 클라이언트를 파이썬 60줄로 만들었다. 하는 일은 둘이다. `/client.json`을 서비스하고, `/callback`으로 돌아오는 인증 코드를 받는다. 브라우저 로그인은 사람이 해야 하니 Chrome 자동화로 alice가 로그인하고 동의했다.

401에서 안내문 주소를 받고, 안내문에서 Keycloak을 찾고, discovery에서 `client_id_metadata_document_supported: true`를 확인하고, URL을 client_id로 로그인 화면을 열고, 동의하고, 콜백으로 코드를 받아 토큰으로 바꿨다. 토큰의 aud는 게이트웨이 URL, azp는 소개 문서 URL, scope는 openid mcp:tools. 그 토큰으로 tools/list를 부르니 도구 12개가 왔고, issues_create(acme/api)는 허용, repo_delete는 UNLISTED_TOOL로 거부됐다. 끝까지 통과했다.

하나 눈에 띈 게 있다. 이 토큰에는 preferred_username도 resource_access(role)도 없었다. profile과 roles 스코프를 요청하지 않았고, 동적 클라이언트에는 기본 스코프가 붙지 않기 때문이다. 게이트웨이가 토큰 role을 안 쓰고 Keycloak에서 현재 role을 조회하는 설계라서 판정은 정상이었다. 토큰 role을 믿는 설계였으면 표준 경로로 들어온 클라이언트가 전부 NO_ROLE이 됐을 것이다. (4)편의 설계 결정이 여기서 한 번 값을 했다.

> 실험 240회는 이 경로로 돌린 게 아니다. CIMD 흐름은 7단계에서 사람이 로그인해야 해서 자동 반복이 불가능하다. (1)(2)편의 실험은 측정용 probe 클라이언트의 password grant로 토큰을 받았다. 게이트웨이는 토큰이 어느 경로로 나왔는지 구분하지 않고 aud도 같은 스코프 매퍼가 넣으니 판정은 같다. 다만 "CIMD로 로그인한 클라이언트가 240회를 돌렸다"는 뜻은 아니다.
>
{: .prompt-warning }

## 밟은 함정 여섯 개

순서대로 밟았다. realm import JSON에 `clientScopes`를 넣으면 기본 스코프(profile, email, roles)가 생성되지 않아 토큰에 username과 role이 빠진다. mcp:* 스코프는 import가 아니라 Admin REST로 만들어야 했다. 같은 JSON에 주석 용도로 `_note` 키 하나 넣었다가 Keycloak이 미지 필드라며 기동 실패 루프에 빠졌다. 서비스 계정 클라이언트의 `fullScopeAllowed`가 false면 realm-management role을 줘도 토큰에 안 실려 Admin REST가 403을 낸다.

Keycloak 밖에서도 셋. Spring Boot 부모 POM 대신 BOM import를 쓰면 컴파일러의 `-parameters`가 꺼져 `@PathVariable` 이름 해석이 실패한다. WSL의 /mnt/c에서 실행 중인 jar는 Windows 파일 잠금 때문에 재패키징이 실패하니 빌드 전에 프로세스를 내려야 한다. Keycloak SPI 설정 키는 provider id에 하이픈이 있으면 `KC_SPI_EVENTS_LISTENER__GATEWAY_WEBHOOK__URL`처럼 이중 밑줄로 써야 한다.

## 다음 편

토큰은 받았다. 그 토큰을 받는 쪽, 게이트웨이는 무엇을 해야 하고 무엇을 하지 말아야 하나. (4)편은 도구를 하나도 갖지 않은 200줄짜리 경비실의 설계 결정이다.


---

**MCP 게이트웨이 실측기 시리즈**

- (0) [왜 이걸 만들고 재기로 했나](/posts/mcp-gateway-lab-0/)
- (1) [정상 업무만 시켰는데 6%가 새더라](/posts/mcp-gateway-lab-1/)
- (2) [권한을 뺏어도 30초 동안 통과된다](/posts/mcp-gateway-lab-2/)
- **(3) [등록 없이 로그인시키기, Keycloak CIMD](/posts/mcp-gateway-lab-3/)**
- (4) [도구 없는 경비실, 200줄 게이트웨이](/posts/mcp-gateway-lab-4/)
- (5) [이 숫자를 믿어도 되는가](/posts/mcp-gateway-lab-5/)
