## 📅 Day 17: Run and Compare DVC Experiments

### 1. Task

- **요구사항**: 하이퍼파라미터 `n_estimators` 값을 다양하게 변경하며 총 3번의 DVC 실험(Experiment)을 수행하고, 성능이 가장 우수한 모델을 메인 작업 공간(Workspace)에 반영한다.

- **대상**: `params.yaml`, `dvc.yaml` 및 학습 파이프라인

- **목표**: `f1_score` 지표가 가장 높은 실험의 데이터(`n_estimators`, `metrics.json`, `models/model.pkl`)를 추적 대상으로 승격(Promote)시킨다.

---

### 2. Workflow

```text
[Baseline: n_estimators=100]
         │
         ├──> [Exp 1: n_estimators=50]  ──> f1_score: 0.8975
         ├──> [Exp 2: n_estimators=200] ──> f1_score: 0.9200 (최적 모델 🏆)
         └──> [Exp 3: n_estimators=500] ──> f1_score: 0.8300
         │
         ▼ (성능 비교 후 dvc exp apply 실행)
[Workspace Update] ──> `caf2307` 실험 상태로 파일 자동 동기화 및 런타임 검증 완료

```

---

### 3. 해결 과정 (Troubleshooting & Action)

#### 3-1. 하이퍼파라미터 변경 및 실험 수행

`params.yaml` 구조에 맞게 명시적 파라미터 재정의 옵션(`-S`)을 사용하여 하이퍼파라미터를 변경해가며 총 3회 독립적인 실험을 트리거하였다.

```bash
# n_estimators를 50, 200, 500으로 각각 설정하여 실험 파이프라인 가동
dvc exp run -S n_estimators=50
dvc exp run -S n_estimators=200
dvc exp run -S n_estimators=500

```

#### 3-2. 실험 결과 비교 및 최적 모델 적용

생성된 임시 실험들의 메트릭을 테이블 형태로 대조하여 가장 높은 `f1_score`를 기록한 실험 ID(`caf2307`)를 식별하고, 이를 메인 작업 공간에 최종 적용하였다.

```bash
# 전체 실험 리스트와 평가지표(f1_score) 출력 및 대조
dvc exp show

# f1_score가 0.92로 가장 높은 'caf2307' 실험의 상태를 워크스페이스에 반영
dvc exp apply caf2307

```

---

### 4. 무엇을 배웠는가 (Takeaway)

- **하이퍼파라미터 튜닝의 목적**: 인공지능 모델이 더 똑똑하게 문제를 풀 수 있도록 내부 조절 나사(`n_estimators`)를 요리조리 돌려보는 과정이다. 무작정 규칙을 많이 만든다고 성능이 무조건 좋아지는 것이 아니기 때문에, 최적의 나사 조임 정도를 찾는 실험이 필수적이다.

- **DVC Experiments**: 과학 시간에 실험 노트를 쓰듯이, 인공지능 모델의 설정값을 바꿀 때마다 그 결과를 임시 보관함에 깔끔하게 기록해 두는 기능이다. 기존 코드를 더럽히거나 일일이 백업본 파일을 만들지 않고도 여러 버전의 실험을 안전하게 시도할 수 있게 해 준다.

- **명령어 역할 정리**:
- `dvc exp run -S`: 원래 코드나 파일을 직접 수정하지 않고, "이번 한 번만 이 조절 나사 값으로 시험 삼아 가동해 봐!"라고 임시 명령을 내리는 스위치다.
- `dvc exp show`: 지금까지 테스트했던 임시 성적표들을 한자리에 모아서 한눈에 비교해 주는 종합 게시판이다.
- `dvc exp apply`: 수많은 테스트용 임시 성적표 중 "이 2등 제품이 제일 마음에 드니 정식 제품으로 채택하겠다!" 하고 메인 작업실로 결과물을 가져와 정식 등록하는 명령어다.
