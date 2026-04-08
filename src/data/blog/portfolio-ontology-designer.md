---
author: 박영상
pubDatetime: 2025-12-01T09:00:00Z
title: Ontology Designer - Knowledge Graph IDE
slug: ontology-designer
featured: true
draft: false
tags:
  - Knowledge Graph
  - LLM
  - FastAPI
  - TypeScript
  - Sigma.js
  - GraphRAG
  - Ontology
description: RDF/OWL 기반 엔터프라이즈 온톨로지 IDE. LLM + Vector Search + OWL Reasoning 통합 파이프라인
---

# 🧬 Ontology Designer

> **지식 그래프(Knowledge Graph)를 시각적으로 설계·편집·배포하는 엔터프라이즈 웹 기반 온톨로지 IDE**
>
> RDF / OWL 표준 기반 지식 그래프를 코드 한 줄 없이 설계하고,
> LLM · Vector Search · OWL Reasoning을 결합한 자동화 파이프라인으로
> RDB · PDF · 자연어로부터 온톨로지를 생성하는 통합 도구

---

## 📌 한눈에 보기

| 항목 | 내용 |
|---|---|
| **프로젝트명** | Ontology Designer (BIMATRIX Trinity 제품군) |
| **역할** | **1인 풀스택 개발** — 기획·설계·프론트엔드·백엔드·LLM 파이프라인·CI/CD·운영 전부 |
| **소속** | 비아이매트릭스 기술연구소 AI 융합팀 |
| **기간** | 2024 ~ 현재 (지속 운영 중) |
| **사용 범위** | **사내 제품 + 외부 고객사 실사용** (B2B · On-Prem 배포) |
| **배포 방식** | Trinity 제품 WAS 통합 배포, Docker 이미지 기반, Jenkins CI/CD 다중 브랜치 자동화 |
| **프론트엔드 규모** | **TypeScript 35,000+ LOC · 60+ 모듈** |
| **백엔드 규모** | **Python FastAPI 7,000+ LOC** (ontology.py 4,145 + graphdb.py 2,362 + fuseki_client.py 830+) |
| **엔드포인트** | **25+ REST API** (PDF 파싱 · TTL 생성 · SPARQL · 스키마 추출 · NL-to-SPARQL 등) |

---

## 🎯 프로젝트 배경

### 해결하려는 문제

> **"도메인 전문가가 직접 지식 그래프를 만들고, LLM이 그걸 정확히 이해하게 한다."**

- **RAG의 한계**: 단순 벡터 검색은 "의미적으로 비슷한 청크"를 찾아줄 뿐, **관계와 구조**를 이해하지 못함 → **Knowledge Graph RAG (GraphRAG)** 가 필요
- **기존 도구의 한계**: Protégé 같은 오픈소스는 설치형·학습 곡선 높음·한국어 지원 미흡·기업 데이터 시스템과 통합 어려움
- **데이터 소스 다양성**: 관계형 DB · PDF 매뉴얼 · 자연어 설명 등 흩어진 데이터를 온톨로지로 전환하는 자동화 파이프라인 부재

### 이 프로젝트가 해결한 것

1. **웹 기반·무설치·다국어(한/영/일)** 온톨로지 편집 환경
2. **LLM 자동 생성 파이프라인**: 자연어 → TTL, PDF → 온톨로지, RDB 쿼리 → 인스턴스
3. **Semantic Search + Knowledge Graph 결합**: Elasticsearch 벡터 인덱스로 NL 질의 → SPARQL 자동 변환
4. **OWL 추론 통합**: Jena Fuseki TDB2 + OWL Micro Reasoner, 서버 사이드 제약 검증 및 자동 수정
5. **Multi-Backend 추상화**: Fuseki / GraphDB / Stardog / Oxigraph / RDFox 인터페이스 설계로 벤더 락인 방지

---

## 🖼️ 주요 화면 & 기능

> **[screenshot: 01-overview.png]** — 전체 레이아웃

### 1. 지식 그래프 시각화 · 편집
> **[screenshot: 02-graph-visualization.png]**

- **Sigma v3 + Graphology** 기반 WebGL 고성능 렌더링
- **ForceAtlas2 + Circular 2-Stage Layout** (아래 기술 도전 섹션 참조)
- **Degree 기반 적응형 노드 크기** (4 ~ 22px 자동 스케일)
- **클래스 뷰 ↔ 인스턴스 뷰** 모드 전환 (동일 그래프의 추상화 수준 변경)

### 2. 멀티 클래스 선택 + K-Hop 포커스
> **[screenshot: 03-multiselect-khop.png]**

- **Ctrl+Click 다중 클래스 선택** → 각 클래스의 인스턴스와 **크로스 관계(ObjectProperty)** 자동 표시
- **노드 클릭 시 k-hop depth BFS 탐색 → 포커스 강조**, 나머지 dim 처리
- 포커스 유지한 채 다중 선택 추가 가능 + 카메라 상태 복원

### 3. Filter · Display · Legend 패널
> **[screenshot: 04-filter-display-legend.png]**

- **Filter**: `subClassOf` / `equivalentClass` / `disjointWith` / `Property (domain/range)` 관계 종류별 토글
- **Display**: 노드 라벨 · 엣지 라벨 · 엣지 표시 개별 제어 (성능 튜닝 옵션)
- **Legend**: 노드 타입별 색상 범례

### 4. 클래스 트리 패널
> **[screenshot: 05-class-tree.png]**

- **계층 구조** 표시 (subClassOf 기반), 인스턴스 개수 뱃지
- **Ctrl+Click 다중 선택**, 배경 클릭 시 ROOT 리셋
- Floating / Docked 모드 · 리사이즈 가능

### 5. 인스펙터 패널 (속성 편집)
> **[screenshot: 06-inspector.png]**

- 클래스·인스턴스·관계(엣지) 3가지 뷰 모드
- 자동 저장 (onBlur), Datatype/Object Property 별도 에디터
- 다국어 라벨 (`@ko`, `@en`, `@jp`)

### 6. AI Helper (LLM 4가지 모드)
> **[screenshot: 07-ai-helper.png]**

**GPT-OSS 120B** 기반 자체 LLM 엔드포인트 연동 (온프레미스)

| 모드 | 설명 | 백엔드 엔드포인트 |
|---|---|---|
| **생성** | 자연어 → OWL TTL 자동 생성 | `POST /ontology/generate-ttl` |
| **설명** | 클래스/속성에 대한 도메인 관점 설명 생성 | `POST /ontology/generate-ttl` (explain mode) |
| **품질 검증** | 네이밍·레이블 누락·계층 이슈 자동 감지 | Fuseki validation API |
| **다국어 번역** | ko/en/jp label 일괄 번역 | LLM translation pipeline |

### 7. RDB → Ontology 자동 매핑
> **[screenshot: 08-rdb-to-ontology.png]**

- **쿼리 직접 입력** 또는 **테이블 선택** 모드
- **DISTINCT 값 기반 클래스 계층 자동 추론**: 컬럼의 유니크 값을 클래스로 변환, 계층 중첩 지원
- **컬럼별 역할 매핑**: `Class` / `DatatypeProperty` / `ObjectProperty` / `Label` / `Instance` 선택

### 8. PDF → Ontology 파이프라인 (3-Stage Async)
> **[screenshot: 09-pdf-ontology.png]**

**비동기 Job 기반, 진행 상태 폴링 + 취소 지원**

```
Stage 0: parse-pdf
    pdfplumber → 19종 시맨틱 블록 감지 (heading/paragraph/table/list/menu_path/form_field/...)
    + Vision LLM (이미지 캡션) → semantic JSON
    ↓
Stage 1: extract-ontology-data
    청크 단위 병렬 LLM 호출 (asyncio.gather)
    → [classes, properties, relationships, individuals] 팩트 JSON
    ↓
Stage 2: json-to-ttl
    팩트 JSON 병합 → LLM TTL 생성 (2-shot prompting)
    → rdflib 검증 → 최종 TTL
```

- **Token Budget 동적 관리**: 8K 컨텍스트 초과 시 자동 재청킹
- **Reasoning Fallback**: content 비었을 때 `<think>` 태그 · `additional_kwargs.reasoning_content`에서 추출
- **Job TTL 10분 자동 정리**, `asyncio.Task.cancel()` 기반 취소

### 9. NL-to-SPARQL (2가지 엔진)
> **[screenshot: 10-nl-to-sparql.png]**

**자연어 질의를 SPARQL로 변환하는 이중 파이프라인**

**A. SPO 모드** (분해형)
```
NL 질문 → LLM이 S/P/O 조건으로 분해 (JSON)
    ↓
schema 추출 (classes, properties, domain/range)
    ↓
각 조건별 IRI 매칭:
    1. localName 직접 매칭
    2. Elasticsearch 시맨틱 검색 (BGE-M3 임베딩)
    3. domain/range 제약 검증
    ↓
SPARQL 생성 → 실행
    ↓
Zero 결과 시 Question Decomposition (최대 5개 서브 질문 분해, 의존 관계 유지)
```

**B. RAG 모드** (템플릿형)
```
키워드 추출 → ES 시맨틱 검색 (threshold 0.5)
    → SPARQL 템플릿 인스턴스화
```

### 10. 온톨로지 병합 마법사
> **[screenshot: 11-merge-wizard.png]**

- **5-Step Wizard**: 소스 선택 → 차이 분석 → 충돌 감지 → 해결 정책 → 실행
- **Rename / Prefer Base / Prefer New / Skip** 4가지 충돌 해결 전략
- 미리보기 + Undo 지원

### 11. 일괄 텍스트 입력 (Bulk Create)
> **[screenshot: 12-bulk-create.png]**

- 들여쓰기 기반 텍스트로 클래스·인스턴스·관계 일괄 생성 (Protégé 스타일)

### 12. TTL Viewer · SPARQL 직접 실행
> **[screenshot: 13-ttl-sparql.png]**

- Prism.js 구문 강조 기반 실시간 TTL 뷰
- SPARQL 직접 실행 (Fuseki 연동) · 결과 테이블

### 13. OWL 제약 검증 + Auto-Fix
> **[screenshot: 14-validation.png]**

**Fuseki 전용 임시 데이터셋(`_validate`, `_validate_inf`) 기반 서버 사이드 검증**

| 카테고리 | 검사 항목 |
|---|---|
| **OWL 제약 (Error)** | `owl:disjointWith` 위반 · `owl:FunctionalProperty` 다중값 · `owl:InverseFunctionalProperty` 다중주어 |
| **품질 경고 (Warning)** | `rdfs:label` 누락 · `rdfs:domain` 누락 · `rdfs:range` 누락 · 빈 Leaf 클래스 |

- 각 위반에 대해 **자동 수정(Auto-Fix) 후보** JSON으로 반환 (choice / select / batch)
- 프론트엔드에서 사용자가 수정안 선택 → 재검증 루프

---

## 🏗️ 시스템 아키텍처

```
┌──────────────────────────────────────────────────────────────┐
│  Browser (TypeScript SPA · 35K LOC · 60+ modules)            │
│  ┌──────────────┬────────────┬────────────┬────────────────┐│
│  │ Sigma.js v3  │ UI System  │ AI Helper  │ Merge · Bulk    ││
│  │ Graphology   │ (tree/     │ (LLM UI)   │ RDB Mapper      ││
│  │ FA2 Layout   │  panel/    │            │ Inspector       ││
│  │              │  modal)    │            │                 ││
│  └──────────────┴────────────┴────────────┴────────────────┘│
│  ┌──────────────────────────────────────────────────────────┐│
│  │ i18next (ko/en/jp) · MiniSearch · N3.js · PDF.js          ││
│  └──────────────────────────────────────────────────────────┘│
└────────────────────────────┬─────────────────────────────────┘
                             │ REST / JSON
┌────────────────────────────┴─────────────────────────────────┐
│  Trinity LangFlow Backend (FastAPI · Python · 7K+ LOC)       │
│                                                               │
│  ┌─ API Layer ─────────────────────────────────────────────┐ │
│  │ ontology.py (4,145L): parse-pdf, generate-ttl,         │ │
│  │                       extract-ontology-data, json-to-ttl│ │
│  │ graphdb.py  (2,362L): SPARQL, NL-to-SPARQL, schema,    │ │
│  │                       graph mgmt, inferred triples     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─ Service Layer ─────────────────────────────────────────┐ │
│  │ TripleStoreClient (ABC) ─┬─ FusekiClient (active)      │ │
│  │                          ├─ GraphDBClient (stub)       │ │
│  │                          ├─ StardogClient (stub)       │ │
│  │                          ├─ OxigraphClient (stub)      │ │
│  │                          └─ RDFoxClient (skeleton)     │ │
│  │                                                          │ │
│  │ NLToSparqlPipeline (2000+L)                             │ │
│  │   ├─ SPO Mode (LLM decomposition)                       │ │
│  │   └─ RAG Mode (keyword + template)                      │ │
│  │                                                          │ │
│  │ SemanticIndex (Elasticsearch + HuggingFace Embeddings)  │ │
│  │   └─ BGE-M3 (default) · per-store indexing              │ │
│  │                                                          │ │
│  │ MXLLModel wrapper                                        │ │
│  │   └─ LangChain Core Messages · Reasoning fallback       │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────┬─────────────────┬────────────────┬────────────────┘
           │                 │                │
  ┌────────▼──────┐  ┌───────▼───────┐ ┌─────▼──────────┐
  │  GPT-OSS      │  │  Jena Fuseki  │ │ Elasticsearch  │
  │  120B         │  │  TDB2 +       │ │ (Vector Index) │
  │  (온프레미스) │  │  OWL Micro    │ │ BGE-M3         │
  │  via          │  │  Reasoner     │ │ Embeddings     │
  │  Bimatrix     │  │               │ │                │
  │  Provider     │  │               │ │                │
  └───────────────┘  └───────────────┘ └────────────────┘
```

### 기술 스택

**Frontend (TypeScript · 35K+ LOC)**
- **시각화**: Sigma.js v3, Graphology, ForceAtlas2, graphology-layout, @sigma/edge-curve, @sigma/node-border
- **데이터**: N3.js (TTL 파서), PDF.js
- **UI**: Vanilla TypeScript + Custom Components, Prism.js, markdown-it
- **i18n**: i18next (한/영/일, 945 × 3 = 2,835 라인)
- **검색**: MiniSearch (클라이언트 풀텍스트)
- **빌드**: Webpack 5, Babel, TypeScript 4.9

**Backend (Python FastAPI · 7K+ LOC)**
- **LLM**: LangChain Core · MXLLModel wrapper · **GPT-OSS 120B** (온프레미스)
- **그래프DB**: Apache Jena Fuseki (TDB2 + OWL Micro Reasoner)
- **벡터DB**: Elasticsearch + HuggingFace Embeddings (**BGE-M3**)
- **RDF 처리**: rdflib · n3 · pyparsing
- **PDF**: pdfplumber + pypdf + Vision LLM
- **비동기**: asyncio · `asyncio.gather()` 병렬 LLM 호출 · ThreadPool executor · Background Job Pattern

**인프라 · 배포**
- **컨테이너**: Docker (Fuseki + Trinity 제품 통합)
- **CI/CD**: Jenkins Pipeline (Active Choices Plugin 기반 다중 브랜치 자동 배포, 직접 설계)
- **저장소**: GitLab (비아이매트릭스 내부)
- **WAS 통합**: Matrix 제품군 Tomcat WAS

---

## 🔥 기술적 도전과 해결

### 🧩 Challenge 1. Jena Fuseki TDB2 + OWL Reasoner 통합

#### 상황
지식 그래프를 **영속 저장**하면서 **OWL 추론**(subClassOf transitive closure 등)도 동작해야 했음. Fuseki 공식 예제는 메모리 모델 기반이라 그대로 쓸 수 없었음.

#### 문제
- `ja:InfModel`은 **Model** 위에만 올릴 수 있고 **Dataset**에 직접 올리면 `ClassCastException`
- TDB2는 **Dataset 단위**로 영속화 → InfModel과 직접 연결 불가
- `ja:InfDataset` 같은 타입은 Jena에 존재하지 않음
- Fuseki는 `fuseki:dataset`으로 **Dataset** 타입을 요구

#### 해결: 4-Layer 아키텍처
```
DatasetTDB2 (영속)
    ↓ tdb2:GraphTDB2
GraphTDB2 (default graph → Model 추출)
    ↓ ja:baseModel
InfModel (OWL Micro Reasoner 결합)
    ↓ ja:defaultGraph
RDFDataset (Fuseki가 요구하는 Dataset 타입으로 재포장)
```

**핵심 인사이트**: 모델↔데이터셋 왕복 패턴. TDB2 Dataset에서 default graph(Model)를 꺼내 InfModel을 씌우고, `ja:RDFDataset`으로 다시 Dataset으로 포장.

#### 결과
- TDB2 영속성 + OWL Micro 추론이 단일 SPARQL 쿼리에서 동시 동작
- 서버 재시작 후에도 추론된 `subClassOf*` / `type` 관계가 자연스럽게 유지됨

---

### 🧠 Challenge 2. Knowledge Graph + Vector Search 융합 (NL-to-SPARQL)

#### 상황
사용자가 자연어로 "서울대 출신 영어 잘하는 직원"을 검색하면 SPARQL로 변환해서 실행해야 함. 단순 키워드 매칭으로는 IRI를 못 찾고, LLM만으로는 스키마를 모르고, ES만으로는 관계를 못 씀.

#### 문제
- **스키마 인지**: LLM이 온톨로지의 클래스/속성 IRI를 정확히 써야 함
- **시맨틱 매칭**: "영어 잘하는" → `hr:speaksLanguage hr:English` + `hr:examScore > 900` 같은 복합 조건 분해
- **Zero Result 대응**: 한 번에 못 찾으면 서브 질문으로 분해해야 함

#### 해결: 하이브리드 2-모드 파이프라인

**SPO 모드 (분해형)**
```python
# 1단계: 스키마 컨텍스트 주입
schema = await client.extract_schema(dataset, graph_uris)
system_prompt = build_schema_prompt(schema)  # classes, properties(domain→range)

# 2단계: LLM이 NL → S/P/O 조건 JSON 분해
conditions = await llm.invoke([
    SystemMessage(system_prompt),
    HumanMessage(f"질문: {question}\n조건들을 JSON으로 분해하라")
])

# 3단계: 각 조건의 IRI 매칭 (3-단계 폴백)
for cond in conditions:
    iri = (
        match_local_name(schema, cond.keyword)         # 1. localName 직접
        or search_semantic_index(cond, top_k=5)         # 2. ES 시맨틱 검색 (BGE-M3)
        or fallback_schema_keyword(schema, cond)        # 3. schema fallback
    )
    validate_domain_range(iri, cond, schema)            # 4. domain/range 검증

# 4단계: SPARQL 빌드 & 실행
sparql = build_sparql(conditions)
result = await client.query(dataset, sparql)

# 5단계: Zero 결과 시 Question Decomposition
if not result.rows:
    sub_questions = await llm_decompose(question, max_splits=5)
    results = []
    for sq in sub_questions:
        if sq.depends_on: substitute_entities(sq, prior_results)
        results.append(await execute_spo(sq))
    result = left_outer_join(results)
```

**핵심 포인트**:
1. **Multi-step Fallback**: localName → 시맨틱 → 스키마 키워드 (각 단계가 실패해도 다음이 구원)
2. **Semantic Index 자동 구축**: 스키마 변경 시 Elasticsearch 인덱스 자동 재생성 (`semantic_ontology__{dataset}__{cat_id}` 네이밍으로 per-store 격리)
3. **Domain/Range 제약 검증**: LLM이 엉뚱한 property를 고르지 못하도록 스키마 수준에서 필터
4. **Question Decomposition**: 한 번에 못 풀면 LLM이 서브 질문 5개로 쪼개고, 이전 결과를 변수로 치환하며 순차 실행

#### 결과
- 자연어 질의에 대한 **정확한 SPARQL 생성 + 실행 성공률 크게 향상**
- 복잡한 다중 조건 질의도 분해 전략으로 대응 가능
- **GraphRAG / Knowledge Graph RAG** 실전 파이프라인 구축 경험

> 💡 이 구조는 배민 물어보새의 **Text-To-SQL + Data Discovery** 파이프라인과 정확히 같은 문제 영역입니다. Vector Search(RAG)만으로는 부족한 구조적 추론을 Knowledge Graph로 보강하는 하이브리드 접근.

---

### 🎨 Challenge 3. 대규모 그래프 렌더링 품질 개선

#### 상황
실사용자가 100+ 노드 · 수백 개 엣지의 온톨로지를 열었을 때 **"스파게티 그래프"**가 나옴. 노드가 뭉치고 엣지가 겹치고 라벨이 서로 가려 읽을 수 없음.

#### 문제 분석
1. Sigma 기본 force-directed는 초기 위치에 민감
2. 노드 크기가 고정(15px)이라 중요도 구분 불가
3. 엣지 라벨이 전부 렌더링되어 overlap
4. FA2 파라미터를 노드 수에 비례해 키우니 오히려 사방으로 튀어나감

#### 해결 전략

**1) 2단계 레이아웃 (Circular → ForceAtlas2)**
- 1단계: `graphology-layout`의 circular 배치로 초기 위치 확정
- 2단계: 200 iterations FA2로 리파인 (Barnes-Hut 최적화 활성)
- FA2가 "무한 공간"이 아닌 "원 안에서 재배치"하게 되어 안정적

**2) Degree 기반 적응형 노드 크기**
```typescript
const baseSize = 4 + Math.log(nodeCount + 1) * 2;
const nodeSize = baseSize + Math.sqrt(degree) * 2;  // 4 ~ 22px
```
중심 허브 노드가 자연스럽게 크게 표시 → 시각적 중요도 가시화

**3) 엣지 라벨 On-Demand 렌더링**
- 기본: 엣지 라벨 숨김 (reducer에서 `label: ''`)
- Hover / Click / k-hop 포커스 범위 내에서만 라벨 복원
- Sigma가 edge `forceLabel`을 지원하지 않는 한계를 **reducer 내에서 라벨 자체를 교체**하는 방식으로 우회

**4) Sigma 렌더링 튜닝**
- `labelRenderedSizeThreshold: 4`, `labelDensity: 1.5`, `labelGridCellSize: 150`
- `defaultEdgeType: "curve"`로 양방향 엣지 구분
- 엣지 두께 축소

#### 결과
- **가독성 대폭 향상** (사용자 피드백: "이제 구조가 한눈에 보인다")
- **상호작용 안정성**: 줌/팬/드래그 중 프레임 드랍 해소
- **중심/말단 구분**: 허브 클래스가 시각적으로 자연스럽게 드러남

---

### ⚙️ Challenge 4. 복잡한 이벤트 시스템의 상태 충돌 해결

#### 상황
hover · click · drag · Ctrl+click · k-hop 포커스 · 필터 · 다중 선택 · 모드 전환이 모두 동시 동작. 한 번의 Sigma `refresh()`에 수십 가지 상태가 엉켜있음.

#### 분석 결과 (19개 이슈 발견: HIGH 5 · MEDIUM 9 · LOW 5)

| # | 문제 | 증상 |
|---|---|---|
| **#1** | `renderEdgeLabels` 상태 충돌 | 엣지 hover 중 노드 클릭 → leave 시점에 포커스 라벨이 꺼짐 |
| **#2** | `selectedNodes` 불일치 | 다중 선택 전 focusNode가 있으면 삭제된 ID가 남아 시각 glitch |
| **#3** | `snapshotPositions` 이중 호출 | renderGraph에서 2번 호출 → 임시 원형 좌표가 위치 캐시에 오염 |
| **#5** | `filterOptions` Race | setMode → getFilterOptions 사이 콜백이 stale 값 참조 |
| **#10** | 드래그 직후 `clickStage` 오탐 | mouseUp 직후 stage 클릭이 잘못 발동 → 선택 해제 |
| **#11** | RAF 타이머 leak | renderGraph 시점의 pending RAF가 killed renderer 참조 |

#### 해결 접근

**상태 플래그 + 가드 + 리소스 정리 패턴**

```typescript
// 상태 플래그
private _isFocusActive = false;
private _lastDragEndTime = 0;
private _lastBuildRanFA2 = false;

// 엣지 leave 가드 — focus 활성 시에는 renderEdgeLabels 끄지 않음
on("leaveEdge", () => {
    if (!this._isFocusActive && !displayOpts.showEdgeLabels) {
        sigma.setSetting("renderEdgeLabels", false);
    }
});

// clickStage 가드 — 드래그 직후 50ms 무시
on("clickStage", () => {
    if (Date.now() - this._lastDragEndTime < 50) return;
    this.setCurrentNode(null);
});

// 리소스 정리
renderGraph() {
    if (this._hoverRefreshRAF) cancelAnimationFrame(this._hoverRefreshRAF);
    this.sigmaRendererRef?.kill();
    // ...
}

// 방어 코드
setFocusByDepth(nodeId: string | null) {
    if (nodeId && !this.graph.hasNode(nodeId)) {
        console.warn(`Node ${nodeId} not found`);
        return;
    }
}
```

#### 결과
- **11개 HIGH/MEDIUM 이슈 수정** — TypeScript 컴파일 + webpack 빌드 모두 clean
- 렌더 루프 중복 호출 감소 (clickNode 이중 refresh 제거 등)
- 재현 어려운 "가끔 라벨 안 나옴" 류 버그 해소

---

### 🤖 Challenge 5. LLM 기반 PDF → Ontology 3-Stage 파이프라인

#### 상황
200페이지 매뉴얼 PDF에서 온톨로지를 자동 생성해야 했음. LLM 컨텍스트는 8K이고, 한 번의 호출로는 불가능. 텍스트 추출도 단순 plain text로는 부족 — **구조(목차, 표, 메뉴 경로)**를 보존해야 의미 있는 온톨로지가 나옴.

#### 문제
1. **구조 손실**: pdfplumber 단순 text 추출 시 표·목차·메뉴 경로 구분 불가
2. **컨텍스트 초과**: 긴 섹션이 LLM 토큰 한계 초과
3. **병렬 처리 필요**: 청크별 순차 호출은 너무 느림
4. **이미지 정보**: 다이어그램/스크린샷은 텍스트로만 추출 안 됨
5. **LLM 불안정성**: content 필드가 비어있고 reasoning에만 답이 있는 경우
6. **Job 생명주기**: 사용자가 취소할 수 있어야 하고, 실패 시 리소스 정리 필요

#### 해결: 3-Stage 비동기 파이프라인

**Stage 0: `parse-pdf` — 시맨틱 블록 추출**
```python
# 19종 시맨틱 블록 타입 감지
BLOCK_TYPES = [
    "title", "heading1", "heading2", "heading3",
    "paragraph", "list_item", "table",
    "menu_path",      # "파일 > 설정 > 프로필" 같은 경로
    "form_field",     # 입력 필드 설명
    "authentication", # 로그인/권한 섹션
    "code_block", "note", "warning", ...
]

# 정규식 기반 감지
HEADING_PATTERNS = [
    r"제\s*\d+\s*장",   # "제 1 장"
    r"\d+\.\s*[가-힣]",  # "1. 개요"
    r"^[A-Z][A-Z\s]+$",  # 영문 대문자
]

# Vision LLM 이미지 캡션
async def _describe_images_with_vlm(images):
    # base64 인코딩 → multipart 요청 → 캡션 텍스트
```

**Stage 1: `extract-ontology-data` — 병렬 팩트 추출**
```python
async def extract_facts(chunks):
    tasks = [
        _call_llm_stage1(chunk, system_prompt)
        for chunk in chunks
    ]
    # 병렬 LLM 호출 (chunk 10개라면 순차 대비 10배 빠름)
    results = await asyncio.gather(*tasks, return_exceptions=True)
    # {classes: [...], properties: [...], relationships: [...], individuals: [...]}
    return merge_facts(results)
```

**Stage 2: `json-to-ttl` — TTL 생성**
```python
async def json_to_ttl(merged_facts):
    # 2-shot prompting (예시 2개 + 실제 요청)
    ttl = await _ttl_call_llm(stage2_system_prompt, merged_facts)
    # rdflib 검증
    g = rdflib.Graph()
    g.parse(data=ttl, format='turtle')
    return ttl
```

**핵심 해결 기법**

1. **Token Budget 동적 관리**
```python
SYSTEM_TOKENS = estimate_tokens(system_prompt)
OUTPUT_BUFFER = 500
MAX_CONTEXT = 8000
available = MAX_CONTEXT - SYSTEM_TOKENS - OUTPUT_BUFFER

for chunk in chunks:
    if estimate_tokens(chunk) > available:
        # 동적 재청킹
        sub_chunks = _ttl_split_text(chunk, available)
        ...
```

2. **Reasoning Content Fallback**
```python
async def _ttl_call_llm(system_prompt, user_input, timeout_sec=600):
    response = await asyncio.wait_for(
        loop.run_in_executor(None, lambda: model.invoke([
            SystemMessage(system_prompt),
            HumanMessage(user_input)
        ])),
        timeout=timeout_sec
    )

    content = clean(response.content)

    # Fallback 1: additional_kwargs.reasoning_content
    if not content:
        reasoning = response.additional_kwargs.get("reasoning_content", "")
        if reasoning: return reasoning

    # Fallback 2: <think>...</think> 태그 안
    if not content:
        match = re.search(r"<think>(.*?)</think>", response.content, re.DOTALL)
        if match: return match.group(1)

    return content
```

3. **비동기 Job Pattern**
```python
_extract_facts_jobs: Dict[str, Job] = {}

class Job:
    status: Literal["processing", "done", "error", "cancelled"]
    progress: str  # "3/10"
    task: asyncio.Task
    created_at: datetime
    result: Any

# Background cleanup
async def _cleanup_old_jobs():
    now = datetime.now()
    expired = [k for k, j in _jobs.items() if now - j.created_at > timedelta(minutes=10)]
    for k in expired:
        _jobs[k].task.cancel()
        del _jobs[k]

# 취소 지원
@router.delete("/extract-ontology-data/{job_id}")
async def cancel(job_id):
    job = _jobs.get(job_id)
    if job: job.task.cancel()
```

4. **Fuseki Skolemization 역변환**
```python
# GSP download 시 blank node가 URI로 변환됨
# _:b0 → <host/dataset/.well-known/genid/b0>
# 역변환:
re.sub(
    r"<https?://[^/]+/[^/]+/\.well-known/genid/([^>]+)>",
    r"_:\1",
    ttl
)
```

#### 결과
- **200페이지 PDF 매뉴얼도 안정적으로 처리** (사내 실사용)
- **병렬 처리로 소요 시간 대폭 단축**
- Reasoning fallback 도입 후 "빈 응답" 에러 사실상 0건
- Job 시스템으로 **진행 상태 폴링 + 취소 + 자동 정리** 모두 지원

---

### 🔌 Challenge 6. Multi-Backend 추상화 레이어 설계

#### 상황
초기에는 Jena Fuseki만 썼지만, 고객사 요구에 따라 **다른 Triple Store로 교체 가능해야** 함. 벤더 락인을 피하고 싶었음.

#### 해결: ABC + Factory + Capability Flags

```python
# base.py — 추상 기반 클래스
class TripleStoreClient(ABC):
    @abstractmethod
    async def ping(self) -> bool: ...

    @abstractmethod
    async def upload_ttl(self, dataset: str, graph_uri: str, ttl: str): ...

    @abstractmethod
    async def download_ttl(self, dataset: str, graph_uri: str) -> str: ...

    @abstractmethod
    async def query(self, dataset: str, sparql: str) -> QueryResult: ...

    @property
    @abstractmethod
    def backend_name(self) -> str: ...

    @property
    @abstractmethod
    def capabilities(self) -> Dict[str, bool]:
        """{'reasoning': True, 'validation': True, 'dataset_management': False}"""

    # 공통 구현 (하위 클래스가 오버라이드 가능)
    async def insert_triples(self, dataset, triples):
        nt = self._triples_to_nt(triples)
        sparql = f"INSERT DATA {{ {nt} }}"
        return await self.update(dataset, sparql)

    # 선택적 기능 — 지원 안 하면 예외
    async def validate_ttl(self, ttl: str):
        raise UnsupportedOperationError(f"{self.backend_name} does not support validation")
```

```python
# factory.py — 싱글톤 팩토리
_client_instance: TripleStoreClient = None

def get_client() -> TripleStoreClient:
    global _client_instance
    if _client_instance: return _client_instance

    backend_type = config.GRAPHDB_TYPE  # 1=Fuseki, 2=GraphDB, ...

    if backend_type == 1:
        _client_instance = FusekiClient(url=config.GRAPHDB_URL)
    elif backend_type == 2:
        _client_instance = GraphDBClient(url=config.GRAPHDB_URL)
    # ...
    else:
        logger.warning(f"Unknown backend {backend_type}, falling back to Fuseki")
        _client_instance = FusekiClient(url=config.GRAPHDB_URL)

    return _client_instance
```

**현재 구현 상태**
- **Fuseki**: 완전 구현 (reasoning + validation + skolemization)
- **GraphDB / Stardog / Oxigraph**: 스텁 (인터페이스만)
- **RDFox**: 스켈레톤

**API 레벨에서의 capability 체크**
```python
@router.post("/datasets/{name}/inferred")
async def get_inferred(name: str):
    client = get_client()
    if not client.capabilities.get("reasoning"):
        raise HTTPException(501, f"{client.backend_name} does not support reasoning")
    return await client.get_inferred_triples(name, ...)
```

#### 결과
- 백엔드 교체 시 API 레이어는 **단 한 줄도 변경 없음**
- 테스트 용이성 향상 (`reset_client()` 주입)
- 향후 GraphDB/Stardog 지원 확장 용이

---

### 🚀 Challenge 7. Jenkins 다중 브랜치 자동 배포 파이프라인

#### 상황
Trinity 제품은 버전별 브랜치(500/600/700 등)로 관리되는데, 온톨로지 디자이너 하나를 **여러 브랜치에 동일 산출물**로 배포해야 했음. 기존 파이프라인은 빌드마다 버전이 달라져 브랜치 간 불일치 발생.

#### 해결
- **Active Choices Plugin**으로 Package 저장소 브랜치를 체크박스로 동적 로드
- **빌드 1회 → Publish 디렉터리 준비 → 각 브랜치에 순차 배포** 패턴 재설계
- `VERSION_BUMP`은 1회만 실행, 나머지는 `none`으로 동일 버전 유지
- `SKIP_DEPLOY` 플래그로 배포 없이 빌드 검증 모드 지원
- Script Approval, Git credential 연동 디버깅

#### 결과
- 3~4개 브랜치에 동일 산출물을 1번 빌드로 일괄 배포
- 배포 시간 단축 + 브랜치 간 버전 일관성 확보

---

## 📐 아키텍처 의사결정

| 선택 | 대안 | 왜 이걸 골랐나 |
|---|---|---|
| **Sigma.js** | Cytoscape · D3 · vis.js | WebGL 기반 성능, 대규모 그래프, 명확한 이벤트 API |
| **Graphology** | Direct Sigma API | 데이터 모델 분리, BFS/degree 등 알고리즘 제공 |
| **ForceAtlas2** | cose · noverlap | Sigma 같은 팀, 튜닝 파라미터 풍부, Barnes-Hut 최적화 |
| **Jena Fuseki** | GraphDB · Blazegraph · Stardog | 오픈소스, TDB2 영속, SPARQL 1.1 표준, 추론기 내장 |
| **FastAPI** | Flask · Django | async/await, Pydantic 자동 문서화, LangChain 궁합 |
| **GPT-OSS 120B** | OpenAI API · Claude API | 온프레미스 · 비용·보안 · Bimatrix Provider 통합 |
| **BGE-M3 Embedding** | OpenAI text-embedding · Cohere | 다국어(한/영/일) 성능, 온프레미스 가능, 오픈 라이선스 |
| **Elasticsearch** | Pinecone · Weaviate · Qdrant | 기존 사내 ES 인프라 재활용, 관리 비용 최소화 |
| **TripleStoreClient ABC** | Fuseki 직접 의존 | 벤더 락인 회피, 테스트 용이성, 확장 가능성 |
| **Vanilla TS** | React · Vue | 제품 WAS(JSP) 통합, 번들 최소화, 성능 |
| **N3.js** | rdflib.js | 경량, tree-shakable, 브라우저 친화 |

---

## 🌍 국제화 (i18n)

- **한국어 · 영어 · 일본어** 완전 지원 (각 945 라인 리소스)
- `i18next` + JSON 리소스
- 런타임 언어 전환 (새로고침 없이)
- 다국어 라벨이 온톨로지 데이터 자체에도 반영 (`@ko`, `@en`, `@jp`)
- LLM 번역 파이프라인으로 기존 온톨로지 일괄 번역 기능

---

## 🛠️ 엔지니어링 프로세스

### 코드 품질
- **TypeScript strict mode** (frontend)
- **Pydantic 기반 API 스키마 검증** (backend)
- **수동 테스트 시나리오 문서화** (14 슬라이드 + 54개 가이드 스크린샷)

### 디버깅 방법론
- **이벤트 플로우 전수 분석** — Sigma 이벤트 핸들러 19개 이슈 발견 · 수정
- **재현 시나리오 기반 가드 추가** — 타이밍 이슈는 타임스탬프 가드, 상태 충돌은 플래그로 차단
- **Auto-Memory 활용** — Jena Fuseki 설정 패턴 등 hard-won knowledge를 외부 메모리에 저장해 컨텍스트 재사용

### 배포
- Docker 이미지 기반 Trinity 제품 통합
- Jenkins CI/CD (GitLab + Active Choices Plugin)
- 라이선스 체크 연동 (사내 MAF 프레임워크)

---

## 📚 사용자 교육 & 문서

- **프레젠테이션 스크립트** (14 슬라이드, 온보딩 시나리오 기반)
- **54개 스크린샷 가이드** (기능별 단계 안내)
- **이론 자료** (도메인 전문가·비개발자용 온톨로지 기초 설명)
- **AI Helper 데모 시나리오**
- **다국어 도움말 HTML** (제품 내장)
- **FAQ · 키보드 단축키 레퍼런스**

---

## 💡 배운 점 · 회고

### 잘한 것
- **도메인 중심 설계**: "설계자가 OWL 용어를 몰라도 되도록"이라는 원칙 고수
- **추상화의 힘**: TripleStoreClient ABC 덕분에 Fuseki 외 백엔드 확장 경로 확보
- **하이브리드 RAG 접근**: Vector Search만의 한계를 Knowledge Graph로 보강
- **LLM을 "블랙박스"가 아닌 "파이프라인"으로**: 2-stage 분리 · reasoning fallback · 토큰 버짓 · 검증 단계
- **1인 풀스택의 장점**: 프론트 상태와 백엔드 API 계약을 한 머리에 담고 있어 인터페이스 불일치 제로
- **기술 부채 즉시 정리**: 19개 이벤트 버그를 한 번에 찾아 수정

### 아쉬운 것 · 개선하고 싶은 것
- **E2E 테스트 부재**: Playwright 도입 검토 중
- **Kubernetes 미경험**: Docker + WAS 통합 방식 → 클라우드 네이티브 전환은 다음 과제
- **성능 프로파일링**: 대규모 그래프(500+ 노드)에서의 렌더 병목을 체계적으로 측정해보고 싶음
- **LLM 품질 모니터링**: 생성된 TTL의 품질을 지속적으로 평가하는 메트릭/로깅 시스템 (LangFuse 같은) 아직 미도입
- **End-to-end GraphRAG 평가**: 생성된 온톨로지가 실제 RAG 정확도 개선에 얼마나 기여하는지 측정은 다음 단계

### 이 프로젝트가 남긴 것
1. **지식 그래프 도메인 전문성** — RDF/OWL/SPARQL/Jena 생태계 실전 이해
2. **LLM 서비스 개발 실전** — LangChain 메시지, 프롬프트 설계, 파싱 실패 대응, 긴 출력 처리, 토큰 버짓
3. **하이브리드 검색 시스템** — Vector Search (BGE-M3) + Knowledge Graph 결합 파이프라인
4. **엔터프라이즈 배포 경험** — WAS 통합, 다국어, 다중 브랜치 CI/CD, 라이선스 연동
5. **대규모 시각화 성능 튜닝** — WebGL, 레이아웃 알고리즘, 이벤트 시스템 디버깅
6. **백엔드 추상화 설계** — ABC + Factory + Capability Flags 패턴

---

## 🔗 관련 링크 및 참고

- **기술 영역**: Knowledge Graph · OWL · SPARQL · LangChain · FastAPI · Sigma.js · ElasticSearch · BGE-M3 · GraphRAG
- **연관 제품**: BIMATRIX Trinity (LangFlow 기반 AI 워크플로우 플랫폼)
- **관련 개념**: Knowledge Graph RAG · Ontology-based RAG · Text-to-SPARQL · Hybrid Search

> 💬 스크린샷 및 상세 코드는 사내/고객사 배포 제품 특성상 **개별 요청 시 제공** 가능합니다.
