## 📅 Day 38: 모델 병렬 학습(Parallel-training) 파이프라인 성능 최적화 및 로깅 교정

### 1. Task

- **요구사항**: xFusionCorp Industries의 ML 플랫폼 팀은 사기 탐지 모델의 학습 속도 최적화를 위해 단일 코어(Serial)와 다중 코어(Parallel) 환경 간의 성능(Bake-off) 테스트를 수행합니다. 현재 병렬 학습 스크립트(`train_parallel.py`)가 실제로는 다중 코어를 사용하지 않고 단일 코어로만 구동되며, MLflow 추적 로그에서도 두 런(Run)의 설정값이 하드코딩되어 명확히 구분되지 않는 문제를 교정해야 합니다.

- **목표**:
  1. Scikit-learn의 `n_jobs` 파라미터를 교정하여 두 번째 훈련이 모든 가용 CPU 코어(`-1`)를 사용하도록 수정
  2. MLflow `parallel-training` 실험 내에 생성된 런(Run)에, 하드코딩된 `"all"` 대신 실제 적용된 워커 수(`1` 및 `-1`)를 명확하게 로깅
  3. 병렬 처리(`n_jobs=-1`)가 직렬 처리(`n_jobs=1`) 대비 최소 10% 이상 빠른 훈련 소요 시간(`metrics.training_time_seconds`)을 기록하도록 성능 차이 입증

---

### 2. Workflow

```text
[Training Pipeline: train_parallel.py]
  │
  ├── Iteration 1: Serial Training
  │      ├── Model: RandomForestClassifier(n_jobs=1)
  │      └── MLflow: log_param("n_jobs", 1) ──> log_metric("training_time_seconds", 0.65s)
  │
  ├── Iteration 2: Parallel Training
  │      ├── Model: RandomForestClassifier(n_jobs=-1)  <-- [Fix: 가용 CPU 코어 전체 할당]
  │      └── MLflow: log_param("n_jobs", -1) ──> log_metric("training_time_seconds", 0.47s)
  │
  ▼
[End State: MLflow UI & Artifacts]
  ├── UI Compare View: 직렬/병렬 환경 간의 명확한 학습 소요 시간(Wall-time) 차이 확인 가능
  └── models/model.pkl 최종 모델 저장 완료

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 병렬 처리 워커 수 설정 오류 식별 및 수정

스크립트의 워커 수 리스트가 단일 코어(`1`)로만 하드코딩되어 있어, 루프를 두 번 돌아도 동일한 자원만 소모하는 문제를 확인했습니다. Scikit-learn에서 지원하는 가용 코어 전체 사용 지시자인 `-1`을 할당했습니다.

```python
# src/models/train_parallel.py (Line 24 부근)

# [수정 전]: 루프의 두 번째 실행에서도 단일 코어만 사용
# N_JOBS_VALUES = [1, 1]

# [수정 후]: 두 번째 실행 시 시스템의 모든 코어를 사용하도록 변경
N_JOBS_VALUES = [1, -1]

```

#### 3-2. MLflow 파라미터 하드코딩 오류 교정

UI에서 두 개의 런(Run)을 명확히 구분할 수 있도록, `mlflow.log_param` 함수에서 무의미하게 하드코딩되어 있던 문자열을 실제 주입되는 변수(1, -1)로 변경했습니다.

```python
# src/models/train_parallel.py (Line 42 부근)

# [수정 전]: 모든 런(Run)의 n_jobs 파라미터가 "all"로 동일하게 기록됨
# mlflow.log_param("n_jobs", "all")

# [수정 후]: 실제 할당된 코어 설정값(1 또는 -1)이 기록되도록 변수 매핑
mlflow.log_param("n_jobs", n_jobs)

```

#### 3-3. 파이프라인 구동 및 속도 개선(성능 최적화) 검증

수정된 코드를 터미널에서 구동하여 직렬 훈련과 병렬 훈련의 소요 시간을 검증했습니다.

```bash
# 훈련 스크립트 실행
python3 src/models/train_parallel.py

# 터미널 출력 결과 (성능 최적화 검증)
# [serial] n_jobs=1  training_time_seconds=0.650
# [parallel] n_jobs=-1  training_time_seconds=0.470

```

단일 코어(0.65초) 대비 다중 코어(0.47초)가 최소 10% 이상(실제 약 27%) 더 빠르게 학습을 완료하여 병렬 최적화가 정상 동작함을 확인했습니다.

---

### 4. 핵심 개념 정리

- **병렬 처리(Parallel Processing)와 `n_jobs**`: 컴퓨팅 자원(CPU 코어)을 여러 개 동시에 사용하여, 하나의 큰 작업을 여러 조각으로 나누어 동시에 처리하는 기법입니다. Scikit-learn에서는 `n_jobs=-1`로 설정하면 현재 시스템이 가진 모든 코어를 100% 동원하여 학습을 수행합니다.

> 💡 비유하자면, '1,000장의 아파트 분양 전단지 돌리기' 작업과 같습니다. 혼자서(단일 코어, `n_jobs=1`) 1층부터 100층까지 돌리면 10시간이 걸리지만, 친구 10명(다중 코어, `n_jobs=-1`)을 모아서 10개 층씩 나누어 동시에 돌리면 단 1시간 만에 끝낼 수 있는 것과 완벽히 같은 원리입니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 스크립트 실행 버튼만 누르면 '알아서 컴퓨터가 좋은 사양을 다 끌어다 쓰겠지'라고 막연하게 생각했던 것이 얼마나 위험한 착각인지 눈으로 확인했습니다. 코드에 명시적으로 자원 활용을 지시(`n_jobs=-1`)하지 않으면, 64코어짜리 값비싼 클라우드 인스턴스를 빌려놓고도 코어 1개만 혹사시키며 "왜 이렇게 학습이 느리지?" 하고 기다리는 바보 같은 상황이 연출됩니다.

- **Pain Point**: 실무에서 수십 기가바이트(GB) 단위의 대용량 데이터를 다룰 때, 단일 코어와 다중 코어의 학습 시간 차이는 몇 초가 아니라 '몇 시간' 또는 '며칠'의 단위로 벌어집니다. 파라미터가 `"all"`처럼 하드코딩되어 MLflow에 잘못 기록되면, 비싼 서버 비용을 결제한 관리자 입장에서는 어떤 모델이 최적화된 리소스를 썼는지 알 길이 없고 인프라 성능 보고서는 신뢰를 잃게 됩니다.

- **성장 포인트**: 컴퓨팅 리소스를 코드로 직접 제어하여 인프라의 잠재력을 100% 끌어내는 방법을 배웠고, 이를 '소요 시간(Wall-time)'이라는 명확한 메트릭으로 중앙 대시보드에 증명하는 파이프라인을 경험했습니다. 앞으로 모델의 규모가 커지거나 데이터가 늘어나 학습 속도에 병목이 생겼을 때, 무작정 "비싼 서버로 스케일업(Scale-up) 해주세요"라고 요구하기 전에 "알고리즘 코드가 현재 서버의 코어를 100% 병렬로 활용하고 있는가?"를 가장 먼저 점검하고 튜닝할 수 있는 엔지니어의 시야를 갖추게 되었습니다.
