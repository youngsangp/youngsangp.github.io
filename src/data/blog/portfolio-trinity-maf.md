---
author: 박영상
pubDatetime: 2026-01-31T09:00:00Z
title: TRINITY-MAF · Multi-protocol Application Facade
slug: trinity-maf
featured: true
draft: false
tags:
  - Java
  - Reverse Proxy
  - WebSocket
  - SSE
  - LLM Streaming
  - Servlet
  - Gateway
  - Backend
  - Platform Architecture
description: 상용 판매 제품(AUD Platform) 내 Reverse Proxy Backend. LLM Agent · 플로우 디자이너 · 데이터 시각화(Superset) 세 개의 이기종 백엔드를 단일 게이트웨이로 통합. LLM 응답의 실시간 SSE 스트리밍과 WebSocket 양방향 브리지를 지원하며, WAS(Tomcat/JEUS) 위에 라이브러리로 배포되어 다수 고객사에 온프레미스 납품·운영 중입니다.
---

![TRINITY-MAF Architecture](/assets/trinity-maf/architecture.svg)

TRINITY 제품군은 서로 성격이 다른 세 개의 백엔드 — LLM Agent 런타임, 플로우/온톨로지 디자이너, 데이터 시각화(Superset) — 로 구성됩니다. 이 셋을 사용자가 의식하지 않고 하나의 도메인에서 쓸 수 있게 만드는 것이 TRINITY-MAF(Multi-protocol Application Facade)의 역할입니다.

단순한 URL 매핑 수준의 프록시가 아니라, **HTTP · WebSocket · SSE 스트리밍 · 인증 세션**을 동시에 처리하면서 고객사 방화벽과 웹서버 뒤에서도 동작해야 하는 게이트웨이입니다. BIMATRIX의 상용 판매 제품인 **AUD Platform**의 TRINITY 모듈에 포함되어 다수 고객사에 온프레미스로 납품·운영되고 있습니다.

> 상용 판매 제품 특성상 코드 공개는 제한하며, 아키텍처와 설계 의도 중심으로 정리합니다.

---

## At a Glance

- **역할** · 설계 · 구현 · 고객사 환경 대응 (1인)
- **기간** · 2026년 1월 – 현재 (초기 구현 1개월 + 유지보수·기능 개선 지속)
- **상태** · 상용 판매 제품 (AUD Platform) — 다수 고객사 온프레미스 납품·운영 중
- **스택** · Java 17, Jakarta EE / Tomcat 9+, Apache HttpClient 5, Jakarta WebSocket, Gradle (Kotlin DSL)

---

## 왜 만들었나

TRINITY 제품군이 여러 백엔드로 나뉘면서 클라이언트 입장에서 세 가지 문제가 생겼습니다.

**첫째, CORS와 도메인 파편화.** 각 백엔드가 다른 포트·호스트에 뜨니 브라우저는 매번 CORS를 마주쳤고, 사용자 입장에선 "같은 제품인데 주소가 여러 개"라는 혼란이 있었습니다.

**둘째, 고객사 방화벽.** 온프레미스 배포 시 고객사 보안팀에 여러 포트를 한꺼번에 열어달라고 요청하는 것은 현실적으로 쉽지 않습니다. 한 엔드포인트만 열고 내부에서 라우팅하는 쪽이 훨씬 깔끔합니다.

**셋째, 인증 일원화.** 특히 Superset은 별도 계정 체계가 있어서 사용자가 로그인 정보를 따로 외울 필요가 없어야 했습니다. 제품 입장에선 Superset 계정 자체가 사용자에게 노출될 이유가 없고요.

결국 답은 **"모든 클라이언트 트래픽이 통과하는 얇은 Facade 한 장"** 이었습니다. 그것이 TRINITY-MAF입니다.

---

## 무엇을 만들었나

TRINITY-MAF는 Tomcat 위에 얹힌 Servlet Filter입니다. 클라이언트 요청이 들어오면 URI 프리픽스를 보고 세 백엔드 중 하나로 투명하게 포워딩합니다.

- `/trinity/*` → TRINITY Designer (Langflow · Ontology Designer)
- `/agent/*` → TRINITY Portal / Agent Runtime
- `/visual/*` → Apache Superset

단순해 보이지만 실제로는 여러 레이어가 얽혀 있습니다. HTTP 요청은 Apache HttpClient 5 기반으로 포워딩하면서 Location 헤더를 Facade 경로로 다시 써주고, hop-by-hop 헤더(Connection, TE 등)를 제거하고, X-Forwarded-* 헤더를 세팅합니다. WebSocket은 Jakarta WebSocket API 위에서 **양방향 브리지**를 직접 구현했고, Superset은 **공통 계정 자동 로그인 + CSRF 토큰 캐시**를 관리합니다.

각 백엔드의 특성을 Facade가 흡수해 주는 덕에, 클라이언트는 "TRINITY"라는 하나의 컨텍스트만 보면 됩니다.

---

## 설계에서 신경 쓴 것들

### 1. URI 라우팅은 "경계"까지 본다

가장 단순해 보이지만 의외로 조심해야 하는 부분이었습니다. `/agent` 라는 프리픽스가 있을 때 `/agents-list` 같은 경로가 잘못 매칭되면 안 됩니다. 그래서 라우팅 로직은 프리픽스 매칭 후 **다음 문자가 `/`이거나 URI 끝**인 경우에만 매칭으로 인정합니다. 복수 라우트가 경합할 땐 longest-prefix-first로 해소합니다.

### 2. HTTP 프록시는 "투명"해야 한다

HttpClient 5 기반으로 직접 리다이렉트를 꺼두고, 대신 백엔드가 보내는 `Location` 헤더를 읽어서 Facade 경로로 다시 씁니다. 예컨대 Superset이 `http://192.168.0.90:9100/login/` 으로 리다이렉트하라고 응답하면, Facade는 그것을 `/matrix/visual/login/` 으로 바꿔서 브라우저에 전달합니다. 이렇게 하지 않으면 사용자가 갑자기 내부 IP로 튕겨 나가는 상황이 벌어집니다.

타임아웃은 설정으로 뺐습니다. `TRINITY_PROXY_CONNECT_TIMEOUT`, `TRINITY_PROXY_RESPONSE_TIMEOUT`, `TRINITY_WEBSOCKET_TIMEOUT` 등 — 고객사마다 네트워크 환경이 다르기 때문에 코드 재배포 없이 `설정 매니저` 으로 조정할 수 있도록 했습니다.

### 3. Superset 세션은 Facade가 알아서 관리한다

사용자는 TRINITY에 로그인하면 됩니다. Superset 계정은 몰라도 되고, 알 필요도 없습니다.

이를 위해 `SupersetSessionManager`를 싱글톤으로 두고 30분 TTL 캐시에 **세션 쿠키 + CSRF 토큰**을 저장합니다. 처음 `/visual/*` 요청이 들어오면 Facade가 Superset에 로그인 페이지를 먼저 긁어서 CSRF 토큰을 뽑고, 폼 로그인으로 세션을 만든 뒤 그 쿠키와 토큰을 모든 후속 요청에 주입합니다. 401이 떨어지거나 TTL이 만료되면 자동으로 재로그인합니다.

의존성을 최소화하려고 JSON/HTML 파싱도 Jackson 같은 라이브러리 없이 `indexOf`/부분 문자열 매칭으로 처리했습니다. Facade는 가볍게 유지하고 싶었습니다.

---

## LLM 실시간 스트리밍 — SSE 프록시

LLM 응답을 사용자에게 실시간으로 전달하기 위해 **SSE(Server-Sent Events) 스트리밍 프록시**를 구현했습니다. 전체 스트리밍 경로는 다음과 같습니다.

```
클라이언트 (브라우저)
  ↓ POST /agent/api/chat/completions (stream: true)
TRINITY-MAF (SSE 프록시)
  ↓ prefix 제거 → /api/chat/completions
OpenWebUI (Agent Runtime)
  ↓ 모델 라우팅 → Langflow
Langflow (Flow 실행)
  ↓ LLM 호출
LLM (온프레미스 GPU 서버)
  ↑ 토큰 단위 생성
  ← data: {"choices":[{"delta":{"content":"안"}}]}
  ← data: {"choices":[{"delta":{"content":"녕"}}]}
  ← data: [DONE]
```

백엔드가 `text/event-stream`으로 응답하면, Facade는 `Content-Type`을 감지해 매 chunk마다 즉시 flush하는 스트리밍 모드로 전환합니다. 이 구간의 모든 컴포넌트가 SSE를 지원해야 사용자가 LLM 토큰을 실시간으로 볼 수 있으며, MAF는 그 첫 번째 관문입니다.

SSE는 일반 HTTP 위에서 동작하기 때문에 **앞단 웹서버(Apache/Nginx) 설정 변경 없이** 바로 사용할 수 있습니다. WebSocket과 달리 `Upgrade` 헤더 이슈가 없어, 고객사 인프라 환경에 영향을 받지 않습니다. WebSocket은 양방향 실시간 통신(Socket.IO 기반 채팅)에, SSE는 LLM 토큰 스트리밍에 각각 사용하며, 두 프로토콜 모두 MAF가 프록시합니다.

---

## WebSocket 브리지

Agent 백엔드는 Socket.IO 기반 스트리밍을 씁니다. LLM 응답 토큰이 실시간으로 내려오는 구조라 이 연결이 끊기면 사용자 경험이 크게 무너집니다. 그런데 WebSocket 프록시는 단순한 HTTP 포워딩보다 훨씬 까다로운 구간이 많았습니다.

### 동시 전송 문제

Jakarta WebSocket의 `getAsyncRemote().sendText()`는 **같은 세션에서 동시에 호출하면 `IllegalStateException`** 을 던집니다. 브리지 구조상 클라이언트와 백엔드 양쪽에서 메시지가 비동기로 들어오니 그냥 전달만 해선 곧바로 터졌습니다.

그래서 방향별로 `ConcurrentLinkedQueue` + `AtomicBoolean`을 한 쌍씩 두고, "전송 중" 플래그가 꺼져 있을 때만 큐에서 하나 꺼내 보내고, 완료 콜백에서 다시 큐를 드레인하는 방식으로 **각 방향의 전송을 직렬화**했습니다. 큐 최대 크기는 1024로 두어 백프레셔도 같이 다룹니다.

### 진짜 어려웠던 것 — 고객사 웹서버 뒤의 헤더 유실

설계의 정합성보다 훨씬 시간이 많이 들었던 건 **고객사 웹서버 환경**이었습니다. 여러 고객사가 TRINITY 앞단에 nginx나 Apache를 두고 운영하는데, 웹서버가 WebSocket 핸드셰이크의 `Upgrade`, `Connection` 헤더를 **묵묵히 지워버리는** 일이 반복적으로 발생했습니다. 심지어 어떤 환경에선 Origin 헤더나 Cookie 일부만 사라지기도 했습니다.

WebSocket 프로토콜 입장에서 `Upgrade: websocket` 이 사라지면 애초에 핸드셰이크가 성립하지 않습니다. 그런데 로그만 보면 "그냥 연결 실패" 로 찍히니, 원인을 찾는 데 시간이 걸렸습니다.

이 문제를 풀기 위해 두 방향으로 접근했습니다.

첫째, **Facade 쪽에서 헤더 복원·재전송을 최대한 방어적으로** 했습니다. `SocketIoServerConfigurator`에서 클라이언트 핸드셰이크의 모든 헤더를 복사해 두고, `SocketIoClientConfigurator`에서 백엔드로 나갈 때 Cookie · Authorization · Origin · User-Agent를 명시적으로 주입합니다. 중간 웹서버가 일부 헤더를 지우더라도 Facade → 백엔드 구간에서는 복원되도록요.

둘째, **고객사 웹서버 설정 가이드를 만들었습니다.** nginx라면 `proxy_http_version 1.1`, `Upgrade`/`Connection` 헤더 명시적 전달, `proxy_read_timeout` 상향 같은 설정 세트를 문서화해서 고객사 인프라팀과 함께 맞추는 형태로요. 프록시 설정이 한 줄만 빠져도 증상이 "가끔 끊긴다"로 나타나기 때문에 가이드를 따라가며 재현·검증하는 과정을 여러 고객사에서 반복했습니다.

코드보다 환경이 더 어려운 문제였고, 지금도 새로운 고객사가 붙을 때마다 가장 먼저 맞춰봐야 하는 부분입니다.

---

## Tomcat · JEUS 호환

TRINITY는 고객사 환경에 따라 Tomcat이 아닌 JEUS 같은 WAS에 올라가기도 합니다. Jakarta WebSocket 엔드포인트는 보통 Servlet Container Initializer를 통해 자동 등록되는데, JEUS에서는 이게 조용히 실패하는 경우가 있었습니다.

해결책은 `WsEndpointCheckListener` — ServletContextListener에서 WAS 종류를 감지해서, Tomcat이면 SCI에 맡기고 그 외 환경이면 **수동 등록**으로 우회합니다. 체크는 Tomcat의 내부 클래스(`org.apache.tomcat.websocket.server.WsFilter`)가 classpath에 있는지로 판단합니다. 지저분하지만 가장 신뢰할 수 있는 방식이었습니다.

---

## 운영에서의 선택

Facade는 얇아야 합니다. 로직이 두꺼워지면 디버깅이 어려워지고, 장애 영향도가 커집니다. 그래서 몇 가지 원칙을 지켰습니다.

- **상태 최소화** — Superset 세션 캐시 외에는 상태를 두지 않습니다. 재시작이 싸야 합니다.
- **설정 외부화** — 백엔드 URL, 타임아웃, 버퍼 크기, WebSocket idle timeout 모두 `설정 매니저` 으로 뺐습니다. 고객사 튜닝 시 코드 재빌드가 없어야 했습니다.
- **Pooling 기반 HTTP 클라이언트** — 매 요청마다 connection을 새로 열지 않도록 `PoolingHttpClientConnectionManager`를 썼습니다. WebSocket idle timeout은 600초, 메시지 버퍼는 10MB를 기본값으로 두되 모두 재정의 가능합니다.
- **로깅의 결을 맞춤** — MATRIX 프레임워크의 `프레임워크 로거`로 통일해서 다른 TRINITY 컴포넌트 로그와 한 줄에 섞여 읽히도록 했습니다. 장애 조사 시 "어디서 끊겼는지"를 Facade 단에서 먼저 봐야 할 때가 많습니다.

---

## 회고

TRINITY-MAF는 코드량 기준으로 크지 않은 프로젝트입니다(메인 클래스 13개, 약 1.3k LOC). 하지만 들어간 시간의 상당 부분은 **"코드가 옳게 동작하는 것"** 보다 **"낯선 환경에서도 동작하게 만드는 것"** 에 쓰였습니다.

사내 개발 환경에서는 WebSocket 브리지가 한 번에 붙는데, 고객사에 가면 nginx 설정 한 줄 때문에 며칠을 헤매기도 합니다. 그 경험이 이 프로젝트의 설계 결정 대부분을 만들었습니다 — 헤더를 방어적으로 복원하고, 타임아웃을 모두 외부 설정으로 빼고, Tomcat이 아닐 수도 있다는 걸 코드로 가정하고, 로그를 장애 조사 관점으로 남기는 것.

"얇지만 현장을 버틸 수 있는 Facade" — 그게 목표였고, 현재는 AUD Platform의 일부로 여러 고객사에 납품되어 조용히 자기 일을 하고 있습니다.
