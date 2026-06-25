## 📅 Day 27: Load Model from Registry with Custom Preprocessing

### 1. Task

- **요구사항**: 모델 레지스트리(Model Registry)에 등록된 최상위 운영 모델(`fraud-detector@champion`)을 개발 환경으로 리드(Load)하고, 원시 데이터가 모델에 유입되기 전 스케일링 전처리를 수행하는 프로덕션 추론 파이프라인 스크립트를 완성하여 배치 결과 자동 파일화 환경을 구축합니다.

- **목표**:
  1. `models:/fraud-detector@champion` 포인터를 통한 전용 모델 아티팩트 다운로드 및 로드
  2. 전처리용 커스텀 추론 객체 `ScaledPredictor`와 MLflow 내장 엔진의 상호작용 규격 동기화
  3. 배치 추론 결과를 `prediction` 컬럼으로 결합하여 헤더를 포함한 11행의 CSV 산출물 생성

---

### 2. Workflow

```text
[ MLflow Model Registry ] ──> models:/fraud-detector@champion
                                      │
                                      v (mlflow.pyfunc.load_model)
[ Production Batch Script ] ──────────────────────────────────────────+
  ├── 1. Load Model into 'inner_model'                                |
  ├── 2. Compute Mean/Std from /root/code/data/inputs.csv             |
  ├── 3. Instantiate ScaledPredictor(inner_model, mean, std)          |
  └── 4. Execute .predict(None, inputs.values)                        |
                                                                      |
 [Output Asset] ──> Write to /root/code/predictions.csv (index=False) |
+---------------------------------------------------------------------+

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. MLflow PyFunc 모델 동적 로드 소스 자산 수정

`predict_with_preprocessing.py` 파일의 첫 번째 TODO 영역에서 모델 레지스트리 허브와 통신하여 에일리어스(`@champion`) 기반의 추록 바이너리를 가용 변수에 바인딩했습니다.

```python
import mlflow.pyfunc
import pandas as pd

# TODO 1: MLflow Model Registry에서 프로덕션 전용 champion 모델을 pyfunc 레이어로 로드합니다.
inner_model = mlflow.pyfunc.load_model(MODEL_URI)

```

#### 3-2. 전처리 래퍼 인터페이스 동기화 및 트러블슈팅

1. **트러블슈팅 및 정정 (TypeError)**: 초기에 `predictor.predict(inputs)` 형태로 일반적인 데이터프레임을 직접 전달했을 때 `TypeError: ScaledPredictor.predict() missing 1 required positional argument: 'model_input'` 아키텍처 에러가 관측되었습니다.
2. **원인 분석**: 코드 내에 임베디드된 Custom 전처리 클래스 `ScaledPredictor`는 MLflow 고유의 `PythonModel` 표준을 상속받아 빌드되었습니다. 이 표준 규격 상 `.predict()` 메서드의 첫 번째 인자(`context`)는 시스템 제어용 내부 변수(None 허용)이며, 실제 학습 데이터셋은 두 번째 인자인 `model_input`으로 바인딩되도록 강제되어 있었습니다.
3. **최종 코드 수정**: 인프라 서명 구조를 완전히 정정한 코드를 두 번째 TODO 블록에 반영했습니다.

```python
# 입력 데이터 로드 및 통계치 추출 스캐폴딩
inputs = pd.read_csv(INPUT_CSV)
mean = inputs.values.mean(axis=0)
std = inputs.values.std(axis=0)
std[std == 0] = 1.0
predictor = ScaledPredictor(inner_model, mean, std)

# TODO 2: MLflow PyFunc 가이드 표준(context, model_input)에 의거하여 인자를 매핑하고 디스크에 저장합니다.
predictions = predictor.predict(None, inputs.values)
inputs['prediction'] = predictions
inputs.to_csv(OUTPUT_CSV, index=False)

```

#### 3-3. 스크립트 파이프라인 가동 및 무결성 검증

터미널 인프라 버퍼에서 파이썬 엔진을 구동하고 산출물 파일의 행 수와 타겟 데이터 유실 여부를 완벽히 교차 검증했습니다.

```bash
# 1. 배치 파이프라인 스크립트 가동
root@controlplane ~/code via 🐍 v3.12.3
$ python3 /root/code/predict_with_preprocessing.py

# 2. 산출물 CSV 파일 엔티티 로그 및 컬럼 매핑 유효성 분석
root@controlplane ~/code via 🐍 v3.12.3
$ cat /root/code/predictions.csv
feature_a,feature_b,prediction
0.1,0.5,0
0.3,0.2,0
0.8,0.9,0
0.2,0.4,0
0.6,0.1,0
0.5,0.7,0
0.9,0.3,0
0.4,0.8,0
0.7,0.6,0
0.0,1.0,0

```

---

### 4. 핵심 개념 정리

#### 4-1. MLflow PyFunc (Python Model Flavor)

- **전문적 정의**: 다양한 머신러닝 프레임워크(Scikit-Learn, PyTorch 등)의 고유 모델 객체들을 MLflow가 자체 제공하는 표준 `pyfunc` 형태의 단일 공통 포맷으로 추상화하여 감싸는 범용 래퍼 인터페이스입니다.
  💡 **비유하자면**: **'국가별 플러그를 하나로 합쳐주는 만능 돼지코 어댑터'**. 110V를 쓰든 220V를 쓰든(Sklearn이든 PyTorch든) 상관없이 전 세계 공용 멀티 어댑터(`pyfunc`)에 꽂아 넣기만 하면 벽면 콘센트(추론 서버 인프라)에 형태 변환 없이 바로 연결하여 전기를 쓸 수 있게 해주는 도구입니다.

#### 4-2. 전처리 래퍼(PyFunc Wrapper) 가이드를 엄격히 맞춰야 하는 이유

- **전문적 정의**: MLflow의 `PythonModel` 아키텍처는 가상 머신이나 컨테이너 서비스에 배포되어 예측을 수행할 때 반드시 `predict(self, context, model_input)`라는 고정된 API 시그니처 서명(Signature)을 준수하도록 인프라 프로토콜이 설계되어 있습니다.

- **실무 관점의 이유**:
  1. **시스템 연계 표준화**: 만약 개발자가 편의대로 `predict(inputs)`처럼 인자 구조를 바꾸어 배포해 버리면, 상위 추론 서버 인프라(예: MLflow 내장 서빙, Seldon Core 등)는 컨테이너 가동 시 내부 함수 구격을 해석하지 못해 런타임 에러(`TypeError`)를 뿜으며 즉시 충돌이 발생합니다.
  2. **환경 일관성 보장**: 전처리 로직(`ScaledPredictor`)이 모델 파일과 별개로 돌아가면 학습 때와 서빙 때의 데이터 스케일링 기준이 달라져 엉뚱한 예측값(Data Drift)을 낳게 됩니다. 래퍼 가이드를 준수하여 전처리 함수 구조를 인프라 표준에 통합해야만 배포 환경이 바뀌더라도 무결한 서빙 정밀도를 유지할 수 있습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **인프라 인터페이스 서명(Signature) 검증의 중요성**: 에러가 발생했을 때 단순 코드 수정에 그치지 않고 왜 상위 클래스가 `(None, inputs.values)` 형태의 2개의 인자를 수용하도록 프로토콜을 강제했는지, 즉 MLOps 표준 API 규격을 역추적(RCA)하여 시스템 아키텍처의 의도를 파악하는 훈련을 완수했습니다.
