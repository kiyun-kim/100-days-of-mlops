## 📅 Day 24: MLflow Autologging을 활용한 파이프라인 메타데이터 수집 자동화

### 1. Task

- **요구사항**: 데이터 과학자가 모델 학습 코드를 작성할 때 파라미터나 메트릭 로깅 코드를 수동으로 누적 기입하는 번거로움을 제거하고, 인프라 및 모델 라이브러리(Scikit-Learn) 레벨에서 실험 자산을 완전 자동 캡처(Autologging)하는 체계를 구축합니다.

- **목표**:
  1. `autolog-demo` 익스페리먼트 공간 자동 지정 및 격리
  2. `mlflow.sklearn.autolog()` 활성화를 통한 묵시적 파라미터 및 평가지표의 일괄 수집
  3. 훈련 완료된 바이너리 아티팩트 및 환경 디스크립터(`MLmodel`)의 자동 적재

---

### 2. Workflow

```text
+-----------------------------------------------------------------+
|               /root/code/autolog_experiment.py                  |
|                                                                 |
|   1. mlflow.set_experiment("autolog-demo")                      |
|   2. mlflow.sklearn.autolog()  <-- [자동 추적 레이어 활성화]    |
|   3. model.fit(X, y)           <-- [Scikit-Learn 내부 가로채기] |
+-----------------------------------------------------------------+
                               |
                               v (메타데이터 자동 패킹 및 전송)
+-----------------------------------------------------------------+
|                     MLflow Tracking Server                      |
|  - Run Name: likeable-pug-486                                   |
|  - Parameters: 14개 자동 바인딩 (C, penalty, solver 등)         |
|  - Metrics: 7개 자동 추적 (training_f1_score 등)                |
|  - Artifacts: model/ 디렉터리 내 바이너리 및 패키지 사양 보존   |
|    |                                                            |
|    +---> MLmodel (YAML 디스크립터)                              |
|    +---> conda.yaml, requirements.txt (의존성 사양)             |
+-----------------------------------------------------------------+

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 소스 코드 수정 및 후킹(Hooking) 활성화

`/root/code/autolog_experiment.py` 소스 파일에서 추적 대상 서버를 선언한 후, 학습 프레임워크가 실행되기 전에 익스페리먼트 컨텍스트와 자동 로깅을 정의했습니다.

```python
import mlflow
import mlflow.sklearn
from sklearn.linear_model import LogisticRegression
import numpy as np

mlflow.set_tracking_uri("http://localhost:5000")

# TODO 1: 활성 익스페리먼트를 'autolog-demo'로 지정하여 Default 영역 오염을 방지합니다.
mlflow.set_experiment("autolog-demo")

# TODO 2: Sklearn 플레버에 대한 자동 로깅을 활성화하여 .fit() 호출을 후킹합니다.
mlflow.sklearn.autolog()

# Toy 데이터셋 정의
X = np.array([[0, 0], [0, 1], [1, 0], [1, 1]])
y = np.array([0, 0, 1, 1])

# 모델 인스턴스화 및 학습 (이 시점에 Autolog가 동작하여 메타데이터를 전송합니다)
model = LogisticRegression(C=1.0, max_iter=100, random_state=42)
model.fit(X, y)

print("Autolog run complete — check the MLflow UI")

```

#### 3-2. 스크립트 가동 및 인프라 로깅 확인

터미널 환경에서 스크립트를 실행하여 신규 Run ID `c3a7ac7515914fe29ea12efe0f8f2800`로 이력이 등록되는 콘솔 로그를 확보했습니다.

```bash
root@controlplane ~/code via 🐍 v3.12.3
$ python3 /root/code/autolog_experiment.py

2026/06/22 10:31:40 INFO mlflow.tracking.fluent: Experiment with name 'autolog-demo' does not exist. Creating a new experiment.
2026/06/22 10:31:42 INFO mlflow.utils.autologging_utils: Created MLflow autologging run with ID 'c3a7ac7515914fe29ea12efe0f8f2800', which will track hyperparameters, performance metrics, model artifacts, and lineage information for the current sklearn workflow
2026/06/22 10:31:42 WARNING mlflow.models.model: Saving scikit-learn models in the pickle or cloudpickle format requires exercising caution...
⚡ View run likeable-pug-486 at: http://localhost:5000/#/experiments/1/runs/c3a7ac7515914fe29ea12efe0f8f2800
Autolog run complete — check the MLflow UI

```

---

### 4. 핵심 개념 정리

#### 4-1. MLflow Autologging

- **전문적 정의**: 머신러닝 프레임워크(Scikit-Learn 등)의 내부 학습 함수(`fit()`)를 래핑하여, 추가적인 코드 없이 모든 하이퍼파라미터, 메트릭, 학습 곡선, 출력 모델을 추적 서버에 전송하는 내장 기능입니다.

💡 **쉬운 비유**: **'자동차의 블랙박스 자동 녹화 모드'**. 운전자가 사고가 날 때마다 직접 캠코더를 켜서 수동으로 녹화(수동 로깅)할 필요 없이, 시동을 걸면(`autolog()`) 속도, 브레이크 깊이, 주변 영상(파라미터 및 아티팩트)이 블랙박스 메모리에 자동으로 상시 저장되는 시스템과 같습니다.

#### 4-2. MLmodel Descriptor

- **전문적 정의**: MLflow가 모델 아티팩트를 저장할 때 루트 디렉터리에 자동으로 생성하는 YAML 형식의 메타데이터 파일입니다. 이 파일에는 모델의 플레버(Flavors), 학습된 파이썬 버전, 의존성 패키지 정보(Conda/Pip 환경)가 선언되어 있습니다.

💡 **쉬운 비유**: **'밀키트 조리법 설명서'**. 가공된 고기(모델 바이너리 파일)만 있으면 이걸 어떻게 요리하고 어떤 냄비(파이썬 버전)를 써야 하는지 알 수 없습니다. MLmodel 파일은 "이 고기는 180°C 에어프라이어에 구워야 하고, 소스는 이런 것들이 필요합니다"라고 적어둔 설명서와 같습니다.

#### 4-3. 의존성 격리 및 환경 재현성 (Dependency Isolation)

- **전문적 정의**: 모델이 학습될 당시의 라이브러리 버전 사양(`requirements.txt`, `conda.yaml`)을 아티팩트 보관함에 함께 동기화하여, 시간이 흐른 뒤에도 동일한 예측 결과(Inference)를 낼 수 있도록 인프라 상태를 고정하는 개념입니다.

💡 **쉬운 비유**: **'타임캡슐에 넣는 재생 기기'**. 옛날 비디오테이프(모델)만 보관하면 나중에 비디오 플레이어(라이브러리 버전)가 없어서 재생을 못 합니다. 테이프를 묻을 때 당시의 비디오 플레이어와 연결 선(의존성 패키지)까지 타임캡슐에 같이 넣어두는 정밀함입니다.

#### 4-4. 실무 관점의 의의

- 프로덕션 MLOps 인프라 환경에서 수동 로깅 방식을 고수하면, 데이터 과학자가 실수로 특정 파라미터나 난수 시드(`random_state`)의 기록을 누락하는 휴먼 에러가 빈번하게 발생합니다. 엔지니어가 파이프라인의 엔트리포인트에 `autolog()` 정책을 적용하면 실험의 완벽한 재현성(Reproducibility)을 보장할 수 있습니다.

또한, 배포(CD) 단계에서 서빙 인프라(쿠버네티스 등)는 아티팩트의 `MLmodel` 파일과 의존성 명세를 읽고 자동으로 동일한 도커 컨테이너 환경을 빌드하므로, 인프라 버전 불일치로 인한 예측 에러 및 서비스 장애를 사전 차단할 수 있습니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **인프라 래핑 및 가로채기(Intercepting) 원리**: `mlflow.sklearn.autolog()` 소스코드가 프레임워크 내부 함수와 상호작용하여 데이터를 유실 없이 가로채는 소프트웨어 구조를 이해했습니다. 코드 분석 결과, 명시적 인자 외에 `dual=False`, `tol=0.0001` 등 총 14개의 기본 설정값까지 누락 없이 동적 바인딩되는 팩트를 확인했습니다.

- **선언 순서에 따른 레이어 격리**: 학습 파이프라인 엔진(`model.fit()`)이 시작되기 전에 `set_experiment()`와 `autolog()` 레이어가 먼저 선언되어 적재되어야만 올바른 타겟 목적지로 메타데이터 패킷이 라우팅된다는 라이프사이클 관리 역량을 배양했습니다.

- **배포 가능성(Deployability) 검증 시야**: 단순히 모델 파일(.pkl)만 확보하는 것은 반쪽짜리 인프라며, `MLmodel` 디스크립터와 의존성 구성 파일이 패킹되는 구조를 추적 서버에서 직접 육안으로 확인하며 서빙 파이프라인 연계 방안을 도출할 수 있게 되었습니다.
