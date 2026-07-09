## 📅 Day 39: PyTorch 모델 훈련의 Device-Awareness 최적화 및 Checkpointing 파이프라인 구축

### 1. Task

- **요구사항**: xFusionCorp Industries의 ML 플랫폼 팀은 PyTorch를 활용하여 소형 사기 탐지 모델을 배포하고 있습니다. 현재 훈련 스크립트(`train_pytorch.py`)는 CUDA(GPU) 환경이 존재한다고 하드코딩되어 있어, CPU만 제공되는 랩 환경이나 프로덕션의 다양한 노드에서 텐서 연산 에러를 일으킵니다. 또한, 긴 훈련 세션이 중단되었을 때 복구할 수 있는 체크포인트(Checkpoint) 기능이 누락되어 있습니다.

- **목표**:
  1. 하드코딩된 `.cuda()` 메서드를 모두 제거하고, 런타임에 가용한 디바이스(CPU 또는 GPU)를 동적으로 감지하여 할당하는 Device-Aware 스크립트로 교정
  2. MLflow `pytorch-training` 실험에 하드코딩된 `"cuda"` 대신 런타임에 실제 사용된 디바이스명(예: `"cpu"`)을 정확하게 로깅
  3. 훈련 루프 내에서 매 10번째 에포크(0, 10, 20)마다 모델과 옵티마이저의 상태(State-dict)를 포함한 복구 가능한 체크포인트 파일 생성

---

### 2. Workflow

```text
[Environment Detection]
  └── torch.cuda.is_available() ──> GPU가 없으므로 Device: "cpu" 동적 할당

[Training Pipeline: train_pytorch.py]
  ├── MLflow: log_param("device", "cpu") 기록
  │
  ├── Epoch 0: 학습 진행 ──> [Checkpoint 저장: ckpt_epoch_0.pt]
  ├── Epoch 1~9: 학습 진행 ──> (Loss 연산 및 가중치 업데이트)
  ├── Epoch 10: 학습 진행 ──> [Checkpoint 저장: ckpt_epoch_10.pt]
  ├── ...
  ├── Epoch 20: 학습 진행 ──> [Checkpoint 저장: ckpt_epoch_20.pt]
  │
  ▼
[End State: Artifacts & MLflow]
  ├── models/fraud_model.pt (전체 에포크 완료 후 최종 모델 가중치 저장)
  └── MLflow UI: metrics.final_loss = 0.6110 로깅 완료

```

---

### 3. 해결 과정 (Action & Troubleshooting)

#### 3-1. 디바이스 동적 할당(Device-Aware) 로직 구현

기존 스크립트는 GPU의 존재 여부를 묻지 않고 무조건 `model.cuda()`를 호출하여 CPU 랩 환경에서 첫 텐서 연산 시 즉각적인 셧다운을 유발했습니다. 이를 동적으로 감지하도록 교정했습니다.

```python
# src/models/train_pytorch.py

# [수정 전]: 무조건 GPU(cuda)를 가정하는 하드코딩
# model = model.cuda()
# xb = X_t.cuda()

# [수정 후]: 가용한 가속기(Accelerator)를 스스로 판단하여 할당
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = model.to(device)
xb = X_t.to(device)

# MLflow 로깅 역시 실제 할당된 디바이스 문자열로 교정
mlflow.log_param("device", str(device))

```

#### 3-2. 장기 훈련 대비 체크포인트(Checkpointing) 로직 추가

학습 진행 상태가 유실되는 것을 방지하기 위해, 매 10 에포크마다 현재까지의 에포크, 모델 가중치, 옵티마이저 상태, 손실값을 딕셔너리로 묶어 `torch.save()`로 안전하게 디스크에 백업했습니다.

```python
# src/models/train_pytorch.py (훈련 루프 내부)

# 매 10번째 에포크(0, 10, 20)마다 체크포인트 파일 분리 저장
if epoch % 10 == 0:
    ckpt_path = os.path.join(CHECKPOINT_DIR, f"ckpt_epoch_{epoch}.pt")
    torch.save({
        "epoch": epoch,
        "model_state_dict": model.state_dict(),
        "optimizer_state_dict": optimizer.state_dict(),
        "loss": final_loss
    }, ckpt_path)

```

#### 3-3. 파이프라인 구동 및 무결성 검증

터미널에서 훈련 스크립트를 실행하여 CPU 환경에서 에러 없이 30 에포크가 모두 실행되는지 검증하고, 산출물을 확인했습니다.

```bash
# 훈련 스크립트 실행 (Device-Aware 감지를 통해 CPU로 정상 실행됨)
python3 src/models/train_pytorch.py

# 체크포인트 디렉토리 확인
ls -l /root/code/fraud-detection/checkpoints/
# 출력: ckpt_epoch_0.pt, ckpt_epoch_10.pt, ckpt_epoch_20.pt 정상 생성 확인

```

---

### 4. 핵심 개념 정리

- **Device-Agnostic Code (디바이스 독립적 코드)**: 코드가 실행되는 환경(하드웨어)에 얽매이지 않고, CPU, GPU, 혹은 Apple Silicon(MPS) 등 주어진 자원을 스스로 파악하여 유연하게 대응하는 프로그래밍 패러다임입니다. PyTorch에서는 `.to(device)` 패턴이 표준입니다.

- **Checkpointing (체크포인트)**: 게임의 '세이브 포인트'와 완전히 같습니다. 딥러닝 학습 중 특정 시점의 모델 두뇌 상태(`model_state_dict`)와 훈련의 가속/방향 상태(`optimizer_state_dict`)를 디스크에 백업해 두어, 서버가 정전되거나 메모리가 터져도 처음부터 다시 학습하지 않고 세이브 시점부터 이어서(Resume) 학습할 수 있게 해주는 필수적인 결함 감내(Fault-Tolerance) 기법입니다.

---

### 5. 무엇을 배웠는가 (Takeaway)

- **회고(Retrospective)**: 내 로컬 PC에 비싼 GPU가 달려있다고 해서 `.cuda()`를 남발하여 스크립트를 작성하는 것이 얼마나 이기적이고 위험한 엔지니어링인지 반성했습니다. 개발 환경과 프로덕션 환경, 혹은 동료들의 테스트 환경은 언제나 다를 수 있음을 인정하고, 스크립트 스스로가 환경을 감지하게 만드는 '유연함'이 진정한 안정성임을 깨달았습니다.

- **Pain Point**: 모델이 복잡해질수록 훈련 시간은 몇 주(Weeks) 단위로 길어집니다. 만약 체크포인트 로직 없이 14일째 훈련 중이던 모델이 클라우드 인스턴스의 일시적 네트워크 장애로 셧다운된다면, 회사는 막대한 컴퓨팅 비용과 2주의 시간을 허공에 날리게 됩니다. '어떻게든 끝까지 돌겠지'라는 요행을 바라는 대신, '언제든 죽을 수 있다'고 가정하는 방어적 인프라 설계가 필수적입니다.

- **성장 포인트**: 단순히 "AI 모델을 학습시킨다"는 1차원적 목표를 넘어, "어떤 악조건의 서버에 던져져도 알아서 환경을 인식해 학습하고, 스스로 백업을 남기며 생존하는 강건한 ML 파이프라인"을 구축하는 아키텍트의 관점을 얻게 되었습니다. 이제 인프라 팀이나 다른 부서와 협업할 때 "이 코드 아무 서버에서나 돌리셔도 됩니다. 중간에 꺼지면 마지막 체크포인트부터 이어서 돌게끔 다 설계해 뒀습니다"라고 자신 있게 말할 수 있는 준비가 되었습니다.
