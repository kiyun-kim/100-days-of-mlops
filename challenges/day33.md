## 📅 Day 33: Evaluate a Trained Model and Generate Classification Report

### 1. Task

- **요구사항**: xFusionCorp Industries의 ML 플랫폼 팀 모델 배포 체크리스트를 충족하기 위해, 모델의 성능을 다각도로 검증하는 5가지 필수 평가 지표(Metrics)와 오차 행렬(Confusion Matrix)을 자동 생성하고 중앙 저장소에 로깅하는 평가 스크립트(`evaluate.py`)의 버그 및 경로 오류를 교정합니다.

- **목표**:
  1. MLflow `fraud-detection-eval` 실험(Experiment)에 1개의 Run 생성 및 평가 결과 적재
  2. `accuracy`, `precision`, `recall`, `f1_score`, `auc_roc` 총 5가지의 메트릭이 정확한 키 이름으로 평가 보고서에 포함
  3. 로컬의 `/root/code/fraud-detection/reports/` 절대 경로에 `metrics.json`과 `confusion_matrix.png` 산출물 저장

---

### 2. Workflow

```text
[Data & Pre-trained Model]
  ├── models/model.pkl
  └── data/test.csv
       │
       ▼
[src/models/evaluate.py 실행]
  ├── 1. 예측(Predict) 및 확률(Predict Proba) 연산
  ├── 2. 5대 분류 메트릭 산출 (Accuracy, Precision, Recall, F1, AUC-ROC)
  └── 3. 오차 행렬 이미지 시각화 생성
       │
       ▼
[End State: Artifacts & Logging]
  ├── Local Filesystem: /reports/metrics.json, /reports/confusion_matrix.png 저장 완료
  └── MLflow Server: 'fraud-detection-eval' 런타임에 JSON 및 이미지 Artifact 등록 완료

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 기존 스크립트의 에러 증상 및 원인 분석

평가 스크립트 실행 후 산출물이 지정된 위치에 없는 것을 확인하고 원인을 분석했습니다.

- **원인 1**: 산출물 저장 경로가 임시 디렉토리(`/tmp/metrics.json`)로 하드코딩되어 있었습니다.
- **원인 2**: 요구사항에 명시된 `precision`, `recall`, `auc_roc` 지표가 누락되었고, F1 스코어의 키 이름이 `f1`으로 잘못 기재되어 있었습니다.

#### 3-2. 전역 변수(저장 경로) 수정

로컬 산출물이 올바른 프로젝트 디렉토리 하위에 저장되도록 `src/models/evaluate.py` 상단의 경로 상수를 변경했습니다.

```python
# src/models/evaluate.py 상단 수정 내역

# [수정 전]: 잘못된 임시 경로
# METRICS_JSON = "/tmp/metrics.json"

# [수정 후]: 요구사항에 부합하는 절대 경로로 교체
METRICS_JSON = "/root/code/fraud-detection/reports/metrics.json"

```

#### 3-3. 5대 핵심 메트릭 로직 추가 및 키 이름 동기화

`main()` 함수 내부의 지표 산출 딕셔너리를 수정하여 누락된 함수를 추가하고 요구사항과 100% 일치하는 포맷을 구성했습니다. 특히 `auc_roc`의 경우 단순 예측 결과(`preds`)가 아닌 클래스별 확률값(`proba`)이 필요하므로 이를 반영했습니다.

```python
# src/models/evaluate.py (Line 60 부근)

# [수정 전]
# metrics = {
#     "accuracy": round(accuracy_score(y, preds), 6),
#     "f1": round(f1_score(y, preds), 6),
# }

# [수정 후]
metrics = {
    "accuracy": round(accuracy_score(y, preds), 6),
    "precision": round(precision_score(y, preds), 6),
    "recall": round(recall_score(y, preds), 6),
    "f1_score": round(f1_score(y, preds), 6),       # 요구사항 키명(f1_score)으로 변경
    "auc_roc": round(roc_auc_score(y, proba), 6)    # 예측 확률(proba)을 사용하는 AUC-ROC 추가
}

```

#### 3-4. 파이프라인 재실행 및 MLflow 적재 검증

스크립트 수정 완료 후 재실행하여 산출물 생성 및 로깅 무결성을 최종 확인했습니다.

```bash
# 평가 스크립트 최종 실행
python3 src/models/evaluate.py

# reports 디렉토리 내 파일 생성 여부 및 내용 확인
cat /root/code/fraud-detection/reports/metrics.json
# {"accuracy": 0.8..., "precision": 0.8..., "recall": 0.8..., "f1_score": 0.8..., "auc_roc": 0.8...} 출력 정상 확인

```

---

### 4. 핵심 개념 정리

- **분류 모델 평가 지표 (Precision, Recall, F1, AUC-ROC)**: 모델이 데이터를 얼마나 잘 분류했는지 다각도로 평가하는 기준입니다. 단순 정확도(Accuracy)만으로는 데이터 불균형이 있을 때 모델의 진짜 성능을 파악하기 어렵기 때문에 여러 지표를 함께 확인해야 합니다.

> 💡 모델 평가를 **'건강검진 암 진단'**에 비유할 수 있습니다.
>
> - **Accuracy (정확도)**: 전체 환자 중 정상인과 암 환자를 올바르게 맞춘 비율입니다.
> - **Precision (정밀도)**: 의사가 "암입니다"라고 진단한 사람 중, 실제로 암에 걸린 사람의 비율입니다. (오진 확률이 낮아야 높음)
> - **Recall (재현율)**: 실제 암 환자 중에서 의사가 놓치지 않고 "암입니다"라고 찾아낸 비율입니다. (놓치는 환자가 없어야 높음)
> - **F1-Score**: 정밀도와 재현율이 한쪽으로 치우치지 않게 균형을 잡아주는 통합 점수입니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **엔지니어의 회고(Retrospective)**: 터미널 창에 출력된 평가 지표를 드래그해서 엑셀에 수동으로 복사해 붙여넣거나, 각 모델마다 평가 스크립트가 중구난방으로 작성되어 있다면, 이는 휴먼 에러가 발생하기 가장 좋은 환경입니다. `f1`을 `f1_score`로 잘못 적어 통합 대시보드가 깨지는 등 사소한 오타 하나가 나중에는 전체 파이프라인의 에러로 스노우볼이 되어 돌아온다는 페인 포인트를 직접 확인했습니다.

- **실무적 페인 포인트(Pain Point)**: 수십 번의 하이퍼파라미터 튜닝이 일어나는 실무 환경에서, 결과를 로컬 파일에만 두거나 표준화되지 않은 키(Key) 값으로 관리하면 지표를 비교 분석하는 데 불필요한 노가다 시간이 소요됩니다. 출력 경로와 데이터 규격(Schema)을 강제하는 스크립트 작성은 선택이 아닌 필수입니다.

- **성장 포인트**: 이번 실습을 통해 모든 평가 지표를 `metrics.json`이라는 단일 규격으로 통일하고 MLflow라는 중앙 저장소에 자동 적재하는 구조를 완성했습니다. 이를 통해 앞으로 동료 엔지니어나 데이터 과학자와 협업할 때 "어제 돌린 모델 정확도가 몇이었죠?"라고 묻는 대신, "MLflow 대시보드에 로깅된 `f1_score` 추이를 확인해 보세요"라고 말할 수 있는 훨씬 프로페셔널하고 일관된 파이프라인 운영 방식을 체득했습니다.
