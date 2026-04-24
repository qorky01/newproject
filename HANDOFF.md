# HANDOFF — 세션 인수인계 문서

> **목적**: 새 채팅 세션에서 즉시 맥락을 복원하고 작업을 이어가기 위한 종합 인수인계 문서
> **작성일**: 2026-04-24
> **작성자**: Claude Code 세션 (이전 세션 최종 산출물)
> **대상**: 다음 Claude Code 세션 + 사용자 본인

---

## 1. 프로젝트 한눈에 보기

### 사용자 배경
- Dassault Systèmes **BIOVIA** 컨설턴트로 일해온 경력 보유
- BIOVIA 컨설팅 사업과 **병행**하여 자체 SaaS 제품 창업 중
- 제품: **배합 R&D + AI 예측 플랫폼**
- 타깃: **한국 중견 소재/화학/화장품 R&D팀** (Level 2~3 데이터 성숙도, 연구팀장 전결)
- 가격 목표: **연 1,500만~4,500만원** (경쟁사 대비 10배 저가 포지션)

### 포지셔닝 한 줄
> "엑셀을 대체하지 않고, 엑셀 데이터를 AI가 쓸 수 있는 배합 지식 자산으로 정제해 주는 한국어 SaaS"

### 현재 단계
- **시장조사(MR) 6개 문서 완료** (`market-research/`)
- **고객 인터뷰 30건 W1~W4 진행 예정**
- **Product-spec W1 초안 완료** (`product-spec/`, 이번 세션 산출)
- 다음: W2 작업 — `04-killer-workflow` 신규 작성 + `02-persona` 가설 초안

---

## 2. 리포 구조 현재 상태

```
/home/user/newproject/
├── CLAUDE.md                          # 프로젝트 작업 규칙 (파일 수정/삭제 확인)
├── HANDOFF.md                         # ← 본 문서
├── market-research/                   # 시장조사 (완료, v1.0)
│   ├── 00-market-research-plan.md     # 4주 MR 일정, Go/No-Go 게이트
│   ├── 01-why-customers-cannot-leave-excel.md  # 엑셀 12강점 + ELN/LIMS 20년 실패사 + 7원칙
│   ├── 02-data-maturity-assessment.md # 5단계 성숙도 모델 + 20문항 체크리스트
│   ├── 03-customer-interview-script.md # Mom Test 기반 60분 인터뷰 스크립트
│   ├── 04-competitor-analysis.md      # 경쟁사 15개+ 딥다이브, 포지셔닝 맵, 차별화 축 5개
│   └── 05-target-customer-list.md     # 6개 산업 한국 중견 TOP 15+ 후보
└── product-spec/                      # 제품 스펙 (W1 v0.1 초안)
    ├── 00-spec-index.md               # 문서 맵, 용어집, Hypothesis 태깅 컨벤션
    ├── 01-product-vision.md           # 포지셔닝, 북극성, 3년 서사, 가설 8개
    ├── 03-domain-model.md             # 엔터티 6개 ERD, 가설 6개
    ├── 07-tech-architecture-sketch.md # 9개 축 후보 비교, 가설 4개
    ├── _assets/                       # (다이어그램·이미지 둘 곳, 현재 비어있음)
    ├── _decisions/
    │   └── adr-001-rdkit-vs-mordred.md  # Descriptor 라이브러리 ADR (Proposed)
    └── _changelog.md                  # 주차별 변경 로그
```

**Git 상태** (최종 commit):
```
7f0a2f7 Add W1 product-spec scaffold (index, vision, domain model, tech sketch, ADR-001)
73eec6f Add 4-week market research project plan
7c653b1 Add target customer long list
51c8738 Add competitor deep-dive matrix
8af5ff4 Add customer interview script v1
8dd17cf Add data maturity assessment sheet
```
- 브랜치: `claude/biovia-consultant-tools-sCTnt`
- remote 푸시 완료

---

## 3. 4주 병행 계획 (MR + Product-Spec)

| 주차 | 시장조사 (MR) | Product-Spec (본 세트) |
|---|---|---|
| **W1** | 인터뷰 대상 확정, 스크립트 리허설 | ✅ 00/01/03/07 + ADR-001 (완료) |
| **W2** | 인터뷰 10건 완료, pain 수렴 점검 | `04-killer-workflow` 신규, `02-persona` 초안 (가설) |
| **W3** | 인터뷰 25건 누적, 가격 검증 | `05-mvp-feature-spec` 초안, `06-pricing` 초안 |
| **W4** | `findings-final.md` 작성, Spec 영향도 매트릭스 | 02/05/06 v1.0 확정, 태그 리콘사일 세션 (H→V/X/OPEN 강제 전환) |

**W2 게이트**: 10건 인터뷰 후 pain 수렴 안 되면 → `04-killer-workflow` 작성 중단, MR 보강 우선.
**W4 게이트**: 태그 리콘사일 필수. 모든 `[H]`를 V/X/OPEN 중 하나로 강제 전환.

---

## 4. Hypothesis 태깅 컨벤션 (본 프로젝트 핵심 디자인)

모든 단정문은 인라인 태그로 검증 상태 표시:

| 태그 | 의미 | 예시 |
|---|---|---|
| `[H]` | Hypothesis — 미검증 | `[H] R&D팀장은 3,000만원 전결 가능하다` |
| `[H→V w3-i12]` | Validated — 주차-인터뷰번호 인용 | `[H→V w3-i12] 퇴사 시 Excel 유실은 자발 언급 pain이다` |
| `[H→X w3-i08]` | Invalidated — 취소선 유지 | `~~[H→X w3-i08] CPG는 가격 민감도 낮다~~` |
| `[DEP:MR-04]` | MR 문서 참조 | `가격대 1,500만~4,500만 [DEP:MR-04]` |
| `⟨confidence: low/med/high⟩` | 섹션 신뢰도 | `## MVP 핵심 기능 ⟨confidence: med⟩` |

각 문서 상단 카운터: `H: 12 · V: 3 · X: 1 · OPEN: 8`

**인터뷰 번호 컨벤션**: `w{주차}-i{01..30}` (예: `w3-i12` = W3 12번째 인터뷰)

---

## 5. 현재 가설 인벤토리 (총 18개, 전부 OPEN)

### 01-product-vision.md (8개)
- **VIS-H1** 한국 중견 R&D에서 "배합 지식 자산화"는 자발 언급 pain이다
- **VIS-H2** "AI 예측"은 구매 트리거로 작동한다
- **VIS-H3** 1,500만~4,500만 구간은 R&D팀장 전결 범위다
- **VIS-H4** 한국어 UI는 구매 결정 요인 Top 5 안이다
- **VIS-H5** Notion+Excel+Python에서 이탈 의향 조직이 충분 존재
- **VIS-H6** 화장품/소재/2차전지 중 Year 1 집중할 1개 세그먼트가 명확히 뜨겁다
- **VIS-H7** Albert Invent를 써본 한국 고객은 "비싸서 못 산다"고 말한다
- **VIS-H8** 퇴사 리스크를 예산 집행 명분으로 쓴다

### 03-domain-model.md (6개)
- **DOM-H1** 고객은 Substance/Grade/Lot 3층을 직관적으로 받아들인다
- **DOM-H2** 단위 5종(wt%/phr/부/mol%/vol%)이면 90% 커버
- **DOM-H3** Property 공통 타입 50~100개 카탈로그가 충분
- **DOM-H4** Formulation 버전 분기 추적은 고빈도 사용
- **DOM-H5** 실측 Component와 목표 Component 차이 ≥5%
- **DOM-H6** Lot 추적 안 하는 고객이 60% 이상

### 07-tech-architecture-sketch.md (4개)
- **TEC-H1** pgvector v1 규모에서 응답 300ms 이내
- **TEC-H2** Excel-like 셀 편집은 ag-grid/handsontable 하나로 충분
- **TEC-H3** 한국 중견 IT는 AWS 서울 리전 SaaS 수용
- **TEC-H4** RDKit 기본 descriptor로 물성 예측 유의

---

## 6. 다음 세션에서 해야 할 일 (우선순위)

### 즉시 착수 가능 (신규 파일, 승인 불필요)

1. **`product-spec/04-killer-workflow.md` 신규 작성** — 9단계 스토리보드
   - "엑셀 업로드 → 파싱 → Substance 매핑 → descriptor 계산 → 유사 배합 검색 → 예측 → 리포트 내보내기"
   - MR-01 §Part4 Win 2,3,6 반영
   - 각 단계별 수락 기준 + 가설 태그

2. **`product-spec/02-persona-and-jtbd.md` 초안(가설 모드)** — W2~W4에 인터뷰로 보강
   - 페르소나 3개 후보: 화장품 R&D팀장 / 소재 기업 연구소장 / 2차전지 소재팀장
   - JTBD 3~5개 초안
   - 구매 트리거·예산·저항 요소

### MR 진척 후 업데이트 (기 커밋 파일 수정 — **사전 승인 필요**)

3. W2 인터뷰 10건 완료 시 → 01/03/07의 `[H]` 태그를 `[H→V w2-iNN]`으로 전환
4. `_changelog.md` W2 엔트리 추가
5. MR-04 §차별화축이 인터뷰로 뒤집히면 01의 매핑 테이블 갱신

### W3~W4 산출 (미작성)

6. `05-mvp-feature-spec.md` (W3 초안) — MVP 기능 ≤5개
7. `06-pricing-and-packaging.md` (W3 초안)
8. `_decisions/adr-002-pgvector-vs-qdrant.md` (POC 후)

---

## 7. 작업 규칙 요약 (`CLAUDE.md` 준수)

- **신규 파일 생성**: 승인 불필요
- **기 커밋 파일 수정**: 어느 파일의 어느 부분을 어떻게 바꿀지 **설명 후 승인**
- **기 커밋 파일 삭제**: 반드시 사전 확인
- 예외: 사용자가 "바로 진행해줘" 명시 시 확인 생략
- 브랜치: `claude/biovia-consultant-tools-sCTnt` (이 브랜치에서만 개발 + 푸시)
- 언어: 모든 문서는 **한국어**. 기술 용어/코드/고유명사만 영어

---

## 8. 중요 참조 (새 세션에서 반드시 읽을 것)

우선순위 순:
1. `HANDOFF.md` (본 문서)
2. `CLAUDE.md`
3. `product-spec/00-spec-index.md` (문서 맵 + 태깅 컨벤션)
4. `market-research/00-market-research-plan.md` (MR 일정, Go/No-Go 게이트)
5. `market-research/01-why-customers-cannot-leave-excel.md` (7가지 제품 원칙 근거)
6. `market-research/04-competitor-analysis.md` (차별화 축 5개)
7. `product-spec/01-product-vision.md` · `03-domain-model.md` · `07-tech-architecture-sketch.md` (W1 산출)

---

## 9. 의도적으로 결정하지 않은 것 (Open Decisions)

- 페르소나 1개 확정 (Year 1 집중 세그먼트) → W4 이후
- MVP 기능 5개 확정 → W3~W4
- 가격 티어 구조 → W3~W4
- 배포 리전 (AWS vs GCP 서울) → TEC-H3 검증 후
- Web/Backend 프레임워크 최종 확정 → MVP 킥오프 직전
- 인터뷰 30건 대상 최종 리스트 (`market-research/05`에 후보 TOP 15+만 있음)

---

## 10. 의도적으로 하지 않는 것 (Out of Scope)

- 코드/프로토타입 (별도 phase)
- Figma/와이어프레임 (텍스트 스토리보드만)
- 상세 SQL DDL, OpenAPI 스펙
- 법무 계약서 드래프트 (BIOVIA 리셀러 제약 법무 검토는 W4까지 별도 진행)
- 조직/채용 계획
- 투자 IR 덱 (7개 문서가 원천이 됨)
- GMP/FDA 21 CFR Part 11 규제 대응 (Year 3 이후)
- DOE 추천, 기기 직접 연동, 다국어 (Year 2 이후)
