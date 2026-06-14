## 📅 Day 16: Track ML Metrics with DVC

### 1. Task

- **요구사항**: 학습 파이프라인 실행 결과로 생성되는 `metrics.json` 파일을 DVC 메트릭으로 정식 등록하고 파이프라인을 재실행하여 시스템이 이를 인식하도록 설정한다.
- **대상**: `/root/code/fraud-detection/dvc.yaml`
- **목표**: `dvc metrics show` 명령어를 실행했을 때 `metrics.json` 내에 기록된 평가지표(`accuracy`, `f1_score`)가 터미널 화면에 정상적으로 표기되도록 연동한다.

---

### 2. Workflow

```text
[src/models/train.py]
         │
         ▼ (모델 학습 완료 후 평가지표 매회 자동 생성)
   [metrics.json]
         │
         ▼ (dvc.yaml 내 metrics 속성으로 감시 대상으로 지정)
┌────────────────────────────────────────────────────────┐
│ DVC Pipeline (train stage)                             │
│  - outs: models/model.pkl                              │
│  - metrics: metrics.json (cache: false)                │
└────────────────────────────────────────────────────────┘
         │
         ▼ (dvc repro 명령어로 파이프라인 갱신)
[dvc metrics show] ──> 터미널 화면에 accuracy, f1_score 점수 출력 확인

```

---

### 3. 해결 과정 (Troubleshooting & Action)

#### 3-1. DVC 환경 설정 파일 수정

`dvc.yaml` 파일의 `train` 스테이지 하단에 `metrics` 옵션을 추가하였다. 일반 출력 파일이 아닌 성적표(메트릭) 파일임을 시스템에 선언하였으며, Git으로 수치 변화의 이력을 직관적으로 추적할 수 있도록 DVC 캐시 저장 기능을 비활성화하였다.

```yaml
# /root/code/fraud-detection/dvc.yaml 파일 수정 내용
train:
  cmd: python src/models/train.py
  deps:
    - data/processed/train.csv
    - src/models/train.py
  outs:
    - models/model.pkl
  metrics:
    - metrics.json:
        cache: false
```

#### 3-2. 파이프라인 재실행 및 최종 검증

수정한 규칙을 DVC 파이프라인에 동기화하기 위해 재실행 명령을 내리고, 연동된 메트릭 값이 정상적으로 출력되는지 최종 검증을 수행하였다.

```bash
# 프로젝트 위치로 이동
cd /root/code/fraud-detection/

# 파이프라인을 다시 실행하여 변경된 규칙 적용
dvc repro

# 정식 등록된 메트릭 값 화면에 출력
dvc metrics show

```

---

### 4. 무엇을 배웠는가 (Takeaway)

- **DVC 메트릭 (Metrics)**: 학교에서 시험을 보면 나오는 '성적표'와 같은 개념이다. 인공지능 모델도 수학 문제를 풀고 나면 자기가 얼마나 잘 맞혔는지 점수가 나오는데, DVC 시스템에 "이 파일은 일반 파일이 아니라 인공지능의 중요 성적표다!"라고 명시해 주는 역할을 한다. 이렇게 지정해 두면 코드를 수정할 때마다 모델 성능이 올랐는지 떨어졌는지 한눈에 쉽게 비교할 수 있다.

- **캐시 비활성화 (cache: false)**: 거대하고 무거운 영상이나 인공지능 덩어리 파일은 DVC 전용 대형 창고에 따로 보관해야 하지만, 텍스트로 적힌 몇 줄짜리 가벼운 성적표 정보는 굳이 창고에 숨겨둘 필요가 없다. `cache: false` 설정을 부여하면 이 성적표 데이터를 일반 코드처럼 Git이라는 공용 게시판에 그대로 올리게 된다. 덕분에 과거의 성적과 현재의 성적이 어떻게 달라졌는지 글자 비교하듯 직관적으로 편하게 추적할 수 있다.
