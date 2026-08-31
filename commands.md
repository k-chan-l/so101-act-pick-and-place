# 사용한 명령어

셸 히스토리와 HF에 올라간 `train_config.json`에서 복원한, 실제로 실행한 LeRobot 명령어입니다.

> **라운드-이름 대응** — `pick_and_place2` = 1차, `pick_and_place` = 2차, `pick_and_place3` = 3차. 자세한 내용은 [docs/experiments.md](docs/experiments.md).

## 공통 환경변수

```bash
conda activate lerobot
cd ~/lerobot

hf auth login --add-to-git-credential --token <HF_TOKEN>   # 토큰은 저장소에 커밋하지 말 것
hf auth whoami

export HF_USER="k-chan-l"
export TASK_NAME="pick_and_place3"     # 라운드별 데이터셋 이름
export TASK_DESCRIPTION="Pick up the blue cube and place it in the paper cup."
```

---

## record — 데이터 수집

3차(50 에피소드) 수집에 사용한 명령. `fourcc: MJPG`가 2차 조치로 추가된 부분입니다.

```bash
lerobot-record \
  --teleop.type=so101_leader \
  --teleop.port=/dev/so101_leader \
  --teleop.id=leader \
  --robot.type=so101_follower \
  --robot.port=/dev/so101_follower \
  --robot.id=follower \
  --robot.cameras='{
    top: {type: opencv, index_or_path: /dev/cam_top, width: 640, height: 480, fps: 30, fourcc: MJPG},
    wrist: {type: opencv, index_or_path: /dev/cam_wrist, width: 640, height: 480, fps: 30, fourcc: MJPG},
    }' \
  --dataset.single_task=${TASK_NAME} \
  --dataset.repo_id=${HF_USER}/${TASK_NAME} \
  --dataset.num_episodes=50 \
  --dataset.episode_time_s=30 \
  --dataset.reset_time_s=5 \
  --display_data=true \
  --dataset.push_to_hub=true
```

본 수집 전 파이프라인 확인용으로 `--dataset.num_episodes=5`, `--dataset.episode_time_s=20` 짧은 버전을 먼저 돌렸습니다.

---

## train — 학습

> **3차부터 학습은 SSH 원격 머신(RTX 5070 Ti)에서 실행합니다.** 아래 명령은 원격에서 돌리고, 데이터셋과 체크포인트는 HF를 경유해 주고받습니다 (→ [setup.md](setup.md) 4절). 1·2차는 로봇 제어 PC 로컬(RTX 3070 Laptop)에서 학습했습니다.

셸 히스토리에는 재개(resume) 명령만 남아 있지만, HF의 `train_config.json`에서 실제 학습 설정을 복원했습니다. 공통 정책 설정은 두 라운드가 같고 **batch_size / save_freq만 다릅니다**.

| 항목 | 1차 (`pick_and_place2`) | 2차 (`pick_and_place`) |
| --- | --- | --- |
| batch_size | 8 | 16 |
| num_workers | 4 | 8 |
| save_freq | 10,000 | 5,000 |
| steps | 100,000 | 100,000 |
| output_dir | `outputs/train/act_so101/pick_and_place2` | `outputs/train/act_so101/pick_and_place_b16` |
| wandb | 사용 | 미사용 |

공통: ACT / `vision_backbone=resnet18` / `dim_model=512` / `chunk_size=100` / `n_action_steps=100` / `n_encoder_layers=4` / `n_decoder_layers=1` / `use_vae=true` / lr 1e-5 (백본 포함) / weight_decay 1e-4 / seed 1000 / `device=cuda`

2차 학습 명령:

```bash
lerobot-train \
  --dataset.repo_id=k-chan-l/pick_and_place \
  --policy.type=act \
  --policy.device=cuda \
  --policy.push_to_hub=true \
  --policy.repo_id=k-chan-l/pick_and_place_act \
  --output_dir=outputs/train/act_so101/pick_and_place_b16 \
  --job_name=pick_and_place \
  --batch_size=16 \
  --num_workers=8 \
  --steps=100_000 \
  --save_freq=5_000 \
  --log_freq=200 \
  --seed=1000
```

1차는 위에서 `--dataset.repo_id`/`--policy.repo_id`/`--output_dir`/`--job_name`을 `pick_and_place2` 계열로 바꾸고 `--batch_size=8 --num_workers=4 --save_freq=10_000 --wandb.enable=true` 로 실행한 것과 같습니다.

학습 환경 준비:

```bash
pip install -e .              # lerobot 소스 설치
pip install lerobot[training]
pip install lerobot[dataset]
pip install lerobot[feetech]  # STS3215 모터
pip install rerun-sdk         # --display_data=true 시각화
wandb login
```

---

## resume — 이어서 학습

`--config_path`로 체크포인트의 `train_config.json`을 지정하고 `--resume=true`를 붙입니다. 두 정책 모두 최종 config에 `resume: true`가 남아 있어, 실제로 중단·재개를 거쳐 100k 스텝을 채웠습니다. 4차에서 3차 체크포인트에 복구 시연을 얹어 이어서 학습할 때도 이 형태를 사용합니다.

마지막 체크포인트에서 재개:

```bash
lerobot-train \
  --config_path=outputs/train/act_so101/pick_and_place2/checkpoints/last/pretrained_model/train_config.json \
  --resume=true \
  --save_freq=10_000
```

특정 스텝(20000) 체크포인트에서 재개:

```bash
lerobot-train \
  --config_path=outputs/train/act_so101/pick_and_place2/checkpoints/020000/pretrained_model/train_config.json \
  --resume=true \
  --save_freq=10_000
```

---

## rollout — 정책 평가

`--duration=0`은 시간 제한 없이 수동 종료까지 실행합니다. [protocol.md](docs/protocol.md)의 30초 제한은 스톱워치로 계측합니다.

```bash
lerobot-rollout \
  --strategy.type=base \
  --policy.path=${HF_USER}/${TASK_NAME}_act \
  --robot.type=so101_follower \
  --robot.port=/dev/so101_follower \
  --robot.id=follower \
  --robot.cameras='{
      top: {type: opencv, index_or_path: /dev/cam_top, width: 640, height: 480, fps: 30},
      wrist: {type: opencv, index_or_path: /dev/cam_wrist, width: 640, height: 480, fps: 30},
    }' \
  --task="${TASK_DESCRIPTION}" \
  --duration=0 \
  --display_data=true
```

라운드별 `--policy.path`: 1차 `k-chan-l/pick_and_place2_act`, 2차 `k-chan-l/pick_and_place_act`, 3차는 학습 완료 후 기입.

> 롤아웃의 카메라 설정에는 `fourcc: MJPG`가 빠져 있습니다. 학습 데이터는 MJPEG 스트림으로 수집했으므로, 평가 시에도 동일하게 맞추는 편이 관측 분포 일치 측면에서 안전합니다 — 3차 평가부터 추가 권장.

---

## 부속 — 캘리브레이션 / 텔레오퍼레이션

수집 전 매번 실행한 준비 명령입니다.

```bash
# 캘리브레이션 (leader / follower 각각)
lerobot-calibrate --teleop.type=so101_leader --teleop.port=/dev/so101_leader --teleop.id=leader
lerobot-calibrate --robot.type=so101_follower --robot.port=/dev/so101_follower --robot.id=follower

# 카메라 없이 팔 동작만 확인
lerobot-teleoperate \
  --robot.type=so101_follower --robot.port=/dev/so101_follower --robot.id=follower \
  --teleop.type=so101_leader --teleop.port=/dev/so101_leader --teleop.id=leader \
  --display_data=false

# 카메라 포함, 관측 화면 확인 (2차 이후 시연 시 이 화면만 보고 조작)
lerobot-teleoperate \
  --teleop.type=so101_leader --teleop.port=/dev/so101_leader --teleop.id=leader \
  --robot.type=so101_follower --robot.port=/dev/so101_follower --robot.id=follower \
  --robot.cameras='{
    top: {type: opencv, index_or_path: /dev/cam_top, width: 640, height: 480, fps: 30, fourcc: MJPG},
    wrist: {type: opencv, index_or_path: /dev/cam_wrist, width: 640, height: 480, fps: 30, fourcc: MJPG},
  }' \
  --display_data=true
```

장치 심볼릭 링크(`/dev/so101_*`, `/dev/cam_*`) 설정은 [setup.md](setup.md) 참고.
