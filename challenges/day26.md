## 📅 Day 26: Compare Model Runs and Select the Best

### 1. Task

- **요구사항**: 동일한 데이터셋 및 피처 엔지니어링 조건에서 개별 알고리즘 스펙(RandomForest, GradientBoosting, LogisticRegression)에 의해 독립적으로 학습된 후보 모델군을 단일 대시보드 레이어에서 비교 분석하고, 프로덕션 배포 파이프라인으로 이관할 최적의 후보 모델을 선별하여 플래그 태그를 바인딩합니다.

- **목표**:
  1. `model-comparison` 익스페리먼트에 pre-populated된 3개의 가동 실행 이력 검토
  2. MLflow UI 'Compare' 뷰포트를 활성화하여 `metrics.f1_score` 기준 다각적 대조 수행
  3. 최고 성능 모델(GradientBoosting, f1_score=0.91)에 `production-candidate: true` 태그 마킹
  4. 탈락한 나머지 차순위 실행 모델들의 메타데이터 격리 및 무태그 상태 유지

---

### 2. Workflow

```text
             [ 3 Pre-populated Runs in model-comparison ]
         (RandomForest, GradientBoosting, LogisticRegression)
                            |
                            v (UI Multi-Select -> [Compare] Action)
       +--------------------+--------------------+
       |                    |                    |
       v (f1_score = 0.85)  v (f1_score = 0.91)  v (f1_score = 0.78)
 [ RandomForest ]      [ GradientBoosting ]   [ LogisticRegression ]
       |                    |                    |
       v [Keep Neutral]     v [Add Tag]          v [Keep Neutral]
  No candidate        production-candidate:       No candidate
      tag                    true                     tag

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 다중 런 선택 및 교차 비교 인터페이스 진입

1. MLflow UI 메인 대시보드에서 `model-comparison` 익스페리먼트 공간으로 이동했습니다.
2. 실행 테이블 리스트에 노출된 `RandomForest`, `GradientBoosting`, `LogisticRegression` 총 3개의 실행 엔티티 좌측 체크박스를 일괄 선택했습니다.
3. 테이블 제어 툴바 상단에 활성화된 **[Compare]** 버튼을 트리거하여 다중 모델 파라미터-메트릭 대조 뷰포트로 진입했습니다.

#### 3-2. 매트릭스 비교분석 및 우승 모델 도출

1. 비교 레이아웃 내 **Metrics** 패널의 테이블 사양을 분석하여 타겟 평가지표인 `f1_score` 컬럼을 격리 검증했습니다.

- `RandomForest` 실행군 평가지표 ──> `f1_score`: 0.85
- `GradientBoosting` 실행군 평가지표 ──> `f1_score`: 0.91 (**최상위 우승 모델 식별**)
- `LogisticRegression` 실행군 평가지표 ──> `f1_score`: 0.78

#### 3-3. 다운스트림 트리거용 시스템 태그 바인딩

1. Compare 매트릭스 뷰포트에서 최고 성능이 실증된 `GradientBoosting` 모델의 실행 이름 링크를 클릭하여 개별 런(Run) 상세 관리 페이지로 이동했습니다.
2. 하단 **Tags** 패널의 **[Add Tag]** 인터페이스를 활성화하고 배포 자동화 엔진이 감지할 수 있도록 아래 메타데이터 규격을 정확히 주입했습니다.

- **Key**: `production-candidate`
- **Value**: `true`

3. 메인 익스페리먼트 리스트 뷰로 복귀하여 오직 우승 모델에만 해당 플래그 레이어가 바인딩되었고, 나머지 2개 모델은 중립 상태로 격리되었음을 크로스체크했습니다.

---

### 4. 핵심 개념 정리

#### 4-1. MLflow Compare Runs (실험 대조 뷰)

- **전문적 정의**: 단일 혹은 다중 익스페리먼트 내에서 수집된 여러 실행(Runs) 파라미터의 변동에 따른 메트릭 성능 차이를 하이라이팅, 스캐터 플롯, 병렬 좌표계 그래프 등으로 시각화하여 나란히(Side-by-Side) 대조할 수 있도록 돕는 다각적 분석 컴포넌트입니다.
  💡 **비유하자면**, **'신차 출시 전 여러 프로토타입의 스펙 비교표'**. 엔진 배기량, 타이어 종류(하이퍼파라미터)를 다르게 해서 만든 차 3대를 테스트 트랙에 동시에 올려두고, 최고 속도와 연비(메트릭)가 적힌 스펙 비교표를 모니터 하나에 나란히 띄워 어떤 차를 최종 출시할지 결정하는 심사대와 같습니다.

#### 4-2. Downstream Triggering (시스템 태깅을 통한 다운스트림 파이프라인 트리거)

- **전문적 정의**: 특정 기준을 충족한 모델 실행에 제어 플래그용 태그(Control Tag)를 마킹하여, UI 외부의 배치 스크립트나 CI/CD 오케스트레이터가 해당 메타데이터를 쿼리(Query)해 자동으로 모델 레지스트리 승격 및 컨테이너 빌드를 트리거하도록 설계하는 인프라 패턴입니다.
  💡 **비유하자면**, **'수많은 서류 중 통과된 기안서에만 빨간색 [승인] 도장을 찍는 것'**. 결재 대기 중인 수많은 서류함에서 대리인(엔지니어)이 1등 기안서에 빨간 도장(`production-candidate: true`)을 찍어두면, 다음 날 출근한 부장님(자동화 배포 스크립트)이 다른 서류는 보지도 않고 빨간 도장이 찍힌 서류만 골라내어 곧바로 집행(배포)하는 연계 메커니즘입니다.

#### 4-3. 실무 관점의 의의 (실험 거버넌스)

- 실제 프로덕션 MLOps 인프라 환경에서 모델 튜닝 단계(Hyperparameter Sweep)를 거치면 수십, 수백 개의 런(Runs)이 일시에 생성됩니다. 엔지니어가 이 수많은 모델 중 어떤 것이 배포 대상인지 수동으로 추적하는 것은 불가능에 가깝습니다.
- 본 실습과 같이 MLflow Compare 레이어를 통해 최고점을 낸 단 하나의 실행 모델을 엄격하게 식별하고, 명확한 태깅 규칙(Naming Convention)에 따라 `production-candidate: true`를 마킹하는 프로세스는 휴먼 에러를 완벽히 통제합니다. 이를 통해 상위 배포 파이프라인이 주기적으로 MLflow API를 호출하여 "해당 태그를 가진 최신 모델을 가져와라"라는 식의 무중단 자동화 아키텍처를 견고하게 지탱할 수 있습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **다각적 지표 비교를 통한 RCA 역량**: 단일 실행의 로그만 독립적으로 검토하는 방식에서 벗어나, 여러 모델 구조의 평가지표 매트릭스를 단일 뷰포트에 병렬 적재하여 상대적 성능 우위를 정밀 타겟팅하는 구조적 분석 기법을 훈련했습니다.

- **인프라 간 결합도 완화 및 제어 플래그 활용**: 하위 배포 엔진에 "GradientBoosting 모델을 배포하라"고 모델 알고리즘명을 하드코딩하는 대신, `production-candidate`라는 범용적인 추상화 태그를 매개체로 삼아 인프라 레이어 간의 결합도를 완화(Loose Coupling)하는 엔지니어링 설계 안목을 얻었습니다.
