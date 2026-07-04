## 📅 Day 34: Implement Cross-Validation for Model Selection

### 1. Task

- **요구사항**: xFusionCorp Industries의 ML 플랫폼 팀은 불균형한(Imbalanced) 데이터셋(약 70:30 비율) 환경에서 모델의 일반화 성능을 정확하게 측정하기 위해 교차 검증(Cross-Validation)을 수행합니다. 현재 파이프라인은 데이터 분할 시 클래스 비율을 훼손하며, 최종 생성되는 보고서의 스키마(Schema)가 배포 체크리스트 규격을 충족하지 못하는 문제가 있어 이를 교정해야 합니다.

- **목표**:
  1. `KFold` 대신 `StratifiedKFold` 분할기를 사용하여 5개의 폴드(Fold) 모두에서 원본 데이터의 클래스 비율(70:30)을 엄격하게 유지
  2. MLflow `fraud-detection-cv` 실험 내에 1개의 부모 런(Parent Run)과 5개의 자식 런(Child Run)이 중첩(Nested) 로깅되도록 기존 구조 유지
  3. 로컬 경로(`/root/code/fraud-detection/reports/cv_results.json`)에 저장되는 산출물에 7개의 최상위 키(`mean_*`, `std_*`, `folds`)가 모두 포함되도록 집계 딕셔너리 수정

---

### 2. Workflow

```text
[Data: train.csv (Imbalanced 70:30 Ratio)]
  │
  ▼
[StratifiedKFold Splitter] ──> Fold 1 ~ Fold 5 (각 Fold별 70:30 비율 강제 유지)
  │
  ▼
[MLflow Nested Logging]
  ├── Parent Run: 'cv-parent' (cv_type, n_splits 로깅)
  │    ├── Child Run: 'fold-1' (accuracy, f1, roc_auc 로깅)
  │    ├── Child Run: 'fold-2'
  │    ├── ...
  │    └── Child Run: 'fold-5'
  │
  ▼
[Aggregation & Reporting]
  └── reports/cv_results.json (mean_*, std_* 지표 산출 및 리포트 저장)

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 초기 코드 분석 및 원인 식별

파이프라인 실행 시 평가 지표가 폴드마다 심하게 요동치고, 보고서 누락 에러가 발생하는 원인을 식별했습니다.

- **원인 1**: 무작위 분할(`KFold`)을 사용하여 특정 폴드에 타겟 데이터(Fraud)가 편중되거나 아예 누락되는 현상 발생.
- **원인 2**: 최종 JSON 산출물을 구성하는 딕셔너리에 표준편차(`std_*`) 관련 키 3개가 누락됨.

#### 3-2. 데이터 분할기(CV Splitter) 교체

`src/models/cross_validate.py` 파일의 데이터 분할 객체를 수정하여, 타겟 변수(`y`)의 분포 비율을 학습 및 테스트 셋에 동일하게 반영하도록 교정했습니다.

```python
# src/models/cross_validate.py (Line 32 부근)

# [수정 전]: 무작위 분할로 인한 클래스 불균형 붕괴
# cv = KFold(n_splits=N_SPLITS, shuffle=True, random_state=42)

# [수정 후]: StratifiedKFold 적용으로 매 폴드마다 70:30 비율 보장
cv = StratifiedKFold(n_splits=N_SPLITS, shuffle=True, random_state=42)

```

#### 3-3. 집계 딕셔너리(Aggregate Schema) 규격 교정

배포 체크리스트가 요구하는 7개의 최상위 키 규격을 맞추기 위해, 넘파이(`np.std()`)를 활용하여 각 지표의 표준편차를 계산하고 딕셔너리에 추가했습니다.

```python
# src/models/cross_validate.py (Line 66 부근)

# [수정 전]: 표준편차 지표가 누락된 불완전한 스키마
# aggregate = {
#     "mean_accuracy": round(float(np.mean(acc_vals)), 6),
#     "mean_f1": round(float(np.mean(f1_vals)), 6), ...
# }

# [수정 후]: mean_* 및 std_* 지표가 완벽히 매핑된 스키마
aggregate = {
    "mean_accuracy": round(float(np.mean(acc_vals)), 6),
    "std_accuracy": round(float(np.std(acc_vals)), 6),  # [추가] 정확도 표준편차
    "mean_f1": round(float(np.mean(f1_vals)), 6),
    "std_f1": round(float(np.std(f1_vals)), 6),         # [추가] F1 스코어 표준편차
    "mean_roc_auc": round(float(np.mean(auc_vals)), 6),
    "std_roc_auc": round(float(np.std(auc_vals)), 6),   # [추가] ROC-AUC 표준편차
    "folds": fold_results,
}

```

#### 3-4. 최종 실행 및 검증

스크립트 저장 후 터미널에서 파이프라인을 재실행하여 산출물의 무결성을 검증했습니다.

```bash
# 파이프라인 실행
python3 src/models/cross_validate.py

# 로컬 JSON 산출물의 스키마 형태 확인
cat /root/code/fraud-detection/reports/cv_results.json
# 결과: "std_accuracy", "std_f1" 등의 키가 성공적으로 포함되었으며 정상 수치가 출력됨.

```

---

### 4. 핵심 개념 정리

- **Stratified K-Fold (계층적 K-폴드 교차 검증)**: 불균형한 데이터셋을 K개의 폴드로 나눌 때, 각 폴드에 포함된 데이터의 정답(Target) 클래스 비율이 원본 데이터셋의 비율과 동일하도록 분할하는 기법입니다.

> 💡 100명의 학생(남학생 70명, 여학생 30명)을 5개의 조로 나눌 때, 무작위로 뽑으면 어떤 조는 남학생만 20명이 될 수도 있습니다. Stratified K-Fold는 각 조가 반드시 '남학생 14명, 여학생 6명'의 비율을 유지하도록 공평하게 섞어서 배정해 주는 **'조 편성 선생님'**과 같습니다.

- **Nested MLflow Runs (중첩 런)**: 하나의 큰 실험(Parent Run) 아래에 여러 개의 하위 실험(Child Run)을 논리적으로 묶어서 기록하는 MLflow의 로깅 방식입니다.

> 💡 컴퓨터의 **'폴더 구조'**와 같습니다. 바탕화면에 5개의 텍스트 파일을 흩뿌려 놓는 대신, '중간고사 성적'이라는 부모 폴더를 만들고 그 안에 1과목~5과목 성적 파일을 깔끔하게 정리해 두는 것과 같습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **실무적 페인 포인트(Pain Point)**: 이번 미션처럼 보고서 스키마에서 `std_accuracy` 키 하나만 누락되어도, 이 JSON 파일을 읽어들이는 후속 시각화 대시보드나 CI/CD 파이프라인 전체가 파싱 에러를 뱉으며 셧다운됩니다. 데이터를 수동으로 눈대중해서 확인하는 습관을 버리고, 사전에 합의된 데이터 규격(Schema)을 코드 레벨에서 강제하는 꼼꼼함이 실무에서 나의 퇴근 시간을 지켜준다는 것을 명심하게 되었습니다.

- **회고(Retrospective)**: "데이터만 대충 쪼개서 학습시키면 알아서 평가되겠지"라고 가볍게 생각했다가, 불균형 데이터셋에서 일반 `KFold`를 잘못 써서 특정 폴드에 타겟 데이터가 아예 들어가지 않는 아찔한 상황을 시뮬레이션해 볼 수 있었습니다. 평가 지표가 폴드마다 널뛰기하는 것을 보며, 인프라 코드 못지않게 '데이터를 어떻게 공평하게 분배할 것인가'를 통제하는 것이 엔지니어링의 중요한 축임을 체감했습니다.

- **성장 포인트**: MLflow의 부모/자식(Nested) 런 구조를 직접 검증해 보며, 복잡한 교차 검증 이력(전체 평균 지표와 5개 폴드 각각의 개별 지표)을 한눈에 추적하는 중앙 집중형 로깅의 강력함을 배웠습니다. 앞으로 현업 동료들과 협업할 때, "어제 돌린 3번 폴드 성능이 튀던데 이유가 뭘까요?"라며 MLflow 링크 하나로 깔끔하고 프로페셔널하게 소통할 수 있는 기반을 다졌습니다.
