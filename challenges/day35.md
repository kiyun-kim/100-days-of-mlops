## 📅 Day 35: Hyperparameter Tuning with Optuna

### 1. Task

- **요구사항**: xFusionCorp Industries의 ML 플랫폼 팀은 사기 탐지 모델의 하이퍼파라미터(`n_estimators`, `max_depth`)를 최적화하기 위해 Optuna를 도입했습니다. 탐색된 20번의 실험(Trial) 결과를 MLflow Compare 뷰에서 시각적으로 비교 분석할 수 있도록, 튜너 스크립트(`tune.py`)의 탐색 방향 오류를 바로잡고 개별 Trial에 대한 독립적인 로깅 로직을 구축해야 합니다.

- **목표**:
  1. Optuna Study의 최적화 방향(Direction)을 F1 점수 '최대화(maximize)'로 교정
  2. `hyperopt-tuning` 실험(Experiment) 내에 20개의 탐색 시도(Trial)가 각각 독립적인 런(Run)으로 생성되도록 MLflow 로깅 로직 추가 (파라미터 및 지표 적재)
  3. 로컬 경로(`/root/code/fraud-detection/configs/best_params.yaml`)에 탐색된 최적의 파라미터 조합 산출물 저장

---

### 2. Workflow

```text
[Optuna Hyperparameter Tuning Pipeline]
  │
  ├── Trial 1: (n_estimators=100, max_depth=5) ──> [MLflow Log: f1_score=0.42]
  ├── Trial 2: (n_estimators=250, max_depth=12) ──> [MLflow Log: f1_score=0.45]
  ├── ... (반복 탐색 및 검증)
  └── Trial 20: (n_estimators=246, max_depth=16) ──> [MLflow Log: f1_score=0.47]
  │
  ▼
[Aggregation & Export]
  └── configs/best_params.yaml (최고 성능을 기록한 하이퍼파라미터 세트 저장 완료)

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. Optuna 탐색 방향(Direction) 교정

`src/models/tune.py` 파일 내 `main()` 함수에서 Optuna Study 객체를 생성할 때, 목표 지표인 F1 스코어의 특성에 맞게 탐색 방향을 수정했습니다.

```python
# src/models/tune.py (Line 50 부근)

# [수정 전]: 에러율이나 손실(Loss) 관점의 잘못된 최소화 설정
# study = optuna.create_study(
#     direction="minimize", study_name=EXPERIMENT_NAME
# )

# [수정 후]: F1 스코어는 값이 클수록 성능이 좋으므로 최대화 설정으로 변경
study = optuna.create_study(
    direction="maximize", study_name=EXPERIMENT_NAME
)

```

#### 3-2. 개별 Trial에 대한 MLflow 로깅 로직 추가

각 탐색 시도마다 사용된 하이퍼파라미터와 그에 따른 평가 점수를 MLflow 대시보드에 기록하기 위해, `objective()` 함수 내부에 `mlflow.start_run()` 컨텍스트를 추가했습니다.

```python
# src/models/tune.py (Line 43 부근)

def objective(trial, X, y):
    # (생략) 파라미터 샘플링 및 교차 검증 점수(score) 연산 로직
    score = float(np.mean(scores))

    # [추가]: 개별 Trial 종료 시점에 파라미터와 산출된 메트릭을 MLflow에 기록
    with mlflow.start_run():
        mlflow.log_param("n_estimators", n_estimators)
        mlflow.log_param("max_depth", max_depth)
        mlflow.log_metric("f1_score", score)

    return score

```

#### 3-3. 파이프라인 구동 및 최종 산출물 검증

스크립트 수정을 완료한 후 파이프라인을 구동하여 20번의 탐색이 정상 수행되었는지, 최적 파라미터가 YAML로 저장되었는지 확인했습니다.

```bash
# 튜너 스크립트 실행 (20번의 Trial 자동 진행)
python3 src/models/tune.py

# 생성된 최적 파라미터 구성 파일(YAML) 확인
cat /root/code/fraud-detection/configs/best_params.yaml
# 출력 결과: max_depth: 16, n_estimators: 246 (정상 저장 확인)

```

---

### 4. 핵심 개념 정리

- **Optuna**: 머신러닝 모델의 하이퍼파라미터 최적화를 자동화해 주는 프레임워크입니다. 지정된 탐색 공간(Search Space) 내에서 이전 시도의 결과를 바탕으로 다음 시도에 더 나은 조합을 지능적으로 제안합니다.

> 💡 라디오 주파수를 맞출 때, 사람이 손으로 다이얼을 무작위로 조금씩 돌려가며 지지직거리는 소리를 없애는 것이 기존 방식(Grid/Random Search)이라면, Optuna는 '가장 선명한 소리가 나는 주파수 대역을 스스로 추적하여 맞춰주는 스마트 자동 채널 검색 기능'과 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 파라미터를 수동으로 하나씩 바꿔가며 스크립트를 실행하고, 그 결과를 엑셀이나 메모장에 일일이 기록하던 과거의 방식은 비효율의 극치였습니다. Optuna와 MLflow를 연동해 보니, 스크립트 한 번의 실행으로 20번의 실험이 자동으로 진행되고 그 결과가 중앙 대시보드에 일목요연하게 정리되는 것을 보며 자동화 파이프라인의 진정한 위력을 체감했습니다.

- **실무적 페인 포인트(Pain Point)**: 수동으로 하이퍼파라미터를 튜닝하다 보면 "방금 돌린 파라미터 조합이 뭐였지?" 하며 값을 잘못 적거나, 가장 성능이 좋았던 구성을 덮어써서 분실하는 휴먼 에러가 빈번하게 발생합니다. 이는 결국 모델 학습을 처음부터 다시 시작하게 만들어 막대한 컴퓨팅 리소스와 엔지니어의 시간을 낭비하게 만드는 주된 원인입니다.

- **성장 포인트**: 이제는 파라미터를 직감으로 때려 맞추는 것이 아니라, 정의된 탐색 공간 내에서 시스템이 최적값을 스스로 찾고 그 모든 과정을 MLflow에 영구적으로 기록하는 구조를 설계할 수 있게 되었습니다. 앞으로 데이터 과학자나 팀원들과 협업할 때, "파라미터를 바꿔가며 수동으로 기록하지 마시고 이 튜닝 스크립트만 실행하십시오. 결과는 MLflow 대시보드의 Compare 뷰에서 직관적으로 함께 분석하면 됩니다"라고 제안할 수 있는 프로페셔널한 인프라 운영 시각을 갖추게 되었습니다.
