# ADR-001: Descriptor 계산 — RDKit 기본 + Mordred 선택 보강

- **상태**: Proposed (W1 초안, MR 종료 후 Accept 대상)
- **작성일**: 2026-04-23
- **결정자 (예정)**: 엔지니어링 리드, 창업자
- **관련 문서**: `07-tech-architecture-sketch.md §축4`, `03-domain-model.md §Substance.descriptor_vector`

---

## Context

MVP의 킬러 워크플로우(유사 배합 검색 + 물성 예측)는 Substance의 분자 descriptor에 의존한다. Descriptor 라이브러리 선택은 다음에 영향을 준다:

- 계산 시간 / 배치 비용
- descriptor 품질 (유사도·예측 정확도)
- 배포 복잡도 (JVM 포함 여부 등)
- 라이선스 (BIOVIA 리셀러 계약 제약 [DEP:MR-01 §부록B])

## 후보

| 후보 | 언어 | descriptor 수 | 속도 | 라이선스 | 의존성 |
|---|---|---|---|---|---|
| **RDKit** | C++/Python | ~200 | 빠름 | BSD-3 | 없음 |
| **Mordred** | Python (RDKit 위) | 1826 | 느림 | BSD-3 | RDKit 필요 |
| **OpenBabel** | C++/Python | ~50 | 중간 | GPL-2 | 별도 빌드 |
| **CDK** | Java | ~300 | 중간 | LGPL | JVM |
| **Dragon/MOE 등 상용** | 다양 | 많음 | - | 유료 | - |

## 결정 (제안)

**RDKit을 기본 엔진으로 채택하고, descriptor 가 부족하다고 판명되는 특정 property 예측 태스크에서만 Mordred를 선택적 보강으로 추가한다.**

구현:
- `Substance.descriptor_vector` 1차 채움: RDKit descriptor 세트 (고정 목록 ~150개)
- Mordred는 비동기 배치 경로로만, 플래그로 활성화
- OpenBabel·CDK·상용은 제외

## 근거

1. **RDKit은 과학 파이썬 생태계 표준**. 학습 자료, Stack Overflow, 논문 구현 모두 RDKit 중심 → 채용·유지보수 이득
2. **Python 단일 런타임 유지** (FastAPI와 동일 프로세스 또는 워커). JVM 도입(CDK) 회피
3. **라이선스 깨끗함 (BSD-3)**. BIOVIA 리셀러 계약 관점에서 **독립 오픈소스 의존**으로 명확히 주장 가능 [DEP:MR-01 §부록B]
4. **Mordred는 RDKit 위에서 동작**하므로 기본/보강 관계가 자연스러움. 필요할 때만 켠다
5. **계산 시간**: Mordred 1826 descriptor는 Substance 1개당 수백 ms~수 초. 10,000 Substance 배치 시 몇 시간. RDKit 고정 세트는 훨씬 빠름 → v1에 적합 `[H]` (POC 필요)

## 거절 근거 (반대 옵션)

- **OpenBabel**: descriptor 부족 + GPL-2 의존성이 배포 시 법적 리뷰 부담
- **CDK**: JVM 추가로 배포/운영 복잡도 상승. 인력 채용에도 불리
- **상용 (Dragon 등)**: BIOVIA 계약상 제3자 상용 라이브러리 내장이 제약받을 가능성 + 고객에게 가격 전가 부담
- **Mordred 단독**: 속도 느림 + 결측(NaN) 많아 실제 feature로 쓰기 전처리 부담 대

## 검증 (POC, W2)

- [ ] RDKit 150 descriptor로 **공공 polymer property dataset** (Matminer/MaterialsProject)에서 LightGBM R² ≥ 0.6 재현
- [ ] 1,000 Substance descriptor 계산 시간 < 5분 (로컬)
- [ ] 특정 property에서 RDKit 세트가 부족함이 드러나면 Mordred 추가 효과 측정 (R² 향상 vs 계산 시간 페널티)

## 결과

(Accept 시 여기에 실제 POC 수치와 확정 사항을 기록. MR 종료 후 업데이트.)

## 가설 태그

- `[H]` RDKit 150 descriptor로 MVP 4개 타겟 물성 예측에 충분하다
- `[H]` Mordred는 v1에서 **off** 상태 유지해도 고객 만족도 영향 없다
- `[H]` BIOVIA 리셀러 계약상 RDKit 사용에 제한 없다 (W4까지 법무 확정)
