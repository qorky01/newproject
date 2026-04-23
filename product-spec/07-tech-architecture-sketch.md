# 07 · Technical Architecture Sketch

> **버전**: v0.1 (W1 초안) · **작성일**: 2026-04-23
> **카운터**: H: 4 · V: 0 · X: 0 · OPEN: 4
> **MR 민감도**: Low (고객 응답과 무관하게 내부 기술 선택)

---

## 본 문서의 위상

**결정 문서 아님 — 비교 문서**. MR이 끝나고 MVP 기능이 확정(`05`)된 후에 정식 결정한다. 지금은:

1. 후보 기술 스택을 **장단점 비교표**로 정리
2. 결정이 시급한 항목은 **ADR 초안**으로 `_decisions/` 에 배치
3. 결정 시점(언제 확정할지)을 명시

**Out of scope**: 배포 토폴로지 세부, Terraform/IaC, k8s 매니페스트, 모니터링 스택. 모두 개발 phase 시작 시 결정.

---

## 상위 아키텍처 (개념도)

```
┌───────────────────────────────────────────────┐
│                  Web App (SPA)                │
│  — React / Next.js 후보                        │
│  — Excel-like 셀 편집, 유사 배합 검색 UI       │
└──────────────┬────────────────────────────────┘
               │ HTTPS
               ∨
┌───────────────────────────────────────────────┐
│                 API / BFF                     │
│  — FastAPI vs Django 후보                      │
│  — 인증, 엑셀 임포트, CRUD, 유사도 검색         │
└──────┬─────────────┬──────────────────┬───────┘
       │             │                  │
       ∨             ∨                  ∨
┌──────────┐  ┌─────────────┐   ┌──────────────────┐
│ Postgres │  │  Object     │   │   ML / Worker    │
│ (메타·관 │  │  Storage    │   │  RDKit descriptor │
│  계 데이 │  │ (엑셀 원본· │   │  유사도 · 예측    │
│   터)    │  │  COA PDF)   │   │  (async queue)    │
└──────────┘  └─────────────┘   └──────────────────┘
```

MR 민감 항목(페르소나·MVP 기능)이 확정되면 이 다이어그램의 **Worker 부분과 ML 스택 세부**가 크게 바뀔 수 있다. Web/API/DB 축은 상대적으로 안정.

---

## 기술 선택 비교표

### 축 1: Web 프레임워크

| 후보 | 장점 | 단점 | 적합성 |
|---|---|---|---|
| **Next.js (React)** | SSR/SSG, 생태계 최대, 채용 용이, Vercel 배포 편리 | 오버엔지니어링 위험, 초기 학습 비용 | 🟢 **1순위** |
| **Remix** | 단순한 데이터 로딩 모델, 서버 중심 | 생태계 Next 대비 작음 | 🟡 2순위 |
| **Vite + React SPA** | 가볍고 빠름 | SSR 없음, SEO 약함 (B2B엔 덜 중요) | 🟡 3순위 |
| **SvelteKit** | 번들 작고 빠름, 학습 쉬움 | 채용 풀 작음, 한국 React 우위 | ❌ 제외 |

**현재 유력**: Next.js. 단, Excel-like 셀 편집이 1급 기능이므로 **table editor 라이브러리 선택이 프레임워크보다 중요할 수 있다** `[H]`. 후보: `ag-grid`, `handsontable`, `glide-data-grid`, `tanstack-table + 자체 cell editor`.

### 축 2: API / Backend

| 후보 | 장점 | 단점 | 적합성 |
|---|---|---|---|
| **FastAPI (Python)** | RDKit 같은 과학 파이썬 생태계와 **같은 프로세스에서** 호출 가능, 타입 힌트, pydantic | ORM(SQLAlchemy) 학습 곡선, async 관리 주의 | 🟢 **1순위** |
| **Django (Python)** | Admin, ORM, Auth 다 내장 → 초기 속도 빠름 | DRF + RDKit 비동기 조합이 다소 무거움 | 🟡 2순위 (관리자 도구 중요하면 재검토) |
| **NestJS (Node)** | 프런트와 언어 통일, 타입 완전 | RDKit 호출은 별도 서비스 필요 (IPC/HTTP) | ❌ 제외 |
| **Go (Gin/Echo)** | 성능, 배포 간단 | 과학 라이브러리 빈약, 팀 역량 미보유 `[H]` | ❌ 제외 |

**현재 유력**: FastAPI. 근거: RDKit(descriptor) · 유사도 검색 · 간단한 ML 예측을 **같은 Python 런타임**에서 처리하면 MVP 속도 이득 큼. Year 2에서 ML 워크로드 커지면 워커 분리.

### 축 3: 데이터베이스

| 후보 | 장점 | 단점 | 적합성 |
|---|---|---|---|
| **PostgreSQL** | JSONB로 유연 스키마, pgvector로 descriptor 유사도, 생태계 성숙 | 운영 노하우 필요 | 🟢 **1순위** |
| **DuckDB** | 분석 쿼리 매우 빠름, 파일 기반 간편 | 멀티유저 동시 쓰기 약함, SaaS 맞지 않음 | 🟡 로컬/배치 분석 용도 (유사 배합 재계산 배치) |
| **MySQL** | 친숙함 | JSONB/벡터 검색 약함, pgvector 동급 없음 | ❌ 제외 |
| **MongoDB** | 스키마 자유 | 관계 쿼리 필요한 도메인과 안 맞음 | ❌ 제외 |

**현재 유력**: Postgres (메인) + DuckDB (배치 분석 용도 선택적). **pgvector 채택 여부**는 ADR 대상 — 아래 ADR-002 참조.

### 축 4: 분자 Descriptor 계산

| 후보 | 장점 | 단점 |
|---|---|---|
| **RDKit** | Python 1급, 표준, 오픈소스, 커뮤니티 대, MIT 라이선스 | descriptor 세트가 "모든 것"은 아님 |
| **Mordred** | 1800+ descriptor, RDKit 위에서 동작 | 계산 속도 느림, 결측/NaN 많음 |
| **OpenBabel** | 포맷 변환 최강 | descriptor 자체는 RDKit에 미치지 못 함 |
| **CDK (Java)** | 풍부한 descriptor | JVM 호출 필요, 인프라 복잡도 증가 |
| **상용 (Dragon, MOE)** | 정교 | 유료, 라이선스 족쇄 — **BIOVIA 리셀러 계약 리스크 검토 필요** [DEP:MR-01 §부록B] |

**현재 유력**: RDKit 기본 + 선택적 Mordred. 결정은 ADR-001 참조 (아래).

### 축 5: 유사도 / 검색

| 후보 | 장점 | 단점 |
|---|---|---|
| **pgvector (Postgres 확장)** | DB 내 완결, 운영 단순 | 대규모(>10M)에서 성능 둔화 — v1 타겟엔 충분 `[H]` |
| **Qdrant / Weaviate** | 벡터 DB 전용, 메타데이터 필터 강함 | 별도 인프라, 운영 복잡도 증가 |
| **FAISS in-memory** | 압도적 속도 | 상태 없음, 재시작마다 로드 필요 |
| **Elasticsearch dense_vector** | 기존 텍스트 검색 결합 | 비용·운영 부담 |

**현재 유력**: pgvector. Year 1 규모(고객 10곳 × 수천 Formulation)에 충분 `[H]`.

### 축 6: ML / 예측 모델

| 후보 | 장점 | 단점 |
|---|---|---|
| **scikit-learn + LightGBM/XGBoost** | 성숙, 소량 데이터에 강함, 해석 가능 | SOTA 아님 (배합 데이터는 수백~수천 건이라 OK) |
| **PyTorch 커스텀 MLP** | 유연 | 오버엔지니어링, 데이터 부족 시 오히려 열세 |
| **Intellegens Alchemite 유사 (결측 학습)** | 소량/결측 데이터 강점 [DEP:MR-04 §Intellegens] | 자체 구현 난이도 높음 |

**현재 유력**: v1은 **LightGBM + scikit-learn 파이프라인**. Year 2에서 결측 데이터 처리 고도화.

### 축 7: 엑셀 임포트

| 후보 | 장점 | 단점 |
|---|---|---|
| **openpyxl (Python)** | 표준, xlsx 완전 지원 | 대용량 느림 |
| **pandas.read_excel** | 편리 | 메모리 비효율, 병합셀 처리 한계 |
| **python-calamine (Rust 바인딩)** | 매우 빠름, 병합셀/수식 정확 | 생태계 작음 |
| **Client-side (SheetJS)** | 브라우저에서 즉시 파싱, 서버 부하 감소 | 대용량 한계, 로직 중복 |

**현재 유력**: `python-calamine` 주력 + `openpyxl` 폴백. 클라이언트 프리뷰에 SheetJS.

### 축 8: 인증 / 조직

| 후보 | 장점 | 단점 |
|---|---|---|
| **Clerk / WorkOS** | SSO/SAML 즉시, 운영 부담 0 | 비용 (per-user), 락인 |
| **Auth0** | 성숙 | 비용 증가 패턴 |
| **자체 구현 (FastAPI + python-jose)** | 무료, 완전 제어 | SAML/SSO 구현 시간 대 |
| **Supabase Auth** | Postgres와 통합, 무료 티어 | Postgres 운영과 묶임 |

**현재 유력**: Year 1 자체 구현(이메일/비밀번호 + TOTP) → Year 2 SSO 요청 들어오는 시점에 WorkOS 도입. 과투자 회피.

### 축 9: 배포 / 인프라

| 후보 | 장점 | 단점 |
|---|---|---|
| **AWS (ECS/Fargate + RDS)** | 한국 리전, 엔터프라이즈 신뢰 | 복잡, 과투자 위험 |
| **Fly.io** | 간단, 글로벌 | 한국 리전 없음 → 엔터프라이즈 반대 가능 |
| **GCP (Cloud Run + Cloud SQL)** | 서울 리전, 관리 간편 | AWS 대비 한국 고객 익숙도 낮음 `[H]` |
| **NHN/네이버/KT Cloud** | 한국 엔터프라이즈 신뢰 | 글로벌 생태계 약함 |

**현재 유력**: AWS 서울 리전. 한국 중견 엔터프라이즈는 AWS를 기본 수용한다는 가설 `[H]` — MR W3 보안/IT 질문 항목에서 검증.

---

## 가설 (본 문서 레벨)

| ID | 가설 | 검증 방법 | 반증 시 |
|---|---|---|---|
| TEC-H1 | [H] v1 규모에서 pgvector는 유사도 검색 응답 300ms 이내 달성 가능 | POC (10,000 Formulation × 300-dim vector) | Qdrant 도입 |
| TEC-H2 | [H] Excel-like 셀 편집은 `ag-grid` 또는 `handsontable` 하나로 충분 | UI 프로토 30분 스파이크 | 자체 cell editor 구현 |
| TEC-H3 | [H] 한국 중견 IT는 AWS 서울 리전 SaaS를 무리 없이 수용한다 | MR W3 보안/IT 질문 | 온프레미스/VPC peering 옵션 추가 검토 |
| TEC-H4 | [H] RDKit descriptor 세트 (~200개)가 물성 예측에 유의한 feature를 충분히 제공한다 | 오픈 데이터셋 POC (예: polymer property prediction) | Mordred 보강 또는 도메인 feature 수동 추가 |

---

## 결정 시점 로드맵

| 결정 항목 | 누구 | 언제 | 산출물 |
|---|---|---|---|
| Web 프레임워크 확정 (Next vs Remix vs Vite) | 엔지니어링 리드 | MVP 킥오프 직전 | ADR |
| Backend 확정 (FastAPI vs Django) | 엔지니어링 리드 | MVP 킥오프 직전 | ADR |
| 벡터 검색 (pgvector vs Qdrant) | 엔지니어링 리드 | POC 후 (W2) | **ADR-002** (초안 작성 예정) |
| Descriptor 스택 (RDKit vs +Mordred) | 엔지니어링 리드 | W2 POC | **ADR-001** (본 W1 작성) |
| 배포 (AWS vs GCP) | 창업자 + IT 고객 청취 | MR W3 후 | ADR |
| Auth (자체 vs Clerk) | 엔지니어링 리드 | MVP 출시 1주 전 | ADR |

---

## ADR-001 초안: RDKit 중심 + Mordred 선택 보강

W1 시점에 초안 작성 → `_decisions/adr-001-rdkit-vs-mordred.md`에 배치.

---

## 리스크 & 오픈 이슈

1. **BIOVIA 리셀러 계약 제약** [DEP:MR-01 §부록B]: 자사 제품이 특정 상용 라이브러리/API 사용 시 계약 위반 가능성. 모든 의존성은 **오픈소스 또는 독립 벤더**로 한정. 법무 검토 필요 — W4까지.
2. **RDKit 계산 비용**: Substance 수가 커지면 descriptor 재계산 배치 비용 관리 필요. 초기 캐시 전략 단순화 가능하나 Year 2 이후 재설계 가능성.
3. **엑셀 임포트 엣지 케이스**: 병합셀 · 단위 혼재 · 오타 · 한글/영문 컬럼명 혼재. **파서보다 컬럼 매핑 UI**에 투자해야 한다는 `[H]` (MR W3 실제 파일 관찰로 검증).
4. **한국어 화학 표기**: "이산화티타늄" vs "TiO2" vs "티타늄디옥사이드" 별칭 해결은 Substance 모델의 `aliases`에 의존. 별도 사전 관리 툴 필요할 수도.

---

## Market Research 반영 로그

| 주차 | 반영 내용 |
|---|---|
| W1 | MR-04 §Intellegens 소량 데이터 강점 → ML 축6에 언급. MR-01 §부록B BIOVIA 리셀러 제약 → 의존성 제약 리스크. MR-04 §Albert Invent 깊이 → Substance 마스터 설계 참고. |
| W2 | (pending — POC 2건: pgvector 벤치, ag-grid 스파이크) |
| W3 | (pending — TEC-H3 IT 수용도 검증) |
| W4 final | (pending) |
