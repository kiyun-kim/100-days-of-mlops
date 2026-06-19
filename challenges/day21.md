## 📅 Day 21: Log an ML Experiment to MLflow

### 1. Task

- **요구사항**: 모델의 지속적인 개선과 협업을 위해 데이터 과학자가 수행한 학습 실험의 하이퍼파라미터, 성능 메트릭 및 최종 모델 아티팩트를 중앙 집중형 추적 서버에 누락 없이 기록하고 버전 관리 환경을 구축합니다.

- **목표**:
  - `Default` 익스페리먼트 내에 신규 실행(Run)을 생성하고 추적 정보 바인딩
  - 모델 학습 파라미터(`n_estimators`, `max_depth`, `random_state`) 자동 기록
  - 검증 메트릭(`accuracy`, `f1_score`) 기록
  - 학습된 Scikit-learn 모델을 MLflow 아티팩트로 저장

---

### 2. Workflow

```text
+-------------------------------------------------------------+
|                /root/code/log_experiment.py                 |
|                                                             |
|  [1. Load Params] -> [2. Fit Scikit-Learn Model]            |
|                                                             |
|  [3. MLflow Context Start] (mlflow.start_run())             |
|       |                                                     |
|       +---> mlflow.log_params(params)                       |
|       +---> mlflow.log_metrics({"accuracy", "f1_score"})    |
|       +---> mlflow.sklearn.log_model(model, "model")        |
+-------------------------------------------------------------+
                               |
                               v (Port 5000)
+--------------------------------------------------------------+
|                     MLflow Tracking Server                   |
|  - Experiments: Default                                      |
|  - Parameters / Metrics / Artifacts (Model Binary) Registered|
+--------------------------------------------------------------+

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 소스 코드 수정 (Python SDK 연동)

`/root/code/log_experiment.py` 스크립트 내 `mlflow.start_run()` 블록 내부의 TODO를 MLflow 표준 API로 채워 넣었습니다.

```python
import mlflow
import mlflow.sklearn
# ... (중략: 데이터 로드 및 모델 초기화 부분)

with mlflow.start_run():
    # TODO 1: 하이퍼파라미터 딕셔너리를 일괄 기록합니다.
    # 단일 기록(log_param) 대신 log_params를 사용하여 딕셔너리 구조를 그대로 매핑합니다.
    mlflow.log_params(params)

    # TODO 2: 평가 메트릭인 accuracy와 f1 점수를 매핑하여 기록합니다.
    # 명세서의 요구 명칭("accuracy", "f1_score")에 맞추어 키 값을 지정합니다.
    mlflow.log_metrics({"accuracy": accuracy, "f1_score": f1})

    # TODO 3: 학습 완료된 객체를 MLflow 모델 아티팩트로 등록합니다.
    # artifact_path를 "model"로 지정하여 루트 아티팩트 디렉터리 하위에 격리 저장합니다.
    mlflow.sklearn.log_model(model, artifact_path="model")

    print(f"accuracy={accuracy}, f1_score={f1}")

```

#### 3-2. 스크립트 실행 및 로그 검증

수정된 파이썬 스크립트를 실행하여 로깅을 수행하고 정상 출력 여부를 검증했습니다.

```bash
# 스크립트 실행
root@controlplane ~/code via 🐍 v3.12.3
$ python /root/code/log_experiment.py

# 실행 결과 (정상 로그 및 아티팩트 저장 가이드라인 표출 확인)
2026/06/19 10:13:37 WARNING mlflow.models.model: `artifact_path` is deprecated. Please use `name` instead.
accuracy=0.92, f1_score=0.89
🚀 View run ambitious-swan-283 at: http://localhost:5000/#/experiments/0/runs/0cdf445d67a6488d9aa7213c2af54b09
📈 View experiment at: http://localhost:5000/#/experiments/0

```

---

### 4. 핵심 개념 정리

- **MLflow Tracking**: 머신러닝 모델의 파라미터, 코드 버전, 메트릭, 산출물(아티팩트) 등을 로깅하고 시각화할 수 있는 API 및 UI 기반의 중앙 집중형 컴포넌트입니다.

💡 비유하자면 **'실험실의 실험 일지 및 보관함'**. 어떤 재료(하이퍼파라미터)를 넣어서 어떤 결과(메트릭)가 나왔는지 일지에 꼼꼼히 적고, 만들어진 결과물(모델)을 이름표를 붙여 과학실 보관함(아티팩트)에 안전하게 넣어두는 것과 같습니다. 이렇게 해두면 나중에 다른 사람이 와도 똑같이 재현할 수 있습니다.

- **실무 관점의 의의**: 프로덕션 환경의 MLOps 인프라에서 모델 로깅의 자동화는 모델 추적성(Traceability) 확보의 출발점입니다. 엔지니어는 수많은 모델 버전 중 어떤 조건으로 학습된 모델이 최상의 서빙 성능을 냈는지 역추적할 수 있어야 하며, 이를 통해 지속적 학습(Continuous Training) 파이프라인의 데이터 계보(Lineage)를 완벽하게 관리할 수 있습니다.

---

### 5. 무엇을 배웠는가

- **API 스펙 준수의 중요성**: 하이퍼파라미터나 메트릭을 개별적으로 여러 번 호출하는 것보다, `log_params()` 및 `log_metrics()` 처럼 딕셔너리 구조를 활용한 일괄 로깅(Bulk Logging)을 수행하는 것이 네트워크 I/O 병목을 줄이고 코드를 간결하게 유지하는 구조적 접근법임을 배웠습니다.

- **추적 서버 인프라의 격리**: 코드 내부에서 모델을 로컬 디렉터리에 파일로 관리하는 전통적인 방식에서 벗어나, 추적 서버(Tracking Server)라는 전용 인프라 레이어로 상태를 분리(Isolate)함으로써 마이크로서비스 형태의 MLOps 아키텍처에 대응하는 역량을 길렀습니다.
