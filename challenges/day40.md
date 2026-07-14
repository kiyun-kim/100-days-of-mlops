## 📅 Day 40: End-to-End 사기 탐지(Fraud Detection) 파이프라인 통합 및 레지스트리 자동화

### 1. Task

- **요구사항**: xFusionCorp Industries의 ML 플랫폼 팀은 데이터 검증부터 하이퍼파라미터 튜닝, 모델 선택, 레지스트리 등록, 최종 리포트 생성까지 이어지는 5단계의 훈련 파이프라인을 구축했습니다. 하지만 각 스크립트 간의 연결(Wiring) 오류와 미구현된 `# TODO` 블록으로 인해 `make train-pipeline` 명령어가 끝까지 실행되지 못하고 있습니다.

- **목표**:
  1. `Makefile`의 타겟 실행 순서를 교정하여 의존성 충돌 해결
  2. `src/select_model.py`의 잘못된 평가 지표 매핑 수정
  3. `src/register.py`에 선택된 모델을 MLflow 레지스트리에 등록하고 `staging` 별칭(Alias)을 부여하는 로직 추가
  4. `src/report.py`에서 상류 단계(Upstream)의 모든 산출물을 하나의 최종 JSON 리포트로 병합하는 스키마 완성

---

### 2. Workflow

```text
[make train-pipeline] (단일 명령어로 5단계 자동 실행)
  │
  ├── 1. validate_data.py ──> (스키마/결측치 검증) ──> reports/validation_status.json
  │
  ├── 2. tune.py ───────────> (Optuna 튜닝) ──> MLflow Run 5개 이상 적재 (f1_score)
  │
  ├── 3. select_model.py ───> (최우수 모델 선정) ──> reports/selection.json
  │
  ├── 4. register.py ───────> (MLflow Model Registry 등록) ──> 'fraud-detector' (Alias: 'staging')
  │
  └── 5. report.py ─────────> (모든 산출물 통합)
         │
         ▼
[End State: Deployment Ready]
  └── reports/training_report.json (최종 배포 심사용 메타데이터 리포트 생성 완료)

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. `Makefile` 파이프라인 실행 순서(의존성) 교정

모델 튜닝(Optuna)이 완료되지도 않았는데 모델 선택 스크립트가 먼저 실행되어 `no runs in experiment` 에러가 발생하는 뼈아픈 순서 버그를 수정했습니다.

```makefile
# Makefile (수정 후)
train-pipeline:
	python3 src/validate_data.py
	python3 src/tune.py          # 튜닝이 먼저 실행되도록 순서 교정
	python3 src/select_model.py  # 튜닝 결과를 바탕으로 모델 선택
	python3 src/register.py
	python3 src/report.py

```

#### 3-2. `select_model.py` 평가 지표(Metric) 연결 버그 교정

튜닝 파이프라인에서는 `f1_score`를 기록했는데, 정작 모델을 선택할 때는 존재하지 않는 `accuracy`를 기준으로 정렬하고 필터링하는 치명적인 미스매치를 바로잡았습니다.

```python
# src/select_model.py (수정 후)
runs = mlflow.search_runs(
    experiment_ids=[exp.experiment_id],
    order_by=["metrics.f1_score DESC"], # accuracy -> f1_score로 교정
    max_results=200,
)
# ... 중략 ...
score = float(best["metrics.f1_score"]) # accuracy -> f1_score로 교정

```

#### 3-3. `register.py` 모델 레지스트리 별칭(Alias) 할당 로직 구현

선택된 모델이 서빙(Serving) 레이어에서 이름으로 쉽게 호출될 수 있도록, 모델 등록 직후 `staging`이라는 릴리즈 레인(Release-lane) 별칭을 부여하는 로직을 완성했습니다.

```python
# src/register.py (TODO 구현)
model_uri = f"runs:/{selection['run_id']}/model"
version = mlflow.register_model(model_uri, REGISTERED_MODEL_NAME)

# 서빙 레이어에서 'staging' 모델을 바라보도록 Alias 매핑
client.set_registered_model_alias(REGISTERED_MODEL_NAME, RELEASE_ALIAS, version.version)

```

#### 3-4. `report.py` 최종 리포트 병합 로직 구현

각 단계에서 흩어져 있던 메타데이터들(결과 상태, 튜닝 횟수, 모델 종류 등)을 배포 체크리스트 규격에 맞게 하나의 딕셔너리로 조립하여 저장하는 로직을 완성했습니다.

```python
# src/report.py (TODO 구현)
report = {
    "best_model": selection["model_type"],
    "best_params": best_params,
    "metrics": best_metrics,
    "total_trials": total_trials,
    "validation_status": validation["status"]
}

```

---

### 4. 핵심 개념 정리

- **MLflow Model Registry**: 학습이 완료된 모델들을 한곳에 모아 관리하는 '모델 저장소'입니다. 버전 관리(v1, v2...)는 물론, `staging`, `production`과 같은 별칭(Alias)을 통해 현재 서비스에 어떤 모델이 배포되어야 하는지 명확한 상태를 통제할 수 있습니다.
- **Pipeline Orchestration (`make`)**: 복잡한 머신러닝 파이프라인을 구성하는 여러 개의 스크립트를 올바른 의존성 순서대로 묶어, 단 한 줄의 명령어(`make train-pipeline`)로 멱등성(Idempotency) 있게 실행되도록 만드는 자동화 기법입니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 각 스크립트 파일이 개별적으로 완벽하게 동작하더라도, 이들을 하나로 엮는 과정에서 '평가 지표 이름 불일치', '실행 순서 뒤바뀜' 같은 사소한 틈새로 인해 전체 시스템이 붕괴될 수 있음을 생생하게 확인했습니다. 머신러닝 시스템은 단일 스크립트의 화려함보다 각 컴포넌트 간의 매끄러운 톱니바퀴 결합(Wiring)이 훨씬 중요합니다.

- **Pain Point**: 실무에서 팀원들이 튜닝, 평가, 배포 스크립트를 각자 나눠서 개발한 뒤 통합할 때 가장 많이 발생하는 장애가 바로 이번 미션의 에러들입니다. 모델 레지스트리의 Alias 관리가 안 되면 개발 서버에 배포되어야 할 미검증 모델이 프로덕션(Production)으로 흘러 들어가는 대형 사고가 터집니다.

- **성장 포인트**: 이제 단순한 '모델러'를 넘어섰습니다. 데이터 검증부터 최종 리포트 출력, 그리고 모델을 레지스트리에 올려 서빙 레이어와 연결하는 전 과정을 End-to-End로 관장하는 'ML 파이프라인 아키텍트'의 시야를 갖추게 되었습니다. 자동화된 파이프라인을 통해 "내 로컬 컴퓨터에서는 되는데 서버에서는 안 돼요"라는 변명을 원천 차단하는 견고한 인프라를 설계할 수 있습니다.
