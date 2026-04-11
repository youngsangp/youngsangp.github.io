---
author: 박영상
pubDatetime: 2025-10-01T09:00:00Z
title: TRINITY · 오픈소스 기반 Enterprise AI 플랫폼 내재화
slug: trinity-platform
featured: true
draft: false
tags:
  - LLM
  - RAG
  - Agent
  - Langflow
  - FastAPI
  - Elasticsearch
  - Vector Search
  - Docker
  - WebSocket
  - Knowledge Graph
  - Python
description: Langflow와 OpenWebUI 오픈소스를 엔터프라이즈 환경에 맞게 내재화한 AI 플랫폼. 커스텀 Retriever 컴포넌트, 세션·인증 통합, docker-compose 멀티서비스 운영을 담당했습니다.
---

오픈소스 LLM 워크플로우 엔진(Langflow)과 대화형 UI(OpenWebUI)를 기반으로, 사내·고객사에 납품 가능한 **엔터프라이즈 AI 플랫폼**을 구축하는 프로젝트입니다. 기존 AUD(BI) 플랫폼과의 세션·인증 통합, 고객사 방화벽 환경 대응, 도메인별 RAG/Agent 워크플로우 확장을 목표로 진행 중입니다.

> 사내·고객사 배포 제품 특성상 화면 캡처 없이 아키텍처와 설계 중심으로 서술합니다.

---

## At a Glance

- **역할** · AI 융합팀 **팀장** · 플랫폼 통합 설계 + 핵심 컴포넌트 개발 (팀 프로젝트)
- **기간** · 2025년 10월 – 현재
- **상태** · 상용 제품 (사내 + 고객사 온프레미스 운영 중)
- **스택** · Python, Langflow (FastAPI 기반), OpenWebUI, Elasticsearch, BGE-M3, Jena Fuseki, Docker Compose, TRINITY-MAF (Java 리버스 프록시)

---

## 왜 만들었나

사내에 이미 G-MATRIX(Text-to-SQL)와 개별 LLM 연동 기능들이 있었지만, 도메인이 바뀔 때마다 파이프라인을 코드로 다시 짜야 했습니다. 고객사마다 LLM 모델, 데이터 소스, 보안 정책이 달라서 하드코딩된 파이프라인으로는 대응이 어려웠습니다.

Langflow는 워크플로우를 시각적으로 설계하고, 컴포넌트 단위로 교체할 수 있어 이 문제에 적합했습니다. 하지만 오픈소스 그대로는 **기존 플랫폼 인증 연동, 고객사 방화벽 뒤 WebSocket 통신, 사내 데이터 소스(i-META, AUD) 접근**이 불가능했습니다. OpenWebUI도 마찬가지로 세션 관리와 UI 커스터마이징이 필요했습니다.

그래서 오픈소스를 포크하지 않고 **확장 포인트를 활용해 엔터프라이즈 기능을 얹는 방식**으로 내재화했습니다.

---

## 내가 담당한 영역

### 1. Jena Ontology Retriever (546줄)

Langflow RAG 워크플로우에서 **Knowledge Graph를 검색 소스로 사용**할 수 있게 만든 커스텀 Retriever 컴포넌트입니다. 두 가지 모드를 지원합니다:

- **RAG 모드**: 질문에서 키워드를 추출 → Elasticsearch 시맨틱 검색으로 엔티티 매칭 → 각 엔티티에서 양방향 N-hop SPARQL 탐색 → 수집된 트리플을 LLM 컨텍스트로 주입
- **SPO 모드**: LLM이 질문을 Subject/Predicate/Object 조건으로 분해 → 직접 SPARQL 쿼리 생성·실행

벡터 검색(비정형 문서)과 그래프 검색(정형 관계)을 **하나의 Retriever 인터페이스**로 통합해, 워크플로우 설계자가 데이터 소스를 의식하지 않고 조합할 수 있게 만든 것이 핵심 설계 포인트입니다. 시맨틱 임계값, Top-K, 최대 트리플 수, Hop 깊이 등을 Langflow UI에서 파라미터로 조절 가능합니다.

### 2. DataStore Retriever (291줄)

Elasticsearch 벡터 저장소를 Langflow 워크플로우에서 **범용 검색 컴포넌트**로 사용할 수 있게 만든 Retriever입니다.

- Similarity / MMR(Maximal Marginal Relevance) 검색 모드
- 키워드 기반 쿼리 분할 (구분자 기반)
- Score threshold 필터링으로 노이즈 제거
- 메타데이터 추출 및 구조화

i-META 스키마 필드 검색, 문서 청크 검색 등 다양한 데이터 소스에 재사용됩니다.

### 3. 플랫폼 통합 설계 — 세션·인증·CORS

기존 AUD 플랫폼은 Java 기반(Tomcat), Langflow는 Python(FastAPI), OpenWebUI는 별도 서비스입니다. 세 서비스가 **하나의 도메인 아래서 동일한 세션으로 동작**해야 했습니다.

이를 위해 TRINITY-MAF(Java 기반 리버스 프록시)를 직접 설계·구현했습니다:
- HTTP/WebSocket 프로토콜 자동 감지 및 라우팅
- 단일 도메인으로 CORS 해소
- AUD 세션 기반 인증 일원화
- 고객사 웹서버(Apache/Nginx) 뒤에서 WebSocket upgrade 헤더 유실 대응

### 4. docker-compose 멀티서비스 운영

TRINITY 플랫폼 전체를 docker-compose로 구성해 **한 번의 배포로 전체 스택이 올라가도록** 설계했습니다:

- Langflow Backend (FastAPI)
- OpenWebUI Frontend
- Elasticsearch (BGE-M3 임베딩 인덱스)
- Jena Fuseki (Knowledge Graph 저장소)
- Redis (세션/캐시)

서비스 간 의존성 순서, 헬스체크, 볼륨 마운트, 환경변수 관리를 docker-compose.yml 하나로 통합 관리합니다. 고객사별로 LLM 엔드포인트, 모델명, 포트 등만 `.env`로 분리해 환경을 빠르게 구성할 수 있습니다.

---

## 기술 과제

### 오픈소스 내재화의 경계 설정

가장 어려웠던 건 **"어디까지 수정하고 어디부터 확장으로 붙일지"**의 경계를 정하는 것이었습니다.

Langflow 코어를 포크해서 직접 수정하면 단기적으로 빠르지만, 업스트림 업데이트를 따라갈 수 없게 됩니다. 그래서 **Extension API 디렉토리(`api/v1/extension/`)에 커스텀 엔드포인트를 추가하고, 커스텀 컴포넌트는 별도 디렉토리(`flows/components/`)에 분리**하는 원칙을 세웠습니다. 코어 코드 수정은 세션 연동과 인증 미들웨어 삽입 등 최소한으로 제한했습니다.

이 원칙 덕분에 Langflow 버전 업그레이드 시에도 커스텀 기능이 깨지지 않고, 고객사별로 필요한 컴포넌트만 선택적으로 배포할 수 있었습니다.

### 고객사 환경에서의 WebSocket 안정성

온프레미스 고객사 환경에서 가장 까다로웠던 건 WebSocket 연동이었습니다. 고객사 웹서버(Apache/Nginx)가 중간에 들어오면 `Connection: Upgrade` 헤더가 유실되거나, 프록시 타임아웃으로 연결이 끊기는 문제가 반복적으로 발생했습니다. 웹서버 설정(ProxyPass, proxy_read_timeout 등)을 고객사 인프라팀과 맞춰가며 해결한 경험이 가장 실무적으로 어려운 부분이었습니다.

---

## 회고

**오픈소스를 "가져다 쓰는 것"과 "제품에 내재화하는 것"은 완전히 다른 일**이라는 것을 체감했습니다. 코드를 읽는 시간이 쓰는 시간보다 많았고, 확장 포인트를 찾는 것 자체가 설계 역량이었습니다.

팀 프로젝트로서 가장 신경 쓴 부분은 **"팀원이 컴포넌트를 만들 때 플랫폼 구조를 몰라도 되게 하는 것"**이었습니다. Retriever/Store/Tool 인터페이스를 표준화해서, 새 데이터 소스를 붙일 때 기존 패턴만 따르면 워크플로우에 바로 연결되도록 했습니다.

다음 단계로는 Kubernetes 환경에서의 멀티노드 배포와 Langfuse 기반 LLM 호출 모니터링을 도입해, 운영 가시성을 높이고 싶습니다.

---

**Tech Stack** · Python · Langflow (FastAPI 기반) · OpenWebUI · Elasticsearch · BGE-M3 · Jena Fuseki · Docker Compose · Redis · Java (TRINITY-MAF)

**핵심 키워드** · RAG · Agent · Workflow · LLM Pipeline · Vector Search · Knowledge Graph · Retriever · 오픈소스 내재화 · Docker · 인증 통합 · WebSocket
