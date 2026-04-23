# 00 · Spec Index

> **버전**: v0.1 (W1 초안) · **작성일**: 2026-04-23
> **카운터**: H: 0 · V: 0 · X: 0 · OPEN: 0 (W1 시점)

---

## 목적

본 문서는 `product-spec/` 세트의 **진입점**이다. 엔지니어·디자이너·투자자가 처음 들어왔을 때 30분 안에 전체 지형을 파악하도록 한다.

**전제**: 시장조사(`market-research/`, 6개 문서 완료)는 병행 중이며, 인터뷰 30건이 W1~W4에 걸쳐 진행된다. 본 spec 세트는 MR 결과를 기다리지 않고 **가설 기반**으로 먼저 작성되며, 인터뷰 발견이 누적되면 태그 전환으로 수렴된다.

---

## 문서 맵

| # | 파일 | 한 줄 설명 | MR 민감도 | 타깃 완성 주차 |
|---|---|---|:---:|:---:|
| 00 | `00-spec-index.md` | 본 문서 (맵 · 용어집 · 태깅 컨벤션) | Low | W1 |
| 01 | `01-product-vision.md` | 포지셔닝, 북극성, 3년 서사, Out of scope | Low | W1 |
| 02 | `02-persona-and-jtbd.md` | R&D팀장 페르소나 · JTBD 3~5개 · 구매 트리거 | **High** | W2 초안 → W4 확정 |
| 03 | `03-domain-model.md` | Substance/Grade/Lot · Formulation · Experiment · Property ER | Medium | W1 |
| 04 | `04-killer-workflow.md` | 9단계 스토리보드 (Excel → descriptor → 유사 배합 → 예측) | Medium | W2 |
| 05 | `05-mvp-feature-spec.md` | MVP 기능 ≤5개 · In/Out · 수락 기준 | **High** | W3 초안 → W4 확정 |
| 06 | `06-pricing-and-packaging.md` | 티어 · 가격대 검증 로직 · 계약 구조 | **High** | W3 초안 → W4 확정 |
| 07 | `07-tech-architecture-sketch.md` | 후보 비교 (FastAPI vs Django, RDKit vs Mordred 등) · ADR | Low | W1 |

보조:
- `_assets/` — 다이어그램 · 스크린샷
- `_decisions/` — ADR (Architecture Decision Record) 모음
- `_changelog.md` — 주차별 변경 로그 · H 카운터 추이

---

## Hypothesis 태깅 컨벤션

모든 단정문은 **검증 상태**를 인라인 태그로 표시한다. 이 컨벤션이 본 세트의 핵심 디자인이다.

### 태그 종류

| 태그 | 의미 | 예시 |
|---|---|---|
| `[H]` | Hypothesis — 미검증 | `[H] R&D팀장은 3,000만원 단독 전결 가능하다` |
| `[H→V w3-i12]` | Validated — 주차-인터뷰번호로 근거 인용 | `[H→V w3-i12] 연구원 퇴사 시 Excel 데이터 유실은 자발 언급 pain이다` |
| `[H→X w3-i08]` | Invalidated — 반증. 문장은 취소선으로 유지 | `~~[H→X w3-i08] CPG는 가격 민감도가 낮다~~` |
| `[DEP:MR-04]` | Dependency — MR 문서 참조 | `가격대 1,500만~4,500만원 [DEP:MR-04]` |
| `⟨confidence: low/med/high⟩` | 섹션 헤더 옆 신뢰도 | `## MVP 핵심 기능 ⟨confidence: med⟩` |

### 문서 상단 카운터

각 문서는 첫 줄에 Hypothesis 카운터를 갖는다:
```
카운터: H: 12 · V: 3 · X: 1 · OPEN: 8
```
- `H` = 총 가설 수
- `V` = 검증됨 (H→V)
- `X` = 반증됨 (H→X)
- `OPEN` = 아직 `[H]` 상태 (= H − V − X)

### W4 리콘사일 규칙

W4 종료 시점에 모든 `[H]`는 V / X / OPEN 중 하나로 **강제 전환**한다. `OPEN` 항목은 별도 백로그(`_decisions/open-questions.md`)로 이관해 MVP 개발 중 추가 검증 대상으로 남긴다.

---

## MR ↔ Product-Spec 통합

- 각 spec 문서 **하단 고정 섹션** "Market Research 반영 로그" 유지 (`w2 / w3 / w4-final` 엔트리)
- MR `findings-final.md`(W4 산출)에 **"Spec 영향도 매트릭스"** 표 추가 예정: 발견 ↔ spec 태그 뒤집기 매핑
- MR 인터뷰 번호 컨벤션: `w{주차}-i{01..30}` (예: `w3-i12` = W3에 진행한 12번째 인터뷰)

---

## 용어집 (Glossary)

도메인 용어 — 문서 전반에서 동일하게 사용한다.

### 도메인 (화학/배합)

| 용어 | 정의 |
|---|---|
| **Substance** | 논리적 화학물질/원료. 예: "TiO2", "계면활성제 A". 공급사·Lot 독립 |
| **Grade** | Substance의 규격 변종. 예: "TiO2 rutile R-902" vs "R-706". 공급사별 상이 |
| **Lot** | Grade의 배치. 시간축 변수(생산일 · 유통기한 · COA 수치 편차) 보유 |
| **Formulation** | 배합 레시피. `{Substance or Grade, 중량 %} × N` + 가공 조건 |
| **Experiment** | Formulation × 제조/시험 조건 1회 실행. 결과 Property 세트 생성 |
| **Property** | 측정된 물성. 예: `점도 @25°C`, `인장강도`, `pH`. 단위 · 방법 · 시편 정보 포함 |
| **Descriptor** | RDKit/Mordred 등이 Substance의 SMILES에서 계산하는 분자 기술자 벡터 |

### 제품/비즈니스

| 용어 | 정의 |
|---|---|
| **ICP** | Ideal Customer Profile. 본 제품 기준: 한국 중견 소재/화학/화장품 R&D 보유, Data maturity Level 2~3 [DEP:MR-02] |
| **Killer workflow** | 고객이 "이거 없으면 일 못 한다"고 답하는 단일 워크플로우. `04-killer-workflow.md` 참조 |
| **Feature store** | 물성 예측 모델 학습용으로 정제된 피처 테이블. 원천은 Formulation × Property 조인 |
| **Data maturity** | 고객 조직의 실험 데이터 정리도. Level 0(종이) ~ Level 4(LIMS+MLOps) [DEP:MR-02] |

### 태그/카운터

| 용어 | 정의 |
|---|---|
| **Hypothesis (H)** | 미검증 단정문. MR 인터뷰로 V/X 전환 대상 |
| **Validated (V)** | 인터뷰 n건 이상에서 일관되게 확증된 가설 (n 기준은 케이스별) |
| **Invalidated (X)** | 반증 사례가 나온 가설. 취소선으로 문서 내 보존 (삭제하지 않음 — 학습 자산) |
| **ADR** | Architecture Decision Record. `_decisions/adr-NNN-*.md`에 저장 |

---

## 문서 작성 규칙

1. **한국어 원칙**: 본문은 한국어. 단, 기술 용어/고유명사/코드 식별자는 영어 유지 (예: `FastAPI`, `RDKit`, `SMILES`).
2. **단정문 ≠ 가설**: 사실(MR 인용 · 외부 레퍼런스)과 가설을 섞지 않는다. 가설은 반드시 `[H]` 태그.
3. **MR 인용 포맷**: `[DEP:MR-XX §섹션명]` 또는 `[DEP:MR-XX L123]` (line 참조).
4. **파일 수정 규칙**: `CLAUDE.md` 규칙 준수 — 이미 커밋된 파일 수정 시 사용자 사전 승인 필요.
5. **버전 마킹**: 각 문서 상단에 `버전: vX.Y (주차 단계)` 표시. 메이저 변경(가설 → 확정) 시 `v1.0`.

---

## 성공 기준 (W4 말)

- [ ] 7개 문서 v1.0 존재
- [ ] H → V 전환율 ≥ 40%
- [ ] MVP 기능 ≤ 5개로 수렴
- [ ] 페르소나 1개 확정
- [ ] 가격 범위 ±20% 내 수렴 (MR 발언 5건+ 근거)
- [ ] ADR 1~2개 작성 (`_decisions/`)
- [ ] 외부 1인 30분 리뷰 통과 ("이게 무슨 제품인지 이해했다")

---

## Market Research 반영 로그

| 주차 | 반영 내용 |
|---|---|
| W1 | MR `01, 02, 04, 05` 기반으로 초안 작성. 인터뷰 데이터 0건 — 전부 `[H]` 상태 |
| W2 | (pending) |
| W3 | (pending) |
| W4 final | (pending) |
