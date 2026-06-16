## 📅 Day 18: Version Datasets and Models Across Git Branches

### 1. Task

- **요구사항**: 데이터 및 모델의 버전 제어를 위해 Git 브랜치 전환에 따라 DVC가 관리하는 실제 물리 데이터셋이 유기적으로 동기화 및 복구되는지 검증합니다.

- **대상**: `data/raw/transactions.csv`, Git 태그(`v1.0`), 신규 브랜치(`v2-improved`)

- **목표**: 새 데이터셋으로 파이프라인을 학습시킨 후, 기존 `main` 브랜치로 체크아웃했을 때 최초 베이스라인 데이터셋(`v1.0`) 상태로 파일 시스템이 완벽히 롤백되는지 확인합니다.

---

### 2. Workflow

```text
 [main 브랜치 (v1.0 태그 지정)] ──> 데이터 해시: v1 상태 기록
               │
               ▼ (git checkout -b v2-improved)
 [v2-improved 브랜치]
   ├── 1. 신규 데이터셋 덮어쓰기 (transactions_v2.csv)
   ├── 2. DVC 추적 갱신 (dvc add) 및 파이프라인 재실행 (dvc repro)
   └── 3. 메타데이터 커밋 (git commit)
               │
               ▼ (git checkout main)
 [main 브랜치 복귀] ──> dvc checkout 실행 ──> 디스크 상의 데이터가 v1 상태로 자동 롤백 완료

```

---

### 3. 해결 과정 (Troubleshooting & Action)

#### 3-1. 베이스라인 태그 지정 및 실험 브랜치 분기

현재 `main` 브랜치의 안정적인 데이터 상태를 보존하기 위해 Git 태그를 생성하고, 고유 실험 환경을 위한 독립 브랜치를 가동했습니다.

```bash
# 현재 메인 상태를 버전 1.0으로 기록합니다.
git tag v1.0

# v2 데이터셋 실험을 위한 전용 공간(브랜치)을 생성하고 이동합니다.
git checkout -b v2-improved

```

#### 3-2. 데이터 업그레이드 및 파이프라인 재학습

새로운 데이터 파일을 기존 추적 경로에 덮어쓴 뒤, DVC 메타데이터를 갱신하고 전체 데이터 가공 및 학습 파이프라인을 다시 구동하여 커밋을 완료했습니다.

```bash
# 준비된 v2 데이터를 원본 파일명으로 덮어씁니다.
cp data/raw/transactions_v2.csv data/raw/transactions.csv

# DVC 시스템에 데이터 내용물이 변경되었음을 등록합니다.
dvc add data/raw/transactions.csv

# 변경된 데이터를 기반으로 파이프라인 전체를 재실행합니다.
dvc repro

# 변경된 DVC 메타데이터 파일(dvc.lock, .dvc)을 Git에 최종 기록합니다.
git add data/raw/transactions.csv.dvc dvc.lock
git commit -m "feat: upgrade dataset to v2 and rerun pipeline"

```

#### 3-3. 브랜치 복구 및 DVC 체크아웃을 통한 데이터 롤백 검증

과거 버전으로 안전하게 돌아갈 수 있는지 확인하기 위해 메인 브랜치로 복귀한 뒤, DVC 메타데이터 지침에 맞춰 실제 디스크 상의 데이터 파일들을 과거 해시값으로 강제 동기화했습니다.

```bash
# 다시 과거의 main 브랜치 공간으로 이동합니다.
git checkout main

# 중요: Git 브랜치에 맞춰 실제 대용량 데이터 파일 내용물도 이전 v1 상태로 롤백합니다.
dvc checkout

```

---

### 4. 무엇을 배웠는가 (Takeaway)

- **Git 브랜치와 DVC의 공조 체제**: Git은 설계도면(코드와 메타데이터 파일)을 바꾸는 역할을 하고, DVC는 그 도면에 적힌 일련번호(해시값)를 보고 창고에서 실제 거대한 자재(대용량 데이터 파일)를 꺼내와 조립하는 역할을 합니다. 둘이 팀워크를 이루기 때문에 개발자는 코드 브랜치만 바꿔도 대용량 데이터까지 자동으로 과거와 미래를 넘나들며 완벽히 동기화할 수 있습니다.

- **dvc checkout 명령어의 동작 원리**: 도서관에서 책을 빌릴 때 쓰는 '대출증'과 같은 원리입니다. Git 브랜치를 바꾸면 도서 대출증(메타데이터 .dvc 파일)에 적힌 책의 고유 ID가 바뀌게 됩니다. 이때 `dvc checkout`을 입력하면 시스템이 바뀐 대출증 번호를 확인하고, 진짜 책(물리 데이터 파일)을 서가에서 가져와 책상 위에 똑같이 올려놓아 주는 방식으로 작동합니다.

- **실무 관점의 의의**: 본 실습은 인공지능 모델의 '재현성(Reproducibility)'을 확보하는 핵심 프로세스입니다. 실제 운영 환경에서는 수개월 전 배포된 모델에 문제가 생겼을 때, 당시 사용했던 대용량 데이터와 코드를 똑같이 복구해 원인을 분석해야 합니다. Git과 DVC의 연동을 통해 대용량 스토리지를 낭비하지 않으면서도 완벽한 데이터 롤백 및 형상 관리가 가능함을 배웠습니다.
