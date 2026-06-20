## 📅 Day 22: Create and Organize MLflow Experiments

### 1. Task

- **요구사항**: 복수의 팀과 머신러닝 프로젝트가 단일 MLflow 인프라를 공유할 때, 실험 데이터가 뒤섞이거나 오염되지 않도록 격리된 프로젝트별 익스페리먼트 공간을 생성하고 관리 주체를 명시하기 위한 메타데이터 관리 체계를 수립합니다.

- **목표**:
  - 신규 프로젝트 공간인 `fraud-detection` 익스페리먼트 생성 및 전용 태그(`team: ml-platform`) 바인딩
  - 신규 프로젝트 공간인 `churn-prediction` 익스페리먼트 생성 및 전용 태그(`team: analytics`) 바인딩
  - 기존 운영 중인 `legacy-models` 및 `Default` 레코드를 보존하여 백워드 호환성 유지

---

### 2. Workflow

```text
  [ MLflow Web UI Dashboard (Port 5000) ]
                     |
       +-------------+-------------+
       |                           |
       v [Create Experiment]       v [Create Experiment]
+--------------------------+ +--------------------------+
| Name: fraud-detection    | | Name: churn-prediction   |
|                          | |                          |
| [Description]            | | [Description]            |
| "Fraud detection..."     | | (Optional / None)        |
|                          | |                          |
| [Tags]                   | | [Tags]                   |
| - Key: team              | | - Key: team              |
| - Value: ml-platform     | | - Value: analytics       |
+--------------------------+ +--------------------------+

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 익스페리먼트 생성 및 메타데이터 주입

MLflow UI 상단의 **[Create]** 버튼을 통해 두 개의 격리된 엔티티를 웹 환경에서 직접 프로비저닝했습니다.

- **`fraud-detection` 세부 설정**:
- **Description**: `Fraud detection model for financial transactions`
- **Tags**: `team` : `ml-platform`

- **`churn-prediction` 세부 설정**:
- **Tags**: `team` : `analytics`

#### 3-2. 인프라 상태 검증 (UI 레코드 확인)

생성이 완료된 후 관리 대시보드 테이블에 메타데이터가 완벽하게 바인딩된 상태를 검증했습니다.

| Experiment Name      | Time Created / Modified | Description                                        | Tags                                  |
| -------------------- | ----------------------- | -------------------------------------------------- | ------------------------------------- |
| **churn-prediction** | 06/20/2026, 11:49:08 PM | -                                                  | `team: analytics`                     |
| **fraud-detection**  | 06/20/2026, 11:47:31 PM | Fraud detection model for financial transactions   | `team: ml-platform`                   |
| **legacy-models**    | 06/20/2026, 11:44:02 PM | Legacy models from 2023 — retained for auditing... | `team: ml-legacy`, `status: archived` |
| **Default**          | 06/20/2026, 11:43:50 PM | -                                                  | -                                     |

---

### 4. 핵심 개념 정리

- **MLflow Experiment & Tags**: 익스페리먼트는 관련된 상호 실행(Runs)들을 그룹화하는 최상위 논리적 격리 단위이며, 태그는 각 익스페리먼트나 런에 부여되는 키-값(Key-Value) 형태의 동적 메타데이터입니다.

💡**쉬운 비유**로, **'회사의 거대한 중앙 문서고에 팀별 전용 캐비닛과 이름표를 붙이는 것'**. 플랫폼 팀 캐비닛(`fraud-detection`)과 분석 팀 캐비닛(`churn-prediction`)을 따로 만들어 두고, 겉면에 담당 팀 이름표(`team: ml-platform`)를 붙여두는 것과 같습니다. 이렇게 해야 다른 팀이 와서 자기 서류를 엉뚱한 곳에 섞지 않습니다.

- **실무 관점의 의의**: 기업향 엔터프라이즈 MLOps 환경에서는 여러 데이터 과학자 팀이 공용 가상머신(VM)이나 쿠버네티스 클러스터 위에서 하나의 MLflow 서버를 바라보고 작업합니다. 이때 실험 격리(Isolation) 정책과 태그 규격(Naming Convention)을 강제하지 않으면 인프라가 난잡해지고 사후 비용 정산(FinOps)이나 프로젝트 자원 트래킹이 불가능해집니다.

---

### 5. 무엇을 배웠는가

- **인프라 자산 관리 역량**: 머신러닝 파이프라인에서 코드를 잘 짜는 것만큼이나 UI 레벨에서 자산을 명확하게 분류하고 태깅하는 것이 인프라 거버넌스(Governance) 확립에 치명적으로 중요하다는 점을 이해했습니다.

- **기존 시스템 보존(Immutable legacy)**: 신규 자산을 생성하는 과정에서 기존 가동 중인 레코드(`legacy-models`)에 영향을 주지 않고 레이어를 독립적으로 확장하는 인프라 운영 관점을 훈련했습니다.
