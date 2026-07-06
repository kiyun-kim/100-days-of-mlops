## 📅 Day 36: Automated Model Selection with FLAML AutoML

### 1. Task

- **요구사항**: xFusionCorp Industries의 ML 플랫폼 팀은 사기 탐지 모델의 최종 배포를 위해 RandomForest, GradientBoosting, LogisticRegression 세 가지 알고리즘을 경쟁(Bake-off)시키고 있습니다. 각 모델의 학습과 MLflow 로깅은 정상적으로 수행되나, 이 기록들을 조회하여 최종 승자를 선정하는 오케스트레이터 스크립트(`bakeoff.py`)에 정렬 기준 오류와 보고서 규격 누락이 존재하여 이를 교정해야 합니다.

- **목표**:
  1. `bakeoff.py` 내 MLflow 런(Run) 조회 로직의 정렬 기준을 교정하여 가장 높은 F1 점수(Highest F1-Score)를 기록한 모델이 승자로 선정되도록 수정
  2. 최종 산출물인 `/root/code/fraud-detection/reports/winner.json` 파일에 `model_type`, `run_id`, `f1_score` 3가지 키가 정확히 포함되도록 스키마 교정

---

### 2. Workflow

```text
[Candidate Trainers]
  ├── train_rf.py ──> (RF 모델 학습) ──> [MLflow Log: tags.candidate='random_forest', f1_score=0.48]
  ├── train_gb.py ──> (GB 모델 학습) ──> [MLflow Log: tags.candidate='gradient_boosting', f1_score=0.50]
  └── train_lr.py ──> (LR 모델 학습) ──> [MLflow Log: tags.candidate='logistic_regression', f1_score=0.51]
                                          │
[Orchestrator: bakeoff.py] <──────────────┘
  ├── 1. mlflow.search_runs() 호출 (f1_score 기준 DESC 정렬)
  ├── 2. 최상단(Index 0) 런 데이터 추출
  └── 3. 'tags.candidate', 'run_id', 'metrics.f1_score' 매핑
       │
       ▼
[End State: Deployment Report]
  └── reports/winner.json (최종 승리 모델 메타데이터 저장 완료)

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 모델 경쟁(Bake-off) 런타임 구동

우선 비교 대상이 될 3개의 후보 모델 훈련 스크립트를 각각 실행하여 MLflow의 `bakeoff` 실험(Experiment) 내에 평가 데이터를 적재했습니다.

```bash
# 3개 후보 모델 순차 학습 및 로깅
python3 src/models/train_rf.py
python3 src/models/train_gb.py
python3 src/models/train_lr.py

```

#### 3-2. 승자 선정 정렬 로직(`order_by`) 교정

`src/models/bakeoff.py` 파일에서 MLflow의 추적 기록을 조회할 때, 가장 낮은 점수의 모델을 가져오던 기존의 오름차순 정렬 로직을 내림차순으로 교정했습니다.

```python
# src/models/bakeoff.py (Line 38 부근)

# [수정 전]: 최하위 점수를 승자로 잘못 선정하는 오류
# runs = mlflow.search_runs(
#     experiment_ids=[exp.experiment_id],
#     order_by=["metrics.f1_score ASC"],
#     max_results=10,
# )

# [수정 후]: F1 스코어 내림차순 정렬로 최고 성능 모델 탐색
runs = mlflow.search_runs(
    experiment_ids=[exp.experiment_id],
    order_by=["metrics.f1_score DESC"],
    max_results=10,
)

```

#### 3-3. 보고서 딕셔너리(`report`) 누락 키 매핑

승리한 모델의 계열(알고리즘 종류)을 명시하는 `model_type` 키가 누락되어 배포 체크리스트를 통과하지 못하는 문제를 해결하기 위해, MLflow 런 데이터의 `tags.candidate` 값을 매핑하여 추가했습니다.

```python
# src/models/bakeoff.py (Line 49 부근)

# [수정 전]: model_type 정보가 누락된 불완전한 스키마
# report = {
#     "run_id": winner["run_id"],
#     "f1_score": float(winner["metrics.f1_score"]),
# }

# [수정 후]: 모델 종류, 런 ID, 점수가 모두 포함된 완벽한 스키마
report = {
    "model_type": winner["tags.candidate"],
    "run_id": winner["run_id"],
    "f1_score": float(winner["metrics.f1_score"]),
}

```

#### 3-4. 최종 실행 및 결과 검증

오케스트레이터 스크립트를 실행하여 3개의 키가 정상적으로 출력되는지 확인했습니다.

```bash
# 오케스트레이터 스크립트 실행
python3 src/models/bakeoff.py

# 생성된 승자 리포트 규격 검증
cat /root/code/fraud-detection/reports/winner.json
# 결과: "model_type": "logistic_regression" 등 최고 성능 모델의 메타데이터 정상 저장 확인

```

---

### 4. 핵심 개념 정리

- **Bake-off (경쟁 평가)**: 동일한 데이터셋과 평가 지표를 바탕으로 여러 가지 머신러닝 알고리즘이나 모델 아키텍처를 동시에 학습시키고 성능을 겨루게 하여 최적의 모델을 선정하는 과정입니다.

> 💡 비유하자면 **'블라인드 미각 테스트'**와 같습니다. 같은 재료(데이터)로 만든 세 명의 요리사(알고리즘)의 요리를 점수판(MLflow)에 기록하고, 심사위원장(Orchestrator 스크립트)이 가장 점수가 높은 요리사를 최종 승자로 뽑아 레시피를 채택하는 방식입니다.

- **MLflow Search API (`mlflow.search_runs`)**: MLflow 서버에 기록된 수많은 실험 결과들을 코드로 직접 쿼리(Query)하여 원하는 조건에 맞는 데이터만 데이터프레임(DataFrame) 형태로 추출하는 기능입니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 여러 모델을 학습시킨 뒤 가장 성능이 좋은 모델을 고르기 위해, MLflow UI 대시보드에 접속해 눈으로 F1 점수를 비교하고 32자리나 되는 `run_id` 해시값을 마우스로 드래그해서 복사해 오던 수동 작업이 얼마나 위험한지 깨달았습니다. 스크립트의 `order_by` 방향 하나만 잘못되어도 엉뚱한 모델이 서비스에 올라갈 수 있음을 눈으로 확인했습니다.

- **실무적 Pain Point**: 실무에서 하루에도 수십 번씩 모델이 재학습되는 환경이라면, 엔지니어가 매번 대시보드를 확인하고 최고 모델의 ID를 타이핑하여 배포 스크립트에 하드코딩할 수는 없습니다. 이 과정에서 한 글자라도 잘못 복사하면, 모델 가중치를 찾지 못해 서비스 다운타임(장애)이 발생하거나 성능이 한참 떨어지는 과거의 모델이 프로덕션에 배포되는 대형 사고로 이어집니다.

- **성장 포인트**: 이번 미션을 통해 `search_runs` API를 활용하여 MLflow에 기록된 메타데이터를 코드로 직접 질의하고, 가장 객관적인 지표에 따라 사람의 개입 없이 '가장 뛰어난 모델'을 찾아내 JSON 리포트로 자동 산출하는 오케스트레이션 로직을 체득했습니다. 이를 통해 모델의 학습부터 평가, 그리고 배포 파이프라인으로 넘기기 위한 승자 선정 과정까지 완벽하게 자동화(CI/CD)할 수 있는 MLOps 엔지니어로서의 아키텍처 설계 능력을 한 단계 높였습니다.
