# 🚀 100 Days of MLOps Challenge

이 저장소는 KodeKloud에서 제공하는 **100 Days of MLOps Challenge**를 수행하며 학습한 내용을 기록합니다.

데이터 버전 관리(DVC), 실험 추적(MLflow), 데이터 품질 검증(Great Expectations), 컨테이너화(Docker), 모델 서빙(FastAPI, BentoML), 모니터링(Prometheus, Grafana, Evidently), 그리고 쿠버네티스 기반의 CI/CD 및 오케스트레이션(Argo Workflows)까지 엔드투엔드 MLOps 파이프라인을 구축하는 과제를 해결합니다. 단순한 코드 복사가 아닌 MLOps 아키텍처 이해와 문제 해결 과정 중심으로 정리하고 있습니다.

## 📊 챌린지 진행 현황판

> **진행률: 21 / 100일 (21%)** · ✅ 완료 · ⏳ 예정

### 개발 환경 및 데이터 버전 관리 (DVC)

| Day | Topic                                                                                       | Tech Stack  | Status |
| --- | ------------------------------------------------------------------------------------------- | ----------- | ------ |
| 01  | [Create a Python Virtual Environment for ML](./challenges/day01.md)                         | `Python`    | ✅     |
| 02  | [Set Up and Configure Jupyter Notebook Server](./challenges/day02.md)                       | `Jupyter`   | ✅     |
| 03  | [Fix a Broken uv Lockfile Specification](./challenges/day03.md)                             | `uv`        | ✅     |
| 04  | [Create a Standard ML Project Structure](./challenges/day04.md)                             | `MLOps`     | ✅     |
| 05  | [Create a Makefile for ML Workflow Automation](./challenges/day05.md)                       | `Makefile`  | ✅     |
| 06  | [Set Up Code Quality Tools for ML Code](./challenges/day06.md)                              | `Linting`   | ✅     |
| 07  | [Package an ML Project as Installable Python Package](./challenges/day07.md)                | `Python`    | ✅     |
| 08  | [Configure Pre-Commit Hooks for ML Repository](./challenges/day08.md)                       | `Git`       | ✅     |
| 09  | [Create a Custom ML Project Template with Cookiecutter](./challenges/day09.md)              | `Python`    | ✅     |
| 10  | [Install and Initialize DVC in an ML Project](./challenges/day10.md)                        | `DVC`       | ✅     |
| 11  | [Track a Dataset with DVC](./challenges/day11.md)                                           | `DVC`       | ✅     |
| 12  | [Configure a DVC Remote Storage](./challenges/day12.md)                                     | `DVC`       | ✅     |
| 13  | [Pull DVC-Tracked Data from Remote](./challenges/day13.md)                                  | `DVC`       | ✅     |
| 14  | [Create a DVC Pipeline for Data Processing](./challenges/day14.md)                          | `DVC`       | ✅     |
| 15  | [Parameterize a DVC Pipeline](./challenges/day15.md)                                        | `DVC`       | ✅     |
| 16  | [Track ML Metrics with DVC](./challenges/day16.md)                                          | `DVC`       | ✅     |
| 17  | [Run and Compare DVC Experiments](./challenges/day17.md)                                    | `DVC`       | ✅     |
| 18  | [Version Datasets and Models Across Git Branches](./challenges/day18.md)                    | `DVC` `Git` | ✅     |
| 19  | [Build Complete DVC ML Pipeline with Remote Storage and Experiments](./challenges/day19.md) | `DVC`       | ✅     |

### 실험 추적 및 모델 관리 (MLflow)

| Day | Topic                                                                 | Tech Stack | Status |
| --- | --------------------------------------------------------------------- | ---------- | ------ |
| 20  | [Install and Start the MLflow Tracking Server](./challenges/day20.md) | `MLflow`   | ✅     |
| 21  | [Log an ML Experiment to MLflow](./challenges/day21.md)               | `MLflow`   | ✅     |
| 22  | Create and Organize MLflow Experiments                                | `MLflow`   | ⏳     |
| 23  | Search and Query MLflow Runs                                          | `MLflow`   | ⏳     |
| 24  | Enable MLflow Autologging                                             | `MLflow`   | ⏳     |
| 25  | Register, Version, and Manage Model Lifecycle                         | `MLflow`   | ⏳     |
| 26  | Compare Model Runs and Select the Best                                | `MLflow`   | ⏳     |
| 27  | Load Model from Registry with Custom Preprocessing                    | `MLflow`   | ⏳     |
| 28  | Fix a Broken MLflow Project and Re-Run It                             | `MLflow`   | ⏳     |
| 29  | Configure MLflow with Remote Tracking Server and Artifact Store       | `MLflow`   | ⏳     |
| 30  | End-to-End MLflow Lifecycle: Train, Register, Serve, Monitor          | `MLflow`   | ⏳     |

### 모델 훈련 최적화 및 기능 저장소 (Feast)

| Day | Topic                                                             | Tech Stack   | Status |
| --- | ----------------------------------------------------------------- | ------------ | ------ |
| 31  | Train a Scikit-Learn Model with Reproducible Script               | `sklearn`    | ⏳     |
| 32  | Manage Training Configuration with YAML                           | `YAML`       | ⏳     |
| 33  | Evaluate a Trained Model and Generate Classification Report       | `Evaluation` | ⏳     |
| 34  | Implement Cross-Validation for Model Selection                    | `sklearn`    | ⏳     |
| 35  | Hyperparameter Tuning with Optuna                                 | `Optuna`     | ⏳     |
| 36  | Automated Model Selection with FLAML AutoML                       | `FLAML`      | ⏳     |
| 37  | Distributed Model Training with Joblib Parallelization            | `Joblib`     | ⏳     |
| 38  | Build Modular Training Pipeline with Config-Driven Stages         | `MLOps`      | ⏳     |
| 39  | Train a PyTorch Model with GPU Support and Checkpointing          | `PyTorch`    | ⏳     |
| 40  | Production Training System: Tracking, Tuning, and Model Selection | `MLOps`      | ⏳     |
| 41  | Install and Initialize a Feast Feature Store                      | `Feast`      | ⏳     |
| 42  | Define Feature Views in Feast                                     | `Feast`      | ⏳     |
| 43  | Materialize Features to the Online Store                          | `Feast`      | ⏳     |

### 데이터 품질 및 가상 인프라 보안

| Day | Topic                                                    | Tech Stack | Status |
| --- | -------------------------------------------------------- | ---------- | ------ |
| 44  | Store MLflow's Admin Password in HashiCorp Vault         | `Vault`    | ⏳     |
| 45  | Fix a Broken Vault KV Policy for the MLflow Reader       | `Vault`    | ⏳     |
| 46  | Author Data-Quality Expectations with Great Expectations | `Gx`       | ⏳     |
| 47  | Debug a Failing Great Expectations Checkpoint            | `Gx`       | ⏳     |
| 48  | Publish Great Expectations Data Docs as a CI Artefact    | `Gx` `CI`  | ⏳     |
| 49  | Secrets + Data-Quality Integration Capstone              | `Capstone` | ⏳     |

### Docker Containerization

| Day | Topic                                                    | Tech Stack    | Status |
| --- | -------------------------------------------------------- | ------------- | ------ |
| 50  | Create Docker Image for ML Training Environment          | `Docker`      | ⏳     |
| 51  | Create Multi-Stage Docker Build for ML Serving           | `Docker`      | ⏳     |
| 52  | Set Up Local ML Dev Environment with Docker Compose      | `Docker`      | ⏳     |
| 53  | Create GPU-Enabled Docker Image for Deep Learning        | `Docker`      | ⏳     |
| 54  | Push ML Model Images to Container Registry               | `Docker`      | ⏳     |
| 55  | Add Health Checks and Graceful Shutdown to ML Containers | `Docker`      | ⏳     |
| 56  | Automate ML Docker Image Building in CI Pipeline         | `Docker` `CI` | ⏳     |

### 모델 서빙 및 배포 전략

| Day | Topic                                            | Tech Stack | Status |
| --- | ------------------------------------------------ | ---------- | ------ |
| 57  | Serve an ML Model with Flask                     | `Flask`    | ⏳     |
| 58  | Serve an ML Model with FastAPI                   | `FastAPI`  | ⏳     |
| 59  | Run Batch Predictions on a Dataset               | `Serving`  | ⏳     |
| 60  | Package a Model as a BentoML Service             | `BentoML`  | ⏳     |
| 61  | Containerize an ML Model API with Docker         | `Docker`   | ⏳     |
| 62  | Implement A/B Testing for Model Deployment       | `Serving`  | ⏳     |
| 63  | Implement Async Batch Prediction with Task Queue | `Async`    | ⏳     |
| 64  | Serve Multiple Models Behind Unified API Gateway | `Gateway`  | ⏳     |
| 65  | Implement Canary Deployment for Model Updates    | `Serving`  | ⏳     |
| 66  | Production Model Serving with Docker Compose     | `Docker`   | ⏳     |

### 모니터링 및 지속적 통합/배포 (CI/CD)

| Day | Topic                                                 | Tech Stack   | Status |
| --- | ----------------------------------------------------- | ------------ | ------ |
| 67  | Add Prometheus as a Grafana Data Source               | `Grafana`    | ⏳     |
| 68  | Generate a Model Performance Report                   | `Evidently`  | ⏳     |
| 69  | Generate a Data Quality Report                        | `Evidently`  | ⏳     |
| 70  | Create Automated Tests with Evidently Test Suites     | `Evidently`  | ⏳     |
| 71  | Set Up Evidently Monitoring Dashboard                 | `Evidently`  | ⏳     |
| 72  | Set Up Drift Detection Alerts                         | `Prometheus` | ⏳     |
| 73  | Automatic Retraining Triggered by Drift Detection     | `MLOps`      | ⏳     |
| 74  | Monitor Custom Business Metrics Alongside ML Metrics  | `Grafana`    | ⏳     |
| 75  | End-to-End Monitoring: Prometheus, Grafana, Evidently | `Monitor`    | ⏳     |
| 76  | Create CI Pipeline for ML Code Linting and Testing    | `CI/CD`      | ⏳     |
| 77  | Add Data Validation to CI Pipeline                    | `CI/CD`      | ⏳     |
| 78  | Add Model Validation Tests to CI                      | `CI/CD`      | ⏳     |
| 79  | Generate Model Performance Reports in CI with CML     | `CML`        | ⏳     |
| 80  | Automate Model Registration in CI/CD                  | `CI/CD`      | ⏳     |
| 81  | Automate Model Deployment with CD Pipeline            | `CI/CD`      | ⏳     |
| 82  | End-to-End ML CI/CD Pipeline                          | `CI/CD`      | ⏳     |
| 83  | Automated Model Rollback with Health Checks           | `CI/CD`      | ⏳     |
| 84  | Production ML CI/CD with Multi-Environment Promotion  | `CI/CD`      | ⏳     |

### Kubernetes Orchestration & 캡스톤 프로젝트

| Day | Topic                                                                  | Tech Stack   | Status |
| --- | ---------------------------------------------------------------------- | ------------ | ------ |
| 85  | Install Argo Workflows on Kubernetes                                   | `K8s` `Argo` | ⏳     |
| 86  | Create a Basic ML Training Workflow in Argo                            | `Argo`       | ⏳     |
| 87  | Pass Data Between Argo Steps with Output Parameters and Branching      | `Argo`       | ⏳     |
| 88  | Create an ML Pipeline with Prefect                                     | `Prefect`    | ⏳     |
| 89  | Parallel Model Training with Argo withItems Fan-Out                    | `Argo`       | ⏳     |
| 90  | Automated Retraining with Argo CronWorkflow                            | `Argo`       | ⏳     |
| 91  | Production ML Pipeline: Argo Workflows + MLflow on Kubernetes          | `K8s` `Argo` | ⏳     |
| 92  | Deploy an ML Model on Kubernetes                                       | `K8s`        | ⏳     |
| 93  | Configure HPA for ML Serving Deployment                                | `K8s`        | ⏳     |
| 94  | Deploy a Model with KServe InferenceService                            | `KServe`     | ⏳     |
| 95  | Kubeflow Pipelines - Install and Run a Basic KFP Pipeline              | `Kubeflow`   | ⏳     |
| 96  | GitOps Model Deployment with ArgoCD                                    | `ArgoCD`     | ⏳     |
| 97  | Capstone (1/4): End-to-End ML System - Train, Register, Serve          | `Capstone`   | ⏳     |
| 98  | Capstone (2/4): Monitoring and Automated Retraining                    | `Capstone`   | ⏳     |
| 99  | Capstone (3/4): Orchestrate the Full MLOps Loop with Argo Workflows    | `Capstone`   | ⏳     |
| 100 | Capstone (4/4): Close the Loop with Prometheus + Grafana Observability | `Capstone`   | ⏳     |
