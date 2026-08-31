# 환경 설정 및 트러블슈팅 기록

## 1. udev 규칙 — 장치 이름 고정

`/dev/ttyACM*`, `/dev/video*` 번호는 부팅/재연결마다 바뀌므로 매번 명령어를 고쳐야 했습니다. 시리얼 번호와 USB 포트 경로로 심볼릭 링크를 고정했습니다.

### 식별자 확인

```bash
# 시리얼 포트 (팔)
ls /dev/ttyACM*
udevadm info -a -n /dev/ttyACM0 | grep '{serial}' -m 1
udevadm info -a -n /dev/ttyACM1 | grep '{serial}' -m 1

# 카메라
sudo apt install v4l-utils
v4l2-ctl --list-devices
udevadm info -a -n /dev/video2 | grep 'KERNELS=="[0-9]' | head -n 1
udevadm info -a -n /dev/video4 | grep 'KERNELS=="[0-9]' | head -n 1
```

### `/etc/udev/rules.d/99-serial.rules`

```udev
SUBSYSTEM=="tty", ATTRS{serial}=="5B8E114046", SYMLINK+="so101_leader"
SUBSYSTEM=="tty", ATTRS{serial}=="5B8E113009", SYMLINK+="so101_follower"

SUBSYSTEM=="video4linux", KERNELS=="1-2.3:1.0",   ATTR{index}=="0", SYMLINK+="cam_top"
SUBSYSTEM=="video4linux", KERNELS=="1-2.1.2:1.0", ATTR{index}=="0", SYMLINK+="cam_wrist"
```

- 팔은 **시리얼 번호**로 구분합니다 (leader/follower가 같은 모델이라 vendor/product로는 구분 불가).
- 카메라는 동일 모델 2대라 시리얼이 없거나 겹치므로 **USB 포트 경로(`KERNELS`)** 로 구분합니다. 따라서 **카메라를 다른 USB 포트에 꽂으면 규칙이 깨집니다** — 항상 같은 포트에 연결할 것.
- `ATTR{index}=="0"` 은 UVC 장치 하나가 만드는 여러 `/dev/video*` 노드 중 실제 영상 노드만 잡기 위한 조건입니다.

### 적용

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
ls -l /dev/so101_*
ls -l /dev/cam_*
```

### 시리얼 포트 권한

```bash
sudo usermod -a -G dialout $USER   # 재로그인 필요 (권장)
sudo chmod 666 /dev/ttyACM0        # 임시 조치
sudo chmod 666 /dev/ttyACM1
```

---

## 2. 카메라 MJPEG 강제 — USB 대역폭 문제

### 증상
카메라 2대를 640×480 / 30fps로 동시에 열면 스트림이 불안정했습니다. fps를 25로 낮춰도 개선되지 않았고, 텔레오퍼레이션 화면에서 프레임 드랍과 지연이 관찰됐습니다.

### 원인
UVC 카메라가 기본 **YUYV(무압축)** 로 협상되면 640×480×2바이트×30fps ≈ **18MB/s (147Mbps)** 를 한 대가 소비합니다. 두 대가 같은 USB 버스를 공유하면 USB 2.0 실효 대역폭을 넘겨, 커널이 대역폭 확보에 실패하거나 프레임을 버립니다.

### 확인

```bash
v4l2-ctl -d /dev/cam_top --list-formats-ext     # MJPG 지원 여부와 해상도/fps 조합
v4l2-ctl --get-fmt-video -d /dev/cam_top        # 현재 협상된 픽셀 포맷
v4l2-ctl --get-fmt-video -d /dev/cam_wrist
```

### 해결
카메라 설정에 `fourcc: MJPG`를 명시해 **MJPEG 압축 스트림**을 강제했습니다. 압축된 프레임만 전송되므로 대역폭이 한 자릿수 배 줄고, 두 대 모두 640×480 / 30fps가 안정적으로 유지됩니다.

```
--robot.cameras='{
  top:   {type: opencv, index_or_path: /dev/cam_top,   width: 640, height: 480, fps: 30, fourcc: MJPG},
  wrist: {type: opencv, index_or_path: /dev/cam_wrist, width: 640, height: 480, fps: 30, fourcc: MJPG},
}'
```

### 실험 기록 관점에서의 의미
이 문제는 단순한 설정 이슈가 아니라 **학습 데이터 품질에 직접 영향**을 줍니다. 프레임 드랍/지연이 있으면 기록된 이미지와 같은 타임스텝의 액션이 실제로는 다른 시점의 장면에 대응하게 되어 이미지-액션 시간 정렬이 깨집니다. 2차 라운드의 파지 실패 가설 (b)가 바로 이것이며, MJPEG 강제는 3차의 핵심 조치 중 하나입니다 → [docs/experiments.md](docs/experiments.md)

> 추가로 여유가 더 필요하면 두 카메라를 **서로 다른 USB 루트 허브**에 분산 연결하는 방법이 있습니다 (같은 허브 아래 있으면 대역폭을 공유).

---

## 3. 소프트웨어 설치

```bash
conda create -n lerobot python=3.12 -y
conda activate lerobot
conda install ffmpeg -c conda-forge

git clone https://github.com/huggingface/lerobot.git
cd lerobot
pip install -e .
python -c "import lerobot; print(lerobot.__version__)"

pip install lerobot[feetech]    # STS3215 모터
pip install lerobot[dataset]
pip install lerobot[training]
pip install rerun-sdk           # --display_data=true 시각화

wandb login
hf auth login --add-to-git-credential --token <HF_TOKEN>
```

로봇 제어 PC GPU: NVIDIA RTX 3070 Laptop, 8GB (CUDA 12.8 / cuDNN 9.6).

---

## 4. 학습 환경 분리 (3차부터)

2차까지는 로봇 제어 PC(RTX 3070 Laptop, 8GB)에서 수집과 학습을 모두 처리했지만, 3차부터 **학습만 SSH 원격의 RTX 5070 Ti 머신으로 분리**했습니다.

| 역할 | 머신 |
| --- | --- |
| 데이터 수집 (`lerobot-record`), 롤아웃 (`lerobot-rollout`) | 로봇 제어 PC — 팔·카메라가 USB로 물려 있어야 하므로 이동 불가 |
| 학습 (`lerobot-train`) | SSH 원격 머신 (RTX 5070 Ti) |

```bash
ssh <user>@<training-host>     # LAN 내 학습 머신
```

### 이 구성에서 주의할 점

- **데이터셋은 HF를 경유합니다.** 수집 측에서 `--dataset.push_to_hub=true`로 올리고, 학습 측에서 `--dataset.repo_id`로 받습니다. 두 머신이 파일시스템을 공유하지 않으므로 push를 빠뜨리면 학습 측이 예전 리비전을 씁니다.
- **체크포인트도 마찬가지입니다.** `--policy.push_to_hub=true`로 올려야 롤아웃하는 제어 PC에서 `--policy.path=k-chan-l/...` 로 바로 당겨쓸 수 있습니다. 원격에만 남은 `outputs/` 는 제어 PC에서 보이지 않습니다.
- **긴 학습은 세션이 끊기면 같이 죽습니다.** `tmux` 또는 `nohup`으로 띄워두면 SSH 연결이 끊겨도 계속 돕니다.
- 원격 머신에도 동일한 conda 환경과 LeRobot 설치가 필요합니다 (아래 3절과 동일, 단 `lerobot[feetech]`는 모터를 직접 물리지 않으므로 불필요).
