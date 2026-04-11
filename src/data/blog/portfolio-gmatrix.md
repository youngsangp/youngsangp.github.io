---
author: 박영상
pubDatetime: 2025-09-30T09:00:00Z
title: G-MATRIX · 한국어 자연어 → BI 쿼리 생성 엔진
slug: gmatrix
featured: true
draft: false
tags:
  - LLM
  - RAG
  - FastAPI
  - Text-to-SQL
  - Vector Search
  - Korean NLP
  - Haystack
  - FAISS
  - Milvus
  - Multi-Agent
description: 폐쇄망 온프레미스 환경에서 동작하는 한국어 Text-to-SQL 엔진. 하이브리드 검색 + JSON 중간표현(IR) + 다중 에이전트 구조로 BI 리포트를 자연어로 생성.
---

![G-MATRIX — 자연어 질의 → JSON IR 변환](/assets/gmatrix/01-overview.png)

비개발자가 한국어로 질문하면 BI 리포트를 자동으로 만들어 주는 엔터프라이즈 **Text-to-SQL 엔진**입니다. "올해 3분기 월별 연체개월수가 3보다 큰 상품, 잔액 건수 알려줘" 같은 자연어 질의가 들어오면, 사내 BI 리포트 메타데이터와 결합해 구조화된 JSON 쿼리를 생성하고 SQL로 변환·실행합니다.

단순히 LLM API에 스키마를 던지는 구조가 아니라, **폐쇄망 온프레미스 LLM + FAISS/Milvus 하이브리드 RAG + 계층적 힌트 시스템 + JSON 중간표현(IR) + 다중 에이전트** 파이프라인을 직접 설계·구현했습니다. BIMATRIX의 BI 제품군(AUD)에 통합되어 사내 운영 및 고객사 환경에 배포되었습니다.

> 코드와 상세 구현은 회사 자산 특성상 일부만 공개합니다.

---

## At a Glance

- **역할** · **팀 리드 개발** · 백엔드 + LLM 파이프라인 + RAG 시스템 + 프롬프트 설계 + 멀티 에이전트 아키텍처
- **기간** · 2024 – 2025년 9월 (이후 후속 제품 TRINITY로 발전)
- **상태** · 사내 운영 + 외부 고객사 온프레미스 배포
- **스택** · Python · FastAPI · Haystack · SentenceTransformer · FAISS · Milvus · Pydantic · asyncio

---

## 왜 만들었나

기존 BI 리포트는 사용자가 수십 개의 필터·차원·측정값을 직접 조작해야 했고, 이는 데이터 기반 의사결정의 명확한 병목이었습니다. 비개발자(경영진·현업)는 매번 개발자에게 리포트 요청을 해야 했고요.

그렇다고 단순히 "LLM에 스키마 넘기고 SQL 생성받기" 식으로는 엔터프라이즈 환경에 적용할 수 없었습니다. 네 가지 제약이 있었거든요.

1. **폐쇄망/보안** — 외부 LLM API 호출이 금지된 고객사가 다수. 온프레미스 LLM 서빙 필수
2. **한국어 + 도메인 약어** — `"SD매출"`, `"AUD500"`, `"삼십만원"` 같은 표현은 순수 임베딩으로 매칭 실패
3. **레거시 자산** — 수년간 축적된 BI 리포트 SQL 템플릿을 재활용해야 안정성 확보 가능
4. **정확도** — 잘못된 SQL은 경영 의사결정에 직결, 검증 가능한 구조 필요

이 네 가지를 만족하는 파이프라인을 처음부터 설계해야 했습니다.

---

## 시스템 아키텍처

서버 구성은 **Client → AUD(BI 플랫폼) → G-MATRIX ↔ LLM + Vector Store** 입니다. G-MATRIX는 AUD의 백엔드로 동작하며, 자연어 질의를 받아 JSON 쿼리를 생성하고 BI 엔진에 반환합니다.

파이프라인은 크게 **의도 분류 → Key-Value 분리 → 벡터 검색 → 리포트 선택 → GUI 항목 선정 → LLM JSON 생성 → 교정/SQL 변환** 순서로 진행됩니다. 핵심 설계 원칙은 **"LLM은 자연어 이해만, 나머지는 결정론적 코드가"** 입니다. 의도 분류와 key-value 분해, JSON 생성에만 LLM을 사용하고, 벡터 검색·리포트 선택·SQL 변환은 모두 규칙 기반으로 동작합니다.

---

## 핵심 기술 문제 세 가지

### 1. 왜 하이브리드 검색을 만들었나 — 한국어 도메인 약어와의 싸움

순수 임베딩 검색만으로 시작했지만, 운영해 보니 한국어 약어와 코드명에서 매칭이 자주 실패했습니다. 예를 들어 `"SD매출"`을 검색했을 때, 임베딩은 `"매출"` 부분에만 의미를 부여해 엉뚱한 매출 필드로 매칭되곤 했습니다. 사용자 입장에서는 "AI가 이상한 답을 준다"는 불신으로 이어졌습니다.

해결은 **`GMatrixVectorStore`를 Embedding + InvertedIndex 하이브리드 구조**로 재설계하는 것이었습니다.

- **Embedding 경로**: 사내 파인튜닝 임베딩 모델로 의미 기반 검색. 검색 대상별 차등 임계값 적용 (Meta Field 0.8, Glossary 0.9)
- **InvertedIndex 경로**: 부분문자열 기반 정확 매칭으로 약어·코드 보완. `"SD매출"`은 임베딩에서 놓치지만 InvertedIndex에서 `"SD"` 토큰으로 정확히 잡아냄
- **3개 독립 Vector DB**: Meta Field, Glossary, Synonym을 각각 구축해 용도별 최적화
- 두 경로의 결과를 reportcode 필터와 함께 병합하고 Top-K(20) 랭킹

기존 검색 라이브러리(Whoosh 등) 대신 `InvertedIndex`를 직접 구현한 이유는, 메타필드 단위로 토큰을 분해하고 가중치를 도메인에 맞춰 조정해야 했기 때문입니다. 이 변경 후 잘못된 SQL 생성 빈도가 유의미하게 줄었습니다.

---

### 2. 왜 LLM이 SQL 대신 JSON 중간표현(IR)을 생성하게 했나

처음에는 자연스럽게 "LLM에게 SQL을 직접 만들게 하면 되지 않나?" 생각했습니다. 하지만 세 가지 문제가 있었습니다.

- **검증 어려움** — LLM이 생성한 SQL이 정확한지 사후 검증할 방법이 명확하지 않음
- **멀티뷰 merge 불가** — 여러 리포트 결과를 조합하는 시나리오에서 SQL만으로는 표현 한계
- **레거시 호환 안 됨** — 기존 AUD 리포트 엔진의 SQL 정의 형식과 직접 충돌

그래서 **LLM은 JSON IR만 만들고, SQL 생성은 결정론적 빌더가 담당**하도록 분리했습니다. 실제 시스템에서 생성되는 JSON IR의 형태입니다:

```json
{
  "type": "pivot",
  "dimension": ["T그룹값", "고객상태"],
  "measure": [
    {"name": "수익", "summaryType": "Sum"},
    {"name": "고객수"},
    {"name": "말잔"}
  ],
  "filter": {
    "AND": [
      {"name": "T그룹값", "operator": "=", "value": ["G1"]},
      {"name": "수익", "operator": ">=", "value": ["100"]}
    ]
  },
  "reportcode": "RPT_SALES_001"
}
```

이 구조의 이점은 명확했습니다.

- **검증 가능** — JSON 스키마 단계에서 유효성 검사 가능. `Correction` 클래스가 필드명 재검증, 예약어 체크, 차원 위치 보정까지 수행
- **풍부한 연산자** — `=`, `<>`, `IN`, `NOT IN`, `BETWEEN`, `CONTAIN`, `NOT CONTAIN`, `START`, `END` 등 12종 필터 연산자 지원
- **레거시 호환** — 기존 AUD BI 엔진의 리포트 정의와 1:1 매핑
- **디버깅 용이** — LLM 오류와 SQL 생성 오류를 분리해서 추적 가능

핵심 인사이트는 이거였습니다. **LLM의 가장 큰 약점은 비결정성이고, 가장 큰 강점은 모호한 자연어 이해다.** 이해 영역만 LLM에 맡기고, SQL 생성처럼 정밀함이 필요한 단계는 결정론적 코드로 잡았습니다.

---

### 3. 계층적 힌트 시스템 — LLM에게 맥락을 쌓아주기

LLM에 프롬프트 하나를 던지는 대신, **7계층으로 분리된 힌트를 동적 조립**해서 전달하는 구조를 설계했습니다.

| 계층 | 역할 | 예시 |
|---|---|---|
| **BASE_RULE** | 전역 규칙 (config) | "날짜 형식은 YYYYMM" |
| **GUI_HINT** | 사용자가 선택한 필터값 | "T그룹값 = G1" |
| **META_HINT** | 리포트별 고유 규칙 | "이 리포트는 pivot 타입" |
| **FIELD_HINT** | 필드 설명·제약 | "연체개월수: 숫자형, 0–120" |
| **TERM_HINT** | 동의어·다중항목 정의 | "SD매출 = 서비스디자인매출" |
| **DATE_FORMAT** | 날짜 표현 매핑 | "올해→2025, 3분기→07–09" |
| **NUMBER_HINT** | 한국어 숫자 변환 결과 | "'삼십만' = 300000" |

프롬프트 템플릿에는 `{hint}` 플레이스홀더 하나만 있고, 실행 시점에 이 7계층이 동적으로 채워집니다. 이렇게 분리한 덕분에 **리포트가 추가될 때 코드 수정 없이 데이터만 추가**하면 되었고, 프롬프트 디버깅 시 어느 계층에서 문제가 생겼는지 빠르게 특정할 수 있었습니다.

---

## 한국어 특화 전처리

한국어 BI 도메인에서는 임베딩이 만능이 아니었습니다. `TextConverter` 클래스로 세 가지 전처리를 수행합니다:

**한국어 수사·단위 → 숫자 변환**
```
"삼십만원"  →  300000
"1억 2천만"  →  120000000
"이만삼천"   →  23000
```

**시간 표현 파싱** — 8가지 날짜 입도(year, half, quarter, month, week, day 등) 자동 감지. `DateParser`가 `"올해 3분기"`를 `{year: 2025, quarter: 3}`으로 변환하고, 프롬프트에 구체적 날짜 예시를 삽입합니다.

**키워드 기반 key-value 분해** — LLM이 자연어를 `"DATE:올해 3분기/월/연체개월수:3보다 큰"` 형태로 분해한 뒤, 구조화된 딕셔너리로 파싱합니다.

**임베딩만 믿지 않고 규칙 기반 전처리를 적극 도입한 것**이 한국어 도메인에서의 현실적인 선택이었습니다.

---

## Dual SQL 경로

`METASQLBuilder` 안에서 리포트 타입에 따라 두 가지 경로로 자동 분기했습니다.

| 경로 | 대상 | 방식 | 장점 |
|---|---|---|---|
| **AUD 템플릿 바인딩** | 기존 AUD 리포트 (SD 모듈) | 사전 정의 SQL 테이블 + filter 변수 바인딩 | 검증된 SQL, 안정성 |
| **META 동적 생성** | 신규/범용 질의 | JSON IR → SELECT/FROM/WHERE 동적 구성, CalcField·집계 함수 지원 | 유연성, 확장성 |

이 분기 덕분에 **레거시 자산을 100% 재활용하면서도 신규 시나리오 대응이 가능**했습니다.

---

## 멀티 에이전트 구조

`AgentBase` 추상 클래스를 기반으로 `core/agent/` 아래에 일곱 개의 에이전트를 구현했습니다.

- **`agent_router`** — 입력 의도를 5가지(data_retrieval / data_summary_statistics / data_forecast / guide / general)로 분류, 임계값 0.6 기반 라우팅
- **`agent_selector`** — 후보 메타필드 중 최적 선택
- **`retrieve_agent`** — 파이프라인을 이용한 데이터 검색
- **`search_agent`** — 벡터 유사도 기반 메타 검색
- **`agent_summary`** — 결과 요약·통계·분석
- **`general_agent`** — 일반 대화 / 인사 / 도움말
- **`guide_agent`** — 사용 방법 안내

각 에이전트는 독립적인 system_prompt와 temperature를 가지며, 스트리밍 응답도 지원합니다. 이 멀티 에이전트 구조를 직접 손으로 설계해 둔 경험이, 이후 후속 제품 TRINITY의 LangFlow 기반 워크플로우 설계로 자연스럽게 이어졌습니다.

---

## LLM 호출 추상화

`PromptHelper` 클래스에서 `LLM_TYPE` 설정 기반으로 세 가지 모드를 추상화했습니다.

1. **로컬 transformer 직접 로드** — 4-bit NF4 양자화 지원, 개발용 (한국어 오픈소스 LLM)
2. **Haystack PromptNode** — GPT-3.5/4 등 외부 API
3. **사내 OpenAI 호환 엔드포인트** — 프로덕션 기본값, 온프레미스 vLLM 호환 서버

프롬프트는 태스크별 템플릿으로 중앙 관리합니다:
- **`key-value_prompt`** — 형태소 분석 + key-value 분해
- **`zero-shot_prompt_72B`** — 메인 JSON 생성 (72B 모델 최적화)
- **`chain-zero-shot_prompt_72B`** — 연속 질문 시 이전 JSON 수정
- **`intent_classification_prompt`** — 에이전트 라우팅용 의도 분류
- **`copilot_function_prompt`** — AddSort, AddFilter, AddColumn, Format 함수 호출 생성

`temperature=0`, `seed=0`으로 고정해 **Text-to-SQL의 결정성**을 확보했습니다.

---

## 운영 안정성 설계

- **임베딩 캐시** — `EmbeddingCache`로 pickle 영속화. 재기동 시 수 초 내 로드, 신규 문서만 incremental embedding
- **동시성 제어** — `asyncio.Semaphore(4)`로 워커당 최대 동시 요청 제한
- **멀티테넌트 로깅** — 사용자별 개별 로그 파일 생성
- **플랫폼 연동** — Teams 봇, OpenAI GPTs Actions, Kakao 챗봇

---

## 회고

G-MATRIX는 에이전트 프레임워크가 표준화되기 전에 비슷한 구조를 손으로 설계한 프로젝트였습니다. 그래서 어떤 부분은 더 어려웠지만, 그 과정에서 얻은 감각이 가장 큰 자산이 되었습니다.

특히 **"LLM을 어디까지 믿을 것인가"** 의 경계선을 긋는 것이 설계의 핵심이었습니다. LLM은 비결정적인 "이해" 영역에, 결정론적 빌더는 SQL 생성 영역에 두는 분리가 유지보수성과 정확도를 모두 끌어올렸습니다. 이 원칙은 이후 TRINITY와 Ontology Designer에서도 동일하게 적용한 패턴입니다.

또 하나, **순수 벡터 검색의 한계를 실제 프로덕션에서 체감한 것**이 컸습니다. 실제 한국어 BI 도메인에서는 약어·코드·수사 표현 등 임베딩이 못 잡는 표면적 패턴이 너무 많았습니다. InvertedIndex를 직접 구현하고, 검색 타입별로 차등 임계값을 설정하면서 **하이브리드 검색이 답이라는 것**을 몸으로 배웠습니다.

마지막으로, 엔터프라이즈 환경에서는 **최신 기술보다 레거시 호환성**이 프로젝트의 성패를 가른다는 점도 배웠습니다. AUD 리포트 자산을 버리지 않고 결합한 Dual SQL 경로 설계가 G-MATRIX가 실제로 운영에 들어갈 수 있었던 결정적 이유였다고 생각합니다.


---

**Tech Stack** · Python · FastAPI · Haystack · SentenceTransformer(`사내 파인튜닝 임베딩 모델`) · FAISS · Milvus · Pydantic · asyncio · Pickle Cache · WebSocket
