# 📅 Day 31: Train a Scikit-Learn Model with Reproducible Script

### 1. Task

- **요구사항**: 파이썬 소스 코드를 하드코딩 방식으로 변경하지 않고, 설정 파일 수정을 통해 다양한 하이퍼파라미터 및 알고리즘을 실험할 수 있는 유연한 ML 파이프라인 인프라를 구축합니다. 기존 `train_config.yaml` 파일 내의 문법 및 명세 오류를 찾아내어 MLflow 트래킹 인프라와 정상 연동되도록 조치합니다.

- **목표**:
  1. MLflow 트래킹 서버(`http://localhost:5000`)의 `fraud-detection` 실험에 정확히 1개의 학습 Run 기록 적재
  2. 트레이너 스크립트가 지원하는 정확한 에스티메이터 명칭(`RandomForestClassifier`) 매핑 및 target 데이터 컬럼 동기화
  3. 학습 완료 후 직렬화된 모델 파일(`model.pkl`)을 지정된 절대 경로에 안전하게 저장

---

### 2. Workflow

```text
[train_config.yaml] 교정
  │
  ├── model.type: RandomForestClassifier (지원 에스티메이터 명칭 일치)
  ├── data.target_column: is_fraud (데이터셋 헤더 컬럼 일치)
  └── output.model_path: /root/code/fraud-detection/models/model.pkl
  │
  ▼
[Execution: train.py]
  │
  ├── 1. 로컬 데이터셋 로드 & 전처리
  ├── 2. MLflow 로컬 서버 트래킹 엔드포인트 연동 (Port: 5000)
  └── 3. Scikit-learn 모델 학습 및 메트릭 산출 (Accuracy, F1-Score)
  │
  ▼
[End State]
  ├── Local: /root/code/fraud-detection/models/model.pkl 가용성 확보
  └── MLflow Dashboard: 'fraud-detection' Experiment 내 하이퍼파라미터/메트릭 저장 완료

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 초기 상태 분석 및 에러 원인 식별

디렉토리 이동 후 파이프라인을 실행하여 표준 에러 로그를 분석했습니다.

```bash
# 작업 디렉토리 이동 및 스크립트 1차 실행
cd /root/code/fraud-detection/
python3 src/models/train.py

# 발생 에러 확인
ERROR: unknown estimator type 'RandomForest'. Supported: ['GradientBoostingClassifier', 'LogisticRegression', 'RandomForestClassifier']

```

- **원인 분석**: 스크립트가 지원하는 RandomForest 클래스명은 `RandomForestClassifier`이지만 YAML 파일에는 `RandomForest`로 오기입되어 인스턴스화 실패가 발생했습니다.

#### 3-2. 설정 파일(YAML) 수정 및 픽셀 단위 톺아보기

VS Code를 통해 기존의 잘못된 매핑 정보와 경로 설정을 프로덕션 규격에 맞게 전면 교정했습니다.

```yaml
# configs/train_config.yaml 수정 본
model:
  type: RandomForestClassifier # [수정] 지원 에스티메이터 규격명으로 명확히 변경
  n_estimators: 100
  max_depth: 5
  random_state: 42
data:
  train_path: /root/code/fraud-detection/data/train.csv
  target_column: is_fraud # [수정] 데이터셋(train.csv)의 정답 레이블 컬럼명 명시
output:
  model_path: /root/code/fraud-detection/models/model.pkl # [수정] 모델 파일 저장 절대 경로 지정
mlflow:
  tracking_uri: http://localhost:5000
  experiment_name: fraud-detection
```

#### 3-3. 파이프라인 재실행 및 최종 검증

교정된 설정을 기반으로 전체 파이프라인을 실행하고 산출물 상태를 검증했습니다.

```bash
# 학습 파이프라인 재구동
python3 src/models/train.py
# 출력 로그: accuracy=0.8000, f1_score=0.8261 정상 출력 완료
# 출력 로그: View run bright-panda-273 at: http://localhost:5000/#/experiments/1/...

# 생성된 모델 파일 검증
ls -l /root/code/fraud-detection/models/model.pkl
# -rw-r--r-- 1 root root 318521 Jun 30 11:01 /root/code/fraud-detection/models/model.pkl

```

---

### 4. 핵심 개념 정리

- **Config-driven Architecture (설정 기반 아키텍처)**: 소스 코드 내부 논리나 하이퍼파라미터를 하드코딩하지 않고 외부의 YAML, JSON 등의 파일로 분리하여 관리하는 아키텍처 설계 방식입니다. 개발자는 코드를 건드리지 않고 설정 파일만 교체하여 다양한 실험 환경을 제어할 수 있습니다.

  > 💡 믹서기 내부의 모터나 회로를 뜯어고칠 필요 없이, 전면부의 '다이얼 버튼'만 돌려서 분쇄 속도나 시간을 조절하는 것과 같습니다. 코드가 믹서기 본체라면 YAML 파일은 외부 조작 다이얼입니다.

- **MLflow Tracking**: 머신러닝 학습 코드 실행 시 사용한 하이퍼파라미터, 소스 코드 버전, 메트릭(성능 결과값), 산출물(Model Artifact)을 중앙 집중형 서버에 기록하고 시각화해 주는 오픈소스 라이브러리입니다.

  > 💡 자동차의 '블랙박스 다이어리'와 같습니다. 주행 시 속도, 급정거 기록, GPS 좌표가 블랙박스에 자동으로 남아 나중에 주행 습관을 분석할 수 있듯, 학습 과정의 모든 수치 변화를 서버에 타임라인 순으로 박제하여 모니터링할 수 있게 합니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **실무 관점의 의의**: 프로덕션 MLOps 파이프라인 환경에서는 수많은 데이터 과학자들이 동시에 수백 번의 실험을 수행합니다. 이때 코드를 직접 수정하게 하면 형상 관리가 깨지고 휴먼 에러가 발생합니다. 이번 실습을 통해 설정 파일(Config) 레이어와 로직(Python) 레이어를 완벽히 격리하는 구조적 설계의 중요성을 체득했습니다.

- **RCA(근본 원인 분석) 역량**: 단순히 첫 번째 에러(`unknown estimator`)만 해결하고 넘어가는 것이 아니라, 전체 설정 파일의 명세와 실제 로컬 인프라 환경(데이터셋 컬럼명, 타겟 디렉토리 가용성)을 원점 검증(Origin Validation)함으로써 연쇄적인 런타임 크래시를 미연에 방지할 수 있는 방어적 엔지니어링 사고 과정을 훈련했습니다.
