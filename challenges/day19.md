## 📅 Day 19: Build Complete DVC ML Pipeline with Remote Storage and Experiments

### 1. Task

- **요구사항**: 기존 `dvc.yaml` 파이프라인에 숨겨져 있는 잘못된 출력 경로 오류를 찾아 올바르게 수정하고, 미완성된 모델 학습(`train`)과 평가(`evaluate`) 단계를 새로 설계하여 전체 자동화 공정을 완성합니다.

- **목표**: 전체 파이프라인을 성공적으로 구동하여 생성된 최종 결과물들을 클라우드 원격 저장소(`SeaweedFS`)에 안전하게 백업하고, 프로젝트의 현재 상태를 `v1.0` 정식 출시 버전으로 태깅하여 영구 기록합니다

---

### 2. 한눈에 보는 데이터 공장 흐름도 (Workflow)

```text
[1. ingest]      원본 데이터(data.csv) 확인 및 가져오기
     │
     ▼
[2. validate]    데이터가 깨지지 않았는지 검사서(validation.json) 발행
     │
     ▼
[3. preprocess]  ★오류 수정: 흙먼지를 털어내고 'clean.csv'로 저장
     │
     ▼
[4. train]       [신규 추가] 정제된 'clean.csv'를 입력받아 모델(model.pkl) 학습 및 테스트셋 추출
     │
     ▼
[5. evaluate]    [신규 추가] 최종 모델의 수능 성적표(evaluation.json) 발행
     │
     ▼
[DVC PUSH]       완성된 모든 데이터/모델 자재를 'SeaweedFS 클라우드 창고'로 트럭 배송 및 백업 완료!

```

---

### 3. 주요 수행 절차 및 자동화 명령어

#### 3-1. 환경 싱크 및 소스 코드 최적화

설계도면(`dvc.yaml`)과 실제 연산을 수행하는 파이썬 소스 코드(`preprocess.py`)의 입출력 배관이 어긋나면 파이프라인이 즉시 가동을 멈춥니다. 인프라가 요구하는 확정적인 데이터 연결 고리인 **`data/processed/clean.csv`** 경로에 맞춰 모든 싱크를 완벽히 일치시켰습니다.

1. 준비된 학습/평가 스크립트들을 정식 위치로 복사했습니다.

```bash
cp scripts-staging/train.py scripts/
cp scripts-staging/evaluate.py scripts/

```

2. `dvc.yaml`을 수정하여 `preprocess` 스테이지의 아웃풋 경로와 `train` 스테이지의 디펜던시 입력 경로를 모두 `data/processed/clean.csv`로 연결하고, 요구사항에 맞게 `train`과 `evaluate` 스테이지를 하단에 새로 설계했습니다.
3. `scripts/preprocess.py` 소스 코드를 열어, 전처리된 데이터가 저장되는 실제 물리 경로도 똑같이 `data/processed/clean.csv`로 수정하여 데이터 흐름을 동기화했습니다.

#### 3-2. 파이프라인 가동 및 클라우드 백업

모든 파일의 배관 연결이 끝난 후, 터미널에 아래 명령어들을 내려 전체 공정을 가동하고 원격 창고에 안전하게 백업했습니다.

```bash
# 1. 완벽히 연결된 전체 자동화 파이프라인 가동
dvc repro

# 2. 생성된 실제 무거운 자재들을 SeaweedFS 원격 저장소 버킷으로 업로드
dvc push

# 3. 완성된 모든 설계도면과 소스코드를 Git에 기록하고 v1.0 도장 쾅!
git add dvc.yaml dvc.lock scripts/
git commit -m "feat: complete end-to-end ml production pipeline"
git tag v1.0

```

---

### 4. 무엇을 배웠는가 (Takeaway)

- **데이터 파이프라인은 '도미노 게임'이다**: 인공지능 공장은 앞사람이 물건을 만들어 던져주면 뒷사람이 그걸 받아서 가공하는 도미노 게임과 같습니다. 앞사람은 `clean.csv`라는 이름으로 던졌는데 뒷사람은 다른 이름만 기다리고 있으면 전체 공정이 마비됩니다. MLOps에서는 이렇게 단계와 단계 사이의 '연결 경로(I/O 마찰)'를 자석처럼 딱 맞추는 것이 핵심입니다.

- **DVC Push의 실무적 가치**: 우리가 만든 인공지능 모델 파일은 용량이 엄청나게 큽니다. 회사에서 이 무거운 파일들을 카카오톡이나 일반 이메일로 주고받으면 컴퓨터가 터지겠지요? `dvc push` 명령어를 쓰면 코드(도면)는 깃허브에 가볍게 올리고, 실제 무거운 인공지능 덩어리는 안전한 회사 클라우드 비밀 창고(`SeaweedFS`)로 자동 배송해 주기 때문에 대규모 협업이 가능해집니다.
