## 📅 Day 32: Manage Training Configuration with YAML

### 1. Task

- **요구사항**: xFusionCorp Industries의 ML 플랫폼 팀 감사(Audit) 파이프라인은 런(Run) 간 완전한 재현성을 요구합니다. 동일한 데이터셋과 코드를 사용하여 파이프라인을 구동했을 때 난수 생성에 의한 메트릭 변동을 차단하고, 항상 동일한 결과를 출력하도록 스크립트를 결정론적(Deterministic) 상태로 교정해야 합니다.

- **목표**:
  1. 난수 시드(Seed) 고정을 통해 `check_determinism.sh` 무결성 검증 스크립트를 오류(Exit Status `0`) 없이 통과
  2. MLflow `fraud-detection-repro` 실험 내 생성된 런들의 정확도(Accuracy) 및 F1 점수 소수점 6자리까지 완벽 일치
  3. 연속된 3번의 런 결과물(metrics JSON)의 바이트 단위 완전 일치 확보

---

### 2. Workflow

```text
[Non-Deterministic State (수정 전)]
Data ──> train_test_split (무작위 분할) ──> RandomForest (무작위 앙상블) ──> Metrics 불일치 ──> Audit FAIL

[Deterministic State (수정 후 목표 상태)]
Data ──> train_test_split (Seed: 42 고정)
         │
         └──> RandomForest (Seed: 42 고정) ──> Metrics 100% 일치 ──> check_determinism.sh (Exit 0)
                                           └──> MLflow Logging (일관된 메트릭 및 아티팩트 적재)

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 초기 상태 점검 및 원인 식별

현재 파이프라인의 재현성 실패 증상을 확인하기 위해 검증 스크립트를 실행하여 오류 로그를 확인했습니다.

```bash
# 작업 디렉토리 이동 및 무결성 검증 스크립트 실행
cd /root/code/fraud-detection/
./check_determinism.sh

# 출력 결과
# FAIL: the three runs did not produce byte-identical metrics. (diff 로그 발생)

```

#### 3-2. 학습 스크립트(train.py) 난수 시드 고정

데이터 분할 및 모델 가중치 초기화 과정에서 발생하는 무작위성을 제어하기 위해, `src/models/train.py` 내 Scikit-learn 함수들에 `random_state` 파라미터를 명시적으로 할당했습니다. 기존에 존재하던 `stratify=y` 및 `max_depth=5` 설정은 무결성 유지를 위해 그대로 보존했습니다.

```python
# src/models/train.py 수정 내역 (Line 37, 41 부근)

# [수정 전]: 무작위 데이터 분할 (실행마다 학습/테스트 셋이 달라져 성능 요동 발생)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y
)
# [수정 후]: random_state=42 추가로 데이터 분할 기준 고정
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

# [수정 전]: 앙상블 모델 내부 트리의 무작위 피처 샘플링 발생
model = RandomForestClassifier(n_estimators=100, max_depth=5)
# [수정 후]: random_state=42 추가로 알고리즘의 결정론적 동작 보장
model = RandomForestClassifier(n_estimators=100, max_depth=5, random_state=42)

```

#### 3-3. 파이프라인 재실행 및 최종 무결성 검증

스크립트 수정 저장 후, 검증 스크립트를 재실행하여 메트릭 일치 여부와 시스템 종료 코드를 확인했습니다.

```bash
# 무결성 검증 스크립트 재실행
./check_determinism.sh
# 출력 로그: OK: all three runs produced byte-identical metrics.

# 쉘 스크립트 종료 상태 코드 확인
echo $?
# 출력 로그: 0 (정상 통과)

```

---

### 4. 핵심 개념 정리

- **재현성(Reproducibility)과 결정론적(Deterministic) 제어**: 소프트웨어나 알고리즘이 동일한 입력(Data, Code)을 받았을 때, 실행 횟수나 환경에 상관없이 항상 100% 동일한 출력(Metrics, Model)을 반환하는 성질을 의미합니다. 머신러닝에서는 데이터 분할, 가중치 초기화 등의 과정에 난수(Random Number)가 필연적으로 개입하므로, 난수 생성 알고리즘의 시작점인 '시드(Seed)'를 고정하여 이를 인위적으로 제어합니다.

> 💡 난수 생성기를 '음악 앱의 랜덤 재생(Shuffle) 기능'이라고 가정해 보겠습니다. 매번 노래가 무작위로 나오면 어제 들은 순서대로 다시 들을 수 없습니다. 하지만 시드(Seed)를 고정하는 것은 음악 앱에 특정 '랜덤 믹스테이프의 고유 번호'를 입력하는 것과 같습니다. 겉보기에는 무작위로 섞인 곡들이지만, 해당 고유 번호를 입력하는 한 언제나 똑같이 섞인 재생 목록을 완벽하게 재현해 낼 수 있습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **실무 관점의 의의**: 엔터프라이즈 MLOps 환경에서 모델의 성능 지표(Accuracy, F1 Score 등)가 변경되었을 때, 이것이 '데이터 전처리 로직 변경이나 하이퍼파라미터 튜닝' 덕분인지, 아니면 단순한 '난수 분할에 의한 운'인지 명확하게 증명할 수 있어야 합니다. 재현성이 결여된 파이프라인은 감사(Audit)와 A/B 테스트의 신뢰도를 근본적으로 무너뜨리며, 지속적 통합(CI) 자동화를 불가능하게 만듭니다.

- **실무 배포를 위한 필수 요건**: 기업의 프로덕션 환경이나 금융권 등 규제가 엄격한 인프라에서는 결과가 매번 바뀌는 모델을 배포할 수 없습니다. 결과를 100% 검증하고 똑같이 복제해 낼 수 있는 '결정론적 제어(시드 고정)'가 AI 인프라 운영의 기본 전제임을 배웠습니다.

- **검증의 자동화 필요성**: 이번 미션에서 수행한 `check_determinism.sh`와 같은 무결성 검증은 실무에서 엔지니어가 수동으로 하는 것이 아니라, 코드가 푸시될 때마다 검증 스크립트가 자동 실행되도록 CI(지속적 통합) 파이프라인에 녹여내야 사람의 실수를 방지할 수 있다는 인사이트를 얻었습니다.
