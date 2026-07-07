## 📅 Day 37: 4단계 머신러닝 파이프라인 데이터 체인(Wiring) 교정 및 통합 로깅

### 1. Task

- **요구사항**: xFusionCorp Industries의 ML 플랫폼 팀은 전처리(Preprocess), 특징 추출(Featurize), 학습(Train), 평가(Evaluate)로 이어지는 4단계 파이프라인을 단일 MLflow 런(Run)으로 오케스트레이션합니다. 현재 특징 추출 단계가 전처리된 데이터가 아닌 원본 데이터를 읽어와 파이프라인 체인이 끊어진 상태이므로, 이를 교정하여 무결성을 확보해야 합니다.

- **목표**:
  1. `src/featurize.py` 스크립트의 입력 데이터 참조 경로를 원본이 아닌 직전 단계(Preprocess)의 산출물 경로로 교정
  2. `data/features/features.csv` 파일과 `data/processed/train_clean.csv` 파일의 행(Row) 수가 동일하도록(193행) 동기화
  3. `run_pipeline.py`를 실행하여 4단계 과정 전체가 MLflow `training-pipeline` 실험 내 단일 런(Run)으로 통합 기록되도록 구성

---

### 2. Workflow

```text
[Data Source]
  └── data/raw/train.csv (200 rows)

[Pipeline Stages: run_pipeline.py]
  ├── 1. Preprocess (src/preprocess.py)
  │      └── 산출물: data/processed/train_clean.csv (192 rows, 이상치/중복 제거)
  │
  ├── 2. Featurize (src/featurize.py) ──> [Fix: 'processed_path' 참조로 교정]
  │      └── 산출물: data/features/features.csv (192 rows, 파생 변수 추가)
  │
  ├── 3. Train (src/train.py)
  │      └── 산출물: models/model.pkl (가중치 학습)
  │
  └── 4. Evaluate (src/evaluate.py)
         └── 산출물: reports/evaluation.json (accuracy, f1, roc_auc 등 수치형 지표 산출)

[End State: MLflow]
  └── 단 1개의 통합된 Run 기록 생성 (전체 파이프라인 파라미터 및 메트릭 일괄 적재 완료)

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 에러 지점 식별 및 데이터 체인 분석

파이프라인 실행 로그를 확인한 결과, `featurize` 단계에서 불러온 데이터의 행(Row) 수가 원본(200행)과 동일하게 유지되는 것을 발견했습니다. 이는 이전 단계인 전처리(`preprocess.py`)에서 8개의 행이 정상적으로 드롭(Drop)되었음에도 불구하고, 그 산출물이 다음 단계로 전혀 전달되지 않았음을 의미합니다.

#### 3-2. 입력 경로 체인(Input Wiring) 복구

`src/featurize.py` 스크립트를 열어, 데이터를 불러오는 참조 키를 원본 데이터(`raw_path`)에서 전처리 산출물(`processed_path`)로 교정했습니다.

```python
# src/featurize.py (Line 24 부근)

# [수정 전]: 원본 데이터를 그대로 읽어와 전처리(Drop/Clean) 내역이 무시됨
# input_path = config["data"]["raw_path"]

# [수정 후]: 이전 단계의 산출물 경로(processed_path)를 읽어오도록 파이프라인 체인 복구
input_path = config["data"]["processed_path"]

```

#### 3-3. 파이프라인 통합 실행 및 검증

수정된 코드를 저장한 후 `run_pipeline.py` 오케스트레이터 스크립트를 실행하고, 리눅스 명령어(`wc -l`)를 통해 단계별 산출물의 행 수가 193(헤더 1줄 포함, 데이터 192행)으로 정확히 동기화되었는지 검증했습니다.

```bash
# 파이프라인 오케스트레이터 실행
python3 run_pipeline.py

# 산출물 간 행 수(Row count) 동기화 검증
wc -l data/processed/train_clean.csv
# 출력: 193 data/processed/train_clean.csv

wc -l data/features/features.csv
# 출력: 193 data/features/features.csv

```

---

### 4. 핵심 개념 정리

- **ML Pipeline Orchestration (파이프라인 오케스트레이션)**: 여러 단계로 나뉜 머신러닝 작업(데이터 추출, 전처리, 학습, 평가)을 사람이 수동으로 하나씩 실행하는 대신, 정해진 순서와 의존성 규칙에 따라 자동으로 실행되도록 조율하는 과정입니다.

> 💡 비유하자면, 공장의 **'컨베이어 벨트 시스템'**과 같습니다. 1번 작업자가 부품을 깎고 나면 2번 작업자가 그 부품을 받아 조립해야 합니다. 그런데 2번 작업자가 자꾸 가공되지 않은 쇳덩이를 새로 가져와 조립하는 실수를 저지른다면 불량품이 나오게 됩니다. 파이프라인 체인을 복구하는 것은 이 컨베이어 벨트가 올바른 순서대로 흘러가도록 레일을 다시 연결해 주는 작업입니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 개별 스크립트(`train.py`, `evaluate.py` 등) 내부 로직이 제아무리 완벽하게 짜여 있어도, 서로 데이터를 주고받는 '연결 고리(Path)' 하나가 어긋나면 전체 파이프라인이 무용지물이 된다는 사실을 뼈저리게 느꼈습니다. 쓰레기가 들어가면 쓰레기가 나온다는 'GIGO(Garbage In, Garbage Out)' 원칙처럼, 전처리가 무시된 오염된 데이터로 모델 학습을 돌리는 아찔한 상황이었습니다.

- **Pain Point**: 현업에서 파이프라인의 데이터 전달 경로를 하드코딩하거나 수동으로 관리하면, 장애가 발생했을 때 어느 단계에서 데이터가 오염되었는지 역추적하기가 불가능에 가깝습니다. 만약 오늘 이 에러를 잡지 못하고 프로덕션에 넘어갔다면, 데이터 엔지니어링 팀이 기껏 밤새워 정제한 데이터를 모델링 팀에서 허공에 날려버리는 최악의 부서 간 사일로(Silo) 문제가 발생했을 것입니다.

- **성장 포인트**: `pipeline_config.yaml`과 같은 중앙 집중형 설정 파일을 통해 입출력 경로를 한곳에서 관리하고, 오케스트레이터 스크립트 하나로 모든 생애주기를 통제하는 아키텍처의 중요성을 뼈저리게 배웠습니다. 앞으로 모듈식 개발을 진행할 때 "내 스크립트는 잘 돌아가"에서 멈추는 것이 아니라, "내 출력물이 다음 단계 파이프라인의 입력물과 100% 무결하게 호환되는가"를 가장 먼저 검증하는 시스템적 시야를 확보했습니다.
