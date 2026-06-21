## 📅 Day 23: Search and Query MLflow Runs

### 1. Task

- **요구사항**: 대규모 실험(Runs)이 누적된 특정 익스페리먼트 내에서 비즈니스 지표(Metrics) 기준에 따라 배포 후보군 모델과 폐기 대상 모델을 선별하고, 시스템 태깅을 통해 모델 관리 파이프라인의 가시성을 확보합니다.

- **목표**:
  - `fraud-detection` 내 10개의 가동 실행 이력 검토
  - 최상위 모델(`f1_score > 0.85` 중 최고점) 식별 및 `review-status: shortlisted` 마킹
  - 저성능 모델(`f1_score < 0.75`) 일괄 식별 및 `review-status: rejected` 마킹
  - 경계선 성능 및 차순위 모델의 메타데이터 격리 및 무태그 상태 유지

---

### 2. Workflow

```text
       [ 10 Pre-populated Runs in fraud-detection ]
                            |
                            v (UI Filter & f1_score Sorting)
       +--------------------+--------------------+
       |                    |                    |
       v (f1_score = 0.95)  v (0.75 <= f1 <= 0.85)v (f1_score < 0.75)
[ Top 1 Candidate ]    [ Buffer/Boundary ]   [ Under-performers ]
       |                    |                    |
       v [Add Tag]          v [Keep Neutral]     v [Add Tag]
 review-status:        No review-status      review-status:
  shortlisted               tag                rejected

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. MLflow UI 정렬 및 대상 타겟팅

1. `fraud-detection` 익스페리먼트 메인 대시보드에서 `f1_score` 컬럼 헤더를 클릭하여 내림차순 정렬을 수행했습니다.
2. 결과 세트에서 최상단 팩트 매핑을 확인했습니다.

- `likeable-crow-169` (f1_score: 0.95) -> **Shortlist 대상**

#### 3-2. 조건별 런레벨 태그 바인딩 (Triage 수행)

1. `likeable-crow-169` 상세 뷰 진입 후 하단 **Tags** 패널에서 `review-status` : `shortlisted`를 수동 주입했습니다.
2. 메인 화면으로 복귀하여 `metrics.f1_score < 0.75` 쿼리를 필터링 창에 적용하거나 정렬 하단부를 탐색하여 대상을 도출했습니다.

- `abrasive-slug-765` (f1_score: 0.72) -> **Reject 대상**
- `indecisive-panda-768` (f1_score: 0.70) -> **Reject 대상**

3. 추출된 2개의 실행 엔티티에 진입하여 각각 `review-status` : `rejected` 메타데이터를 매핑했습니다.

---

### 4. 핵심 개념 정리

- **Model Triage**: 훈련된 수많은 머신러닝 모델 중 비즈니스 요구사항 및 평가지표를 통과한 최적의 아티팩트를 선별하고 시스템 상태를 업데이트하는 엔지니어링 프로세스입니다.

💡 **쉬운 비유**: **'오디션 심사 후 합격/불합격 스티커 붙이기'**. 10명의 참가자(Run) 중 가장 노래를 잘 부른 1등에게는 '본선 진출' 티켓(`shortlisted`)을 붙여주고, 기준 미달인 참가자들에게는 '탈락' 스티커(`rejected`)를 붙여 분류해 두는 것과 같습니다. 점수가 애매한 중간 참가자들은 다음 심사를 위해 스티커를 붙이지 않고 그대로 대기시킵니다.

- **실무 관점의 의의**: 실제 엔터프라이즈 환경에서 파이프라인이 자동 재학습(Retraining)을 수행하면 매일 수백 개의 모델 실행 레코드가 쌓입니다. 엔지니어는 이러한 UI 기반 필터링 및 태깅 규칙을 이해하고 있어야 향후 CD(지속적 배포) 파이프라인에서 `review-status == 'shortlisted'` 조건의 모델만 스테이징 레지스트리로 승격(Promote)시키는 웹훅(Webhook) 및 자동화 코드를 설계할 수 있습니다.

---

### 5. 무엇을 배웠는가

- **데이터 거버넌스와 메타데이터 구조화**: 인프라 엔지니어링 관점에서 메타데이터 태그는 단순한 텍스트 가독성 확보용이 아니라, 하위 배포 파이프라인의 트리거 조건으로 동작하는 구조적 제어 플래그(Control Flag)임을 인지했습니다.

- **경계 조건(Boundary condition) 관리 역량**: 조건 명세서에 명시되지 않은 중간 상태(`0.75 ≤ f1_score ≤ 0.85`)의 데이터에 임의 변경을 가하지 않고 중립 상태(Neutral) 레이어로 온전하게 격리 유지하는 예외 처리 사고방식을 학습했습니다.
