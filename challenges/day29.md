## 📅 Day 29: Configure MLflow with Remote Tracking Server and Artifact Store

### 1. Task

- **요구사항**: PostgreSQL(메타데이터 DB)과 S3 호환 자체구축형 스토리지인 SeaweedFS(아티팩트 대피소)가 분리된 고가용성 MLOps 토폴로지 환경에서, 스모크 테스트 가동 시 객체 스토리지로 파일이 전송되지 않는 인증 및 엔드포인트 연동 결함을 찾아 정정하고 파일 원격 적재 환경을 수립합니다.

- **목표**:
  1. `/root/code/start-mlflow.sh` 인프라 가동 스크립트 내 누락된 S3 인증 및 타겟 URL 환경 변수 확보
  2. MLflow 데몬의 재기동(`restart-mlflow.sh`) 및 포트 포워딩 레이어 정렬 검증
  3. `log_test_run.py` 실행을 통한 `test-remote` 익스페리먼트 내 아티팩트(`MLmodel`, `model.pkl`) 업로드 완수

---

### 2. Workflow

```text
[ MLflow Client: log_test_run.py ]
                │
                ├─── 메타데이터 전송 ───> [ PostgreSQL (Port 5432) ] ✔️ 정상 가동
                │
                └─── 아티팩트 업로드 ───> [ MLflow Server (Port 5000) ]
                                                │
                                                v (정정된 S3 환경 변수 라우팅)
                                          [ SeaweedFS S3 API (Port 8333) ]
                                                │
                                                v (버킷 내 적재)
                                          /buckets/mlflow-artifacts/

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 초기 통신 예외 모니터링

초기 인프라 상태에서 `python3 /root/code/log_test_run.py` 스모크 테스트를 가동했을 때, 메타데이터는 수집되나 물리적인 바이너리 파일 업로드 단계에서 AWS S3 자격 증명 획득 실패 혹은 엔드포인트 도달 불능(`EndpointConnectionError` 또는 인증 실패 예외) 로그가 관측되었습니다.

#### 3-2. 가동 데몬 스크립트 내부 환경 변수 교정

서버 프로세스가 로컬의 S3 호환 API 규격(SeaweedFS)을 인지할 수 있도록 `/root/code/start-mlflow.sh` 자산 내부에 명세서 지정 계정과 타겟 포트를 명시적으로 주입했습니다.

```bash
# /root/code/start-mlflow.sh 상단에 선언 및 반영한 핵심 환경 변수
export AWS_ACCESS_KEY_ID="weedadmin"
export AWS_SECRET_ACCESS_KEY="weedadmin123"
export MLFLOW_S3_ENDPOINT_URL="http://localhost:8333" # UI 포트(8888)가 아닌 S3 API 전용 포트 지정
export MLFLOW_S3_IGNORE_TLS="true"                    # 내부 비인증 보안 레이어 바이패스 정책 활성화

```

#### 3-3. 서비스 데몬 재가동 및 파이프라인 최종 실증

수정된 런타임 환경 변수가 메모리에 로드되도록 제어 스크립트를 구동하고, 연계 작동 상태를 최종 검증했습니다.

```bash
# 1. MLflow 인프라 백그라운드 프로세스 재생성 및 바인딩
bash /root/code/restart-mlflow.sh

# 2. 스모크 테스트 재가동 및 엔드투엔드 파이프라인 가동
root@controlplane ~
$ python3 /root/code/log_test_run.py
Successfully logged run to remote MLflow server with tracking URI: http://localhost:5000

```

- **결과 검증**: MLflow UI에서 `test-remote` Experiment 내에 성공 Run이 등록되고, SeaweedFS Filer 스토리지 웹 관리 콘솔에 접속하여 `/buckets/mlflow-artifacts/` 하위 경로에 `MLmodel` 및 학습 바이너리인 `model.pkl` 파일이 유실 없이 물리적 오브젝트로 동기화되었음을 확인했습니다.

---

### 4. 핵심 개념 정리

#### 4-1. MLflow Backend Store vs Artifact Store (정보장부와 비밀창고)

- **정의**: MLflow는 아키텍처 효율화를 위해 하이퍼파라미터/메트릭 같은 텍스트성 메타데이터를 저장하는 백엔드 스토어(PostgreSQL 등)와 무거운 모델 바이너리 파일을 저장하는 아티팩트 스토어(S3, SeaweedFS 등)를 이원화하여 관리합니다.

  💡 **'은행의 거래 장부와 진짜 금고'**. 우리가 은행(MLflow)에 가서 돈을 맡기면, 은행원은 컴퓨터 화면 장부(PostgreSQL)에 "홍길동님 500원 입금함"이라고 글씨를 적습니다. 그리고 실제 500원짜리 동전(모델 파일)은 저기 뒤에 있는 커다란 강철 철갑 금고(SeaweedFS 스토리님)에 따로 집어넣습니다. 오늘 실습은 장부(DB)에 글씨는 써지는데, 저기 뒤에 있는 강철 금고(스토리지) 자물쇠 비밀번호와 금고가 어디 방에 있는지 주소를 몰라서 돈을 못 집어넣고 마당에 떨어뜨리던 상황을 고친 것입니다.

#### 4-2. S3 호환 엔드포인트 라우팅 (S3 Compatible Endpoint)

- **정의**: AWS S3 인프라가 아닌 오픈소스 대용량 분산 파일 시스템(SeaweedFS, MinIO 등)을 구축한 경우, MLflow 내부의 AWS SDK 표준 통신 대상 주소를 로컬 내부 주소(`MLFLOW_S3_ENDPOINT_URL`)로 강제 재라우팅하는 설정 기법입니다.
  💡 **'편지 배달 주소 강제로 바꾸기'**. 컴퓨터 내부에는 편지를 보내는 기본 우체부(AWS SDK)가 살고 있습니다. 이 우체부는 아무 말도 안 하면 편지(아티팩트)를 무조건 미국에 있는 '진짜 AWS S3 본사'로 배달하려고 비행기를 탑니다. 그래서 우체부 가방에 "미국 가지 말고, 우리 동네 8333번지 방(SeaweedFS)으로 배달해!"라고 적힌 이정표 주소지(`MLFLOW_S3_ENDPOINT_URL`)를 강력하게 붙여준 것입니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **분산 인프라 토폴로지의 상호 의존성 진단 역량**: 분산 아키텍처 환경에서는 일부 파이프라인(메타데이터 기록)이 성공하더라도 다른 타겟 스토리지 레이어(객체 스토리지 업로드)의 환경 변수 및 포트 매핑 규칙이 어긋나면 전체 시스템 파이프라인이 마비될 수 있다는 다각적 분석 시야를 확보했습니다.

- **포트 서명(Port Signature) 분리 구별 역량**: 스토리지 제어 시 단순 웹 UI 뷰포트 관리용 포트(8888)와 실제 데이터 패킷을 수입하는 API 엔드포인트 포트(8333)의 용도를 인프라 레벨에서 명확히 격리하고 선언해야 통신 교착 상태를 예방할 수 있다는 아키텍처 교훈을 얻었습니다.
