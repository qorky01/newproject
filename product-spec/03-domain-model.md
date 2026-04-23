# 03 · Domain Model

> **버전**: v0.1 (W1 초안) · **작성일**: 2026-04-23
> **카운터**: H: 6 · V: 0 · X: 0 · OPEN: 6
> **MR 민감도**: Medium (엔터티 이름·관계는 MR 무관, 속성 세부는 인터뷰로 보강)

---

## 목적

배합 R&D 데이터의 **논리 엔터티와 관계**를 정의한다. 본 문서는:
- 엔지니어가 DB 스키마를 만들 때 출발점
- UI 설계자가 화면/폼을 설계할 때 공통 어휘
- 영업/컨설팅이 고객 설명 시 일관 용어

**본 문서 범위 아님**: SQL DDL, 인덱스, 물리 스키마, API 페이로드. 논리 모델까지만.

---

## 핵심 엔터티 6개 + 관계 (논리 ERD, 텍스트)

```
┌──────────────┐   1    ┌──────────────┐   1    ┌──────────────┐
│  Substance   │◇──────<│    Grade     │◇──────<│     Lot      │
│  (논리 원료) │        │ (공급사 규격) │        │  (생산 배치)  │
└──────┬───────┘        └──────┬───────┘        └──────┬───────┘
       │                        │                        │
       │ 1                      │ 1                      │ 1
       │                        │                        │
       │                        │                        │
       │     ┌──────────────────┴────────────┐           │
       │     │                                │          │
       │   N │           ComponentRef         │ N        │
       └───>│ (Formulation 내 원료 참조 엔트리) │<─────────┘
            │  substance_id? grade_id? lot_id? │
            │  + 중량/비율                       │
            └────────────┬────────────────────┘
                         │ N
                         │
                         ∨ 1
                ┌──────────────────┐      1        N    ┌──────────────────┐
                │   Formulation    │<──────────────────>│     Experiment   │
                │   (배합 레시피)  │                     │ (1회 제조/측정)   │
                └──────────────────┘                    └────────┬─────────┘
                                                                 │ 1
                                                                 │
                                                                 │ N
                                                          ┌──────┴───────┐
                                                          │   Property   │
                                                          │ (측정 물성치) │
                                                          └──────────────┘
```

(다이어그램은 후속 이미지로 `_assets/erd-v0.1.png` 추가 예정.)

---

## 엔터티 상세

### 1. Substance (논리 원료)

**정의**: 공급사·Lot과 무관하게 **논리적으로 동일한 화학물질**. 레시피 재현성·유사도 검색의 1차 단위.

| 속성 | 타입 | 설명 | 비고 |
|---|---|---|---|
| `id` | UUID | PK | |
| `name` | str | 표시명 (예: "TiO2", "계면활성제 A") | |
| `cas_number` | str? | CAS 번호 | NULL 허용 (혼합물/영업비밀) |
| `smiles` | str? | 표준 SMILES | descriptor 계산 입력 |
| `inchi_key` | str? | InChIKey | 중복 검출 키 |
| `aliases` | str[] | 별칭 (동의어) | 검색 지원 |
| `category` | enum | `polymer` / `solvent` / `pigment` / `surfactant` / `additive` / `other` | UI 필터링 |
| `internal_code` | str? | 고객사 내부 코드 | 엑셀 이관 지원 |
| `descriptor_vector` | float[] | RDKit descriptor 캐시 | SMILES 변경 시 재계산 |
| `created_by` / `created_at` | — | 감사 | |

**핵심 가설**:
- `[H]` 고객사는 동일 Substance를 중복 등록하는 문제가 많다 → **별칭 병합 UI 필수**
- `[H]` SMILES 없는 영업비밀 원료가 전체의 20~40% 차지한다 [DEP:MR-03] → descriptor 미계산 케이스 UX 필요

### 2. Grade (공급사 규격)

**정의**: Substance의 **상용 규격 변종**. 동일 Substance라도 공급사/Grade에 따라 실제 물성 상이.

| 속성 | 타입 | 설명 |
|---|---|---|
| `id` | UUID | PK |
| `substance_id` | FK → Substance | |
| `supplier` | str | 공급사 (예: "Evonik", "Kronos") |
| `grade_name` | str | 규격명 (예: "R-902", "Silbyk 9015") |
| `spec_sheet_url` | str? | 공급사 COA/TDS 링크 |
| `typical_properties` | json | 공급사 공시 물성 (점도/순도/평균입경 등) |
| `status` | enum | `active` / `discontinued` / `evaluating` |
| `internal_code` | str? | 고객사 내부 Grade 코드 |

**왜 분리하는가**: Albert Invent 수준의 **원재료 마스터 깊이**를 확보하기 위함 [DEP:MR-04 §차별화축4]. Grade 단위로 유사 배합 검색/추천하면 Substance 단위보다 정밀도 향상 `[H]`.

### 3. Lot (생산 배치)

**정의**: Grade의 **특정 생산 로트**. 시간축 변수를 담는다 — 동일 Grade라도 Lot별로 실측치 편차.

| 속성 | 타입 | 설명 |
|---|---|---|
| `id` | UUID | PK |
| `grade_id` | FK → Grade | |
| `lot_number` | str | 공급사 Lot 번호 |
| `received_at` | date | 입고일 |
| `expiry_at` | date? | 유효기한 |
| `measured_properties` | json | 실측 COA (공시 대비 실측) |
| `quantity_remaining` | float? | 재고 수량 (optional — LIMS 영역 최소화) |

**설계 원칙**: Lot은 **선택적 레이어**. 고객이 Lot 추적 안 하는 경우 Formulation은 Grade 또는 Substance 수준에서 바로 참조 가능 (ComponentRef 유연 참조 참조).

**Out of scope**: 재고 소진·바코드 스캔·출고 관리 → LIMS 영역 [DEP:MR-04 §Part2]. `quantity_remaining`은 **참조용**만.

### 4. Formulation (배합 레시피)

**정의**: 원료 + 비율 + 공정 조건의 **정적 레시피**. 실제로 제조/측정하면 Experiment가 생성됨.

| 속성 | 타입 | 설명 |
|---|---|---|
| `id` | UUID | PK |
| `name` | str | 레시피명 (예: "CL-2026-0412 v3") |
| `version` | int | 버전 (같은 이름 내 증가) |
| `parent_formulation_id` | FK? → Formulation | 분기/복제 추적 |
| `components` | ComponentRef[] | 원료 리스트 (1~N) |
| `process_parameters` | json | 온도·시간·교반속도 등 |
| `target_properties` | json? | 목표 물성 (예: `점도: 2000±500 cP`) |
| `notes` | text | 자유 메모 (암묵지 보존) [DEP:MR-01 §엑셀강점9] |
| `created_by` / `created_at` | — | |

**ComponentRef 서브엔터티** (Formulation 내부):

| 속성 | 타입 | 설명 |
|---|---|---|
| `substance_id` | FK? | 3층 중 하나는 반드시 지정 |
| `grade_id` | FK? | |
| `lot_id` | FK? | |
| `amount` | float | 수치 |
| `unit` | enum | `wt%` / `phr` / `부` / `g` / `mol%` / `vol%` [DEP:MR-01 §Part3 패턴4] |
| `role` | str? | "주재료" / "가교제" / "용매" 등 자유 라벨 |

**단위 다양성 필수**: MR-01 §스키마근본주의 실패 패턴에서 wt%·phr·부·mol% 혼재가 현장의 실재임. **단위 강제 금지**, 변환 유틸 제공.

### 5. Experiment (실험 실행)

**정의**: Formulation을 **실제로 1회 제조·측정**한 이벤트. 결과 Property 세트 생성.

| 속성 | 타입 | 설명 |
|---|---|---|
| `id` | UUID | PK |
| `formulation_id` | FK → Formulation | |
| `executed_at` | date | 실행일 |
| `executed_by` | user | 실험자 |
| `actual_components` | ComponentRef[] | 실제 투입량 (레시피와 오차 추적) |
| `actual_process` | json | 실제 공정 조건 |
| `batch_size` | float? | 제조량 (g/kg) |
| `notes` | text | 실험자 관찰 메모 |
| `status` | enum | `planned` / `completed` / `failed` |

**왜 Formulation과 분리**: 같은 레시피를 여러 번 실행해도 결과가 다를 수 있음 (Lot 변동·환경 차이). AI 예측의 입력은 "Formulation+실측 Component", 출력은 "Property"로 분리해야 학습 가능 `[H]`.

### 6. Property (측정 물성)

**정의**: Experiment 1회에서 측정된 **단일 물성치**. 한 Experiment는 여러 Property를 갖는다.

| 속성 | 타입 | 설명 |
|---|---|---|
| `id` | UUID | PK |
| `experiment_id` | FK → Experiment | |
| `property_type` | enum/FK | `viscosity` / `tensile_strength` / `pH` / `gloss` / ... (카탈로그) |
| `value` | float | 측정치 |
| `unit` | str | `cP` / `MPa` / `—` 등 |
| `condition` | json? | 측정 조건 (`25°C`, `shear_rate=10s^-1`) |
| `method` | str? | 측정 방법 (`ASTM D445`, `자체법`) |
| `instrument` | str? | 기기명 (optional) |
| `specimen` | json? | 시편 정보 (두께·형상) |
| `measured_at` | datetime | |
| `measured_by` | user | |

**PropertyType 카탈로그**: 초기 공통 타입 50~100개 기본 제공 + 고객 **자체 타입 추가 가능** [DEP:MR-01 §Part5 원칙, Win 4].

---

## 관계 요약표

| from | rel | to | 카디널리티 | 비고 |
|---|---|---|---|---|
| Substance | has | Grade | 1 : N | Grade 0개 허용 (사내 개발 원료) |
| Grade | has | Lot | 1 : N | Lot 추적 안 하는 고객 → 0개 허용 |
| Formulation | contains | ComponentRef | 1 : N | 1개 이상 필수 |
| ComponentRef | refs | Substance/Grade/Lot | N : 1 | 셋 중 **정확히 하나** |
| Formulation | has version ancestor | Formulation | N : 1 | parent_formulation_id |
| Formulation | executed as | Experiment | 1 : N | 1 Formulation 여러 실행 |
| Experiment | has | Property | 1 : N | 0개 허용 (실패 실험) |

---

## 설계 원칙

### 원칙 1: 3층(Substance / Grade / Lot)은 **선택적**으로 펼친다
현장은 Level 2~3에 있다 [DEP:MR-02]. Lot 추적 안 하는 고객이 다수. 제품은 **Substance만 입력해도 작동**해야 하고, Grade/Lot은 나중에 필요할 때 enrichment.

### 원칙 2: 단위 강제 금지
wt%·phr·부·mol%·vol% 공존 허용. UI는 기본 단위(고객 설정)만 보여주되 다른 단위로 입력된 과거 데이터 변환/표시 지원.

### 원칙 3: 엑셀 임포트가 1급 입력 방식
ComponentRef는 엑셀 행 하나에 매핑 가능해야 함. 컬럼 매핑 UI로 Substance name / Grade / Lot / amount / unit을 드래그 매핑.

### 원칙 4: descriptor는 캐시, 원천 아님
Substance에 `descriptor_vector` 캐시. SMILES 변경/RDKit 버전 업그레이드 시 **무효화 → 재계산** 파이프라인 필요. (구현 세부는 `07-tech-architecture`.)

### 원칙 5: "암묵지" 속성 보존
`Formulation.notes`, `Experiment.notes`는 **긴 free text** 지원. 파워유저가 엑셀 셀 색칠/메모로 표현하던 것을 수용 [DEP:MR-01 §엑셀강점9·12].

### 원칙 6: Property는 시간·조건 차원을 가진다
같은 `viscosity`라도 온도·shear rate 다르면 다른 데이터. `condition` JSON 필드로 차원 보존. AI 학습 시 feature로 투입 가능.

---

## 가설 (본 문서 레벨)

| ID | 가설 | 검증 방법 | 반증 시 |
|---|---|---|---|
| DOM-H1 | [H] 고객은 Substance/Grade/Lot 3층을 직관적으로 받아들인다 | MR W2~W3 인터뷰 화이트보드 세션 | 2층(Substance/Grade)으로 단순화 |
| DOM-H2 | [H] 단위는 5개(wt%, phr, 부, mol%, vol%) 지원하면 90% 커버 | MR W3 인터뷰 엑셀 샘플 수집 | 단위 카탈로그 확장 |
| DOM-H3 | [H] Property 공통 타입 50~100개 기본 카탈로그가 초기에 충분하다 | MR W3 인터뷰 측정 항목 리스트 | 카탈로그 확장 또는 자유 타입 전환 |
| DOM-H4 | [H] Formulation 버전 분기 추적 (`parent_formulation_id`)은 고빈도 사용된다 | MR 인터뷰 "레시피 수정 관행" 질문 | 단순 버전 번호로 축소 |
| DOM-H5 | [H] 실측 Component (`actual_components`)는 목표 Component와 충분히 다르다 (≥5%) | MR W3 실제 엑셀 파일 관찰 | 필드 제거, Formulation만 유지 |
| DOM-H6 | [H] Lot 추적 안 하는 고객이 60% 이상 | MR Data Maturity 분류 [DEP:MR-02] | Lot 레이어를 v1에서 기본 숨김 처리 |

---

## MVP 스코프 힌트 (05에서 확정)

**v1 반드시**: Substance, Grade(optional), Formulation, ComponentRef, Experiment, Property
**v1 optional**: Lot (UI상 접기), parent_formulation_id
**v2 이후**: instrument 카탈로그, specimen 상세 모델, Property 측정 불확도 구간

---

## Out of Scope (본 문서 한정)

- 물리 스키마 (PostgreSQL DDL, 인덱스, 파티셔닝) → `07` 및 별도 ADR
- API 페이로드 스펙 (OpenAPI) → 별도 단계
- 감사/버전 로그 세부 설계 → Year 2 이후
- 권한/RBAC 모델 → `05-mvp-feature-spec.md`에서 간략히

---

## Market Research 반영 로그

| 주차 | 반영 내용 |
|---|---|
| W1 | MR-01 §단위 다양성 → ComponentRef.unit 5종 enum. MR-01 §암묵지 → notes 필드. MR-02 §Level2~3 → Lot 선택적 레이어 원칙. MR-04 §Albert Invent 유사 깊이 → Substance/Grade/Lot 3층. |
| W2 | (pending — DOM-H1, H6 검증: 인터뷰 화이트보드에서 3층 납득도) |
| W3 | (pending — DOM-H2, H3, H5: 실제 엑셀 샘플 수집) |
| W4 final | (pending) |
