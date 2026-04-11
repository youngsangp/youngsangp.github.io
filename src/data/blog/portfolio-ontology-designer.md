---
author: 박영상
pubDatetime: 2026-02-01T09:00:00Z
title: Ontology Designer · Knowledge Graph IDE
slug: ontology-designer
featured: true
draft: false
tags:
  - Knowledge Graph
  - LLM
  - RAG
  - Agent
  - Python
  - TypeScript
  - Vector Search
  - Elasticsearch
  - Text-to-SPARQL
  - Ontology
  - Data Pipeline
  - Docker
  - CI/CD
description: RDF/OWL 기반 지식 그래프 IDE. 자연어→SPARQL 변환, Elasticsearch + BGE-M3 하이브리드 RAG, LLM 기반 온톨로지 자동 생성, PDF→Ontology 3-Stage 비동기 데이터 파이프라인을 포함합니다.
---

![Ontology Designer 전체 레이아웃](/assets/ontology-designer/01-overview.png)

도메인 전문가가 직접 지식 그래프를 설계하고, LLM이 그 구조를 정확히 이해할 수 있도록 만드는 웹 기반 온톨로지 IDE입니다. RDF/OWL 표준을 따르되, 사용자가 OWL 용어를 몰라도 되도록 설계했습니다. 자연어·RDB·PDF로부터 온톨로지를 자동 생성하는 파이프라인과 SPARQL 기반 탐색까지 하나의 도구 안에서 처리합니다.

BIMATRIX TRINITY 제품군의 상용 모듈로, 외부 고객사에 온프레미스로 납품·운영되고 있습니다. 기획부터 운영까지 1인이 풀스택으로 담당했습니다.

> 스크린샷과 상세 구현은 사내·고객사 배포 제품 특성상 일부만 공개합니다.

---

## At a Glance

- **역할** · AI 융합팀 **팀장** · 풀스택 설계·구현 (1인) · 프론트엔드 + 백엔드 API + LLM/RAG 파이프라인 + 데이터 파이프라인 + CI/CD + 운영
- **기간** · 2026년 2월 – 현재
- **상태** · 상용 제품 (외부 고객사 온프레미스 납품·운영 중)
- **스택** · Python, TypeScript, Sigma.js, LangChain Core, Langflow (FastAPI 기반), Jena Fuseki (TDB2 + OWL Reasoning), Elasticsearch (BGE-M3), Docker, Jenkins

---

## 왜 만들었나

RAG가 주류가 된 이후에도 한 가지 한계가 분명했습니다. 벡터 검색은 "의미적으로 비슷한 청크"를 잘 찾아주지만, **관계와 구조**는 이해하지 못합니다. "A 부서 소속이면서 B 자격증이 있고 C 프로젝트에 참여한 사람"처럼 조건이 교차하는 질의는 벡터 유사도만으로 풀리지 않습니다.

이 문제의 해답은 결국 지식 그래프(Knowledge Graph)인데, 기존 오픈소스 도구(Protégé 등)는 설치형이고 학습 곡선이 높아 도메인 전문가가 직접 다루기 어려웠습니다. 한국어 지원과 기업 데이터 시스템(RDB·문서·사내 LLM) 통합도 미흡했고요.

그래서 만들었습니다. **비개발자도 웹에서 바로 쓸 수 있고, 기존 데이터로부터 자동 생성되며, LLM이 생성한 온톨로지를 다시 LLM이 활용할 수 있는 도구**를 목표로.

---

## 무엇을 만들었나

### 그래프 기반 시각 편집기

![그래프 시각화 + 좌측 탐색 패널](/assets/ontology-designer/02-graph-visualization.png)

Sigma.js + Graphology + ForceAtlas2 기반의 웹 그래프 에디터입니다. 클래스 뷰와 인스턴스 뷰 전환, k-hop BFS 기반 주변 관계 탐색, 4탭 온톨로지 탐색 패널(클래스·인스턴스·관계·속성)을 지원합니다. OWL 구조를 직관적으로 편집할 수 있도록 설계했습니다.

### 자연어 → SPARQL 검색 파이프라인 (Knowledge Graph Retriever)

Text-to-SQL이 자연어를 SQL로 바꿔 RDB에서 답을 찾듯이, 이 파이프라인은 자연어를 SPARQL로 바꿔 Knowledge Graph에서 답을 찾습니다. 약 3,900줄 규모의 `nl_to_sparql` 모듈을 직접 설계·구현했습니다.

"서울대 출신이면서 영어 시험 점수가 900점 이상인 직원"처럼 조건이 교차하는 질의를 처리하기 위해 7단계 파이프라인으로 구성했습니다:

1. **온톨로지 자동 선택** — 여러 온톨로지가 등록된 환경에서 질문에 가장 적합한 온톨로지를 LLM이 선택
2. **키워드 → IRI 시맨틱 매칭** — Elasticsearch + BGE-M3 임베딩으로 자연어 토큰을 온톨로지 IRI에 매칭, N-hop 구조 확장으로 관련 프로퍼티까지 포함
3. **6종 구조적 템플릿(T1–T6)** — 인스턴스 목록, 서브클래스 탐색, 속성 조회, 카운트, 관계 순회, 서브그래프 추출. 템플릿으로 해결되면 **LLM 호출 없이 즉시 실행**
4. **LLM SPARQL 생성 (Fallback)** — 복합 질의만 LLM에 넘기되, 필터링된 스키마와 의도 분류 결과를 프롬프트에 주입
5. **SPARQL 정적 검증** — GRAPH 절 보정, CURIE↔IRI 수정, 안티패턴 재작성, 안전성 검증(DROP/DELETE 차단)
6. **실행 + 0건 Fallback** — 결과가 없으면 메인 클래스 기반 fallback SPARQL 자동 생성
7. **RAG 모드** — 키워드별 ES 검색 → 엔티티별 양방향 N-hop SPARQL → 트리플 수집 → LLM 컨텍스트 제공

핵심 설계 원칙은 **"템플릿으로 풀 수 있으면 LLM을 쓰지 않는다"**. LLM은 비용과 지연이 크므로 구조적으로 매칭 가능한 질의는 결정론적 코드로 처리하고, 진짜 복합적인 질의에만 LLM을 사용합니다. RAG 모드는 TRINITY 플랫폼의 Langflow 기반 Retriever 컴포넌트로 연결되어 대화형 AI에서 Knowledge Graph 검색을 수행합니다.

### 시맨틱 온톨로지 인덱싱 (Elasticsearch + BGE-M3)

위 NL→SPARQL 파이프라인이 자연어 토큰을 온톨로지 IRI에 정확히 매칭하려면, 온톨로지 전체를 벡터로 인덱싱하는 기반이 필요합니다. `semantic_index` 모듈이 이 역할을 합니다.

온톨로지가 로드되면 SPARQL로 클래스·ObjectProperty·DataProperty·인스턴스를 추출하고, 각 요소의 **다국어 라벨(rdfs:label, skos:prefLabel)과 구조 정보(domain/range, 상위 클래스, 인스턴스 소속)**를 결합해 문서를 생성합니다. 이 문서를 BGE-M3 임베딩 모델로 벡터화한 뒤 Elasticsearch에 저장합니다.

이렇게 하면 사용자가 "영어 점수"라고 입력해도, 온톨로지에 `englishScore`로 정의된 DataProperty를 시맨틱 유사도로 찾아낼 수 있습니다. 단순 키워드 매칭으로는 불가능한 한국어↔영어 크로스링구얼 검색, 약어·동의어 매칭이 가능해집니다. NL→SPARQL의 2단계(IRI 시맨틱 매칭)와 7단계(RAG 모드)가 모두 이 인덱스에 의존합니다.

### AI Helper — LLM 기반 온톨로지 지원

![AI Helper](/assets/ontology-designer/07-ai-helper.png)

온프레미스 LLM(120B급)과 연동되며, 자연어 프롬프트 기반으로 네 가지 모드를 제공합니다.

- **생성 모드**: 자연어 설명을 입력하면 OWL TTL을 자동 생성. 시스템 프롬프트에 현재 온톨로지 스키마를 주입해 기존 구조와 일관된 결과를 유도
- **수정 모드**: "역량 클래스에 자격증 관련 속성 추가해줘" 같은 자연어 지시로 구조 변경. 현재 TTL을 컨텍스트로 넘기고, diff를 계산해 변경분만 적용
- **설명 모드**: 온톨로지 구조를 비개발자도 이해할 수 있는 자연어로 풀어서 설명
- **검증 모드**: 네이밍 일관성, 누락된 라벨·도메인·레인지, 고아 클래스 같은 품질 이슈를 자동 감지하고 수정안 제시

### PDF → Ontology 3-Stage 비동기 데이터 파이프라인

![PDF → Ontology 파이프라인](/assets/ontology-designer/09-pdf-ontology.png)

긴 매뉴얼·규정 문서를 온톨로지로 변환하는 3단계 비동기 파이프라인입니다. Job ID 기반 상태 폴링과 `asyncio.Task.cancel()` 기반 취소를 지원하며, 10분 TTL 자동 정리로 서버 리소스를 관리합니다.

**Stage 0: 시맨틱 블록 추출.** IBM Docling을 사용해 PDF의 문서 구조(표·목차·그림 캡션 등)를 감지합니다. 마크다운 중간 표현을 거쳐 19종 시맨틱 블록(heading, paragraph, table, list 등)으로 분류합니다. PDF 내 임베디드 이미지는 VLM(Vision Language Model)에 전달해 텍스트 설명으로 변환하고, 해당 페이지 블록에 삽입합니다.

**Stage 1: 팩트 추출 (병렬).** 각 청크를 LLM에 보내 `{classes, properties, relationships, individuals}` 형태의 팩트 JSON을 뽑습니다. `asyncio.gather`로 병렬 호출하며, 토큰 예산을 사전 계산해 시스템 프롬프트 + 출력 버퍼를 뺀 나머지에 맞춰 청크를 동적 분할합니다.

**Stage 2: TTL 생성.** 수집된 팩트를 병합한 뒤 참조 무결성 검증(orphan 클래스 제거, 중복 프로퍼티 타입 정리)을 거치고, LLM으로 최종 TTL을 생성합니다. 결과가 크면 파트별로 분할 생성한 뒤 병합하며, rdflib로 파싱 검증합니다. ObjectProperty/DatatypeProperty 이중 선언 같은 OWL DL 위반도 자동 보정합니다.

---

## 시스템 아키텍처

![시스템 아키텍처 다이어그램](/assets/ontology-designer/architecture.svg)
*프론트엔드 · 백엔드 · 외부 시스템 3계층으로 구성된 아키텍처*

백엔드는 세 개의 외부 시스템과 통신합니다 — 온프레미스 LLM(120B급), Jena Fuseki TDB2(지식 그래프 영속 저장 + OWL Micro Reasoning), Elasticsearch(BGE-M3 임베딩 기반 시맨틱 인덱스). 프론트엔드(TypeScript, Sigma.js)는 TRINITY 플랫폼의 Langflow 기반 백엔드 API를 통해 이 시스템들과 통신합니다.

---

## 핵심 기술 문제

### 1. NL→SPARQL 파이프라인의 핵심 설계 결정

위 파이프라인에서 가장 어려웠던 결정은 **"LLM에 언제 의존하고, 언제 의존하지 않을지"**의 경계였습니다.

초기에는 모든 질의를 LLM에 넘겼지만, 응답 시간(3–8초)과 불안정한 SPARQL 문법이 문제였습니다. 분석해 보니 실제 질의의 60–70%는 "A 클래스의 인스턴스 목록", "B의 속성값", "C와 관련된 엔티티"처럼 구조적 패턴으로 분류할 수 있었습니다. 이 발견을 바탕으로 **6종 구조적 템플릿(T1–T6)**을 먼저 시도하고, 매칭되면 LLM을 호출하지 않는 방식으로 전환했습니다. 결과적으로 대부분의 질의가 수십ms 내에 정확한 SPARQL로 변환되고, LLM은 진짜 복합적인 질의에만 사용됩니다.

또한 키워드 → IRI 매칭에서 **규칙 기반 → 시맨틱 검색(BGE-M3) → LLM** 3단계 폴백 구조를 둬, 한 단계가 실패해도 다음 단계에서 복구되도록 설계했습니다. 이 "결정론적 코드 우선, LLM은 fallback" 원칙이 정확도·속도·비용 세 가지를 동시에 잡는 핵심이었습니다.

### 2. PDF→Ontology 비동기 파이프라인의 기술 과제

- **Docling 기반 구조 파싱**: IBM Docling의 마크다운 변환 출력을 ParsedBlock/ParsedSection 포맷으로 정규화해, 이후 팩트 추출·TTL 생성 파이프라인이 파서 출력에 무관하게 동작하도록 설계. 시맨틱 블록 매핑 규칙이 핵심
- **VLM 이미지 파이핑**: PDF 임베디드 이미지를 추출 → base64 인코딩 → OpenAI-compatible multimodal API로 전송 → 텍스트 설명을 해당 페이지 블록에 삽입. 이미지가 많은 문서에서 전체 타임아웃과 개별 실패 처리가 핵심
- **토큰 예산 관리**: 청크별로 시스템 프롬프트 + 출력 버퍼를 뺀 가용 토큰을 사전 계산하고, 초과 시 청크를 동적 분할. LLM 호출 비용과 정확도의 균형

### 3. RAG Retriever 컴포넌트 설계

NL→SPARQL 파이프라인을 단독 API로만 쓰지 않고, TRINITY 플랫폼의 **Langflow 기반 RAG 워크플로우에서 Retriever 컴포넌트로 재사용**할 수 있도록 설계했습니다. 대화형 AI(챗봇)가 Knowledge Graph를 검색 소스로 사용할 때, 질문 키워드별로 Elasticsearch 시맨틱 검색 → 엔티티별 양방향 N-hop SPARQL 탐색 → 트리플 수집 → LLM 컨텍스트 주입 순서로 동작합니다. 벡터 검색(비정형 문서)과 그래프 검색(정형 관계)을 하나의 Retriever 인터페이스로 통합해, 워크플로우 설계자가 데이터 소스를 의식하지 않고 조합할 수 있게 만든 것이 핵심 설계 포인트입니다.

---

## 회고

무엇보다 **LLM을 "마법 블랙박스"가 아닌 "파이프라인의 한 컴포넌트"로 다루는 감각**이 가장 값진 경험이었습니다. 생성형 모델에 어디까지 맡기고 어디부터 결정론적 코드로 보정할지, 모델의 실패를 어떻게 감지하고 복구할지, 토큰 예산을 어떻게 관리할지 — 이런 실무적인 질문들을 몸으로 배웠습니다.

프론트엔드 상태와 백엔드 API 계약을 한 머리에 담고 있어 인터페이스 불일치가 거의 없었다는 점이 프로젝트 진행상 가장 큰 장점이었습니다. 대신 코드 리뷰가 없는 만큼 기술 부채를 쌓지 않으려 의식적으로 노력했습니다.

다음 단계로는 Kubernetes 환경에서의 컨테이너 오케스트레이션 경험을 쌓고, Langfuse 같은 LLMOps 도구를 도입해 NL→SPARQL 파이프라인의 정확도·지연·비용을 정량적으로 추적하는 모니터링 체계를 구축하고 싶습니다. 특히 템플릿 매칭 vs LLM fallback 비율, IRI 매칭 정확도 같은 파이프라인 단계별 메트릭을 체계적으로 수집해 품질을 지속 개선하는 프레임워크가 필요하다고 느끼고 있습니다.

---

**Tech Stack** · Python · TypeScript · Sigma.js · Graphology · LangChain Core · Langflow (FastAPI 기반) · Jena Fuseki (TDB2 + OWL) · Elasticsearch · BGE-M3 · Docling · Docker · Jenkins

**핵심 키워드** · Knowledge Graph · RAG · Hybrid Search · Text-to-SPARQL · LLM Pipeline · Data Pipeline · VectorDB · Embedding · Retriever · LLMOps · 비동기 파이프라인 · Docker
