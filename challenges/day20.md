## 📅 Day 20: Install and Start the MLflow Tracking Server

### 1. 이번 미션의 핵심 요약 (Task)

- **요구사항**: xFusionCorp Industries ML 팀의 실험 이력 관리를 위해, 가이드 라인에 맞춘 MLflow 로컬 추적 서버를 인프라에 구축합니다.

- **목표**: 터미널을 닫아도 죽지 않는 백그라운드 형태로 서버를 가동하고, 프록시 환경에서도 웹 대시보드(UI)가 정상적으로 열리도록 네트워크 옵션을 튜닝합니다.

---

### 2. 한눈에 보는 인프라 아키텍처 (Architecture)

```text
  [ 외부 웹 브라우저 (MLflow UI 버튼) ]
                │
                ▼ (CORS / 모든 호스트 헤더 통과 권한부여)
        [ 5000번 포트 활성화 ]
                │
         mlflow server 가동
         (nohup 백그라운드 유지)
        ┌───────┴───────┐
        ▼               ▼
[SQLite 데이터베이스]  [아티팩트 보관소]
mlflow.db 파일 저장    모델 파일 및 로그 보관
(backend-store)       (default-artifact-root)

```

---

### 3. 주요 수행 절차 및 명령어

#### 3-1. 사전 자재 창고 및 데이터베이스 폴더 생성

MLflow 서버는 기동될 때 저장할 폴더가 없으면 에러를 뿜으며 즉시 중단(Abort)됩니다. 이를 방지하기 위해 백엔드 DB 폴더와 아티팩트 보관 폴더를 먼저 생성했습니다.

```bash
mkdir -p /root/code/mlflow-backend
mkdir -p /root/code/mlflow-artifacts

```

#### 3-2. 모든 제약 조건을 반영한 MLflow 서버 가동

포트 번호(5000), SQLite 절대 경로 바인딩, 그리고 실습실 프록시 서버의 접근 거부를 방지하기 위한 오리진 및 호스트 헤더 전면 개방 옵션(`*`)을 적용하여 백그라운드로 실행했습니다.

```bash
nohup mlflow server \
  --host 0.0.0.0 \
  --port 5000 \
  --backend-store-uri sqlite:////root/code/mlflow-backend/mlflow.db \
  --default-artifact-root /root/code/mlflow-artifacts/ \
  --cors-allowed-origins '*' \
  --allowed-hosts '*' > /root/code/mlflow-backend/mlflow.log 2>&1 &

```

#### 3-3. 인프라 가동태세 최종 검증

서버가 켜진 후 데이터베이스 파일이 제대로 안착했는지, 5000번 포트 배관이 정상적으로 열렸는지 확인하여 가동 성공을 확정했습니다.

```bash
# 1. mlflow.db 파일 생성 확인
ls -la /root/code/mlflow-backend/

# 2. 5000번 포트 프로세스 바인딩 상태 확인
ss -ntlp | grep 5000

```

---

### 4. 무엇을 배웠는가 (Takeaway)

- **MLflow는 실험용 '블랙박스/다이어리'다**: 연구원들이 매번 인공지능을 학습시킬 때마다 "이번엔 하이퍼파라미터를 뭘 썼더라?", "정확도가 몇이었지?" 하고 수첩에 받아 적는 것은 한계가 있습니다. MLflow는 모델이 학습될 때의 모든 기록과 완성된 파일들을 한곳에 자동으로 적어주는 '스마트 다이어리 서버' 역할을 합니다.

- **sqlite://// 슬래시가 4개인 이유**: 데이터베이스 주소를 적을 때 슬래시 기호 개수가 헷갈릴 수 있습니다. 앞에 `sqlite:///`까지는 "이 시스템은 SQLite 엔진을 쓸 거야"라는 가이드 규격(프로토콜)이고, 마지막 4번째 `/`는 리눅스 컴퓨터의 최상위 뿌리 폴더인 **루트(/) 디렉토리**를 뜻하는 절대 경로의 시작점입니다. 문법 기호와 실제 주소 기호가 합쳐져 4개가 된 것입니다.

- **nohup과 &의 콤비네이션**: 명령어 뒤에 `&`만 붙이면 백그라운드로 실행되긴 하지만, 내가 터미널 창을 닫거나 컴퓨터 접속을 끊으면 프로그램도 같이 죽어버립니다. 앞에 `nohup`(No Hang Up)을 붙여주면 "내가 퇴근해서 터미널을 끄더라도 이 서버는 절대 끄지 말고 24시간 계속 돌려라"라는 명령이 되어 안정적인 서비스 운영이 가능해집니다.
