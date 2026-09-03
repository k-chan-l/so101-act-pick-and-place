# 사용한 명령어

실제로 실행한 LeRobot 명령어입니다. 1~2차는 셸 히스토리에 재개(resume) 명령만 남아 있어 HF의 `train_config.json`에서 설정을 복원했고, 3차 이후는 실행 기록 그대로입니다.

> **라운드-이름 대응** — `pick_and_place2`=1차, `pick_and_place`=2차, `pick_and_place3`=3차, `pick_and_place4`=4차·4b, `pick_and_place5`=5차. → [docs/experiments.md](docs/experiments.md)

## 환경변수

```bash
conda activate lerobot
cd ~/lerobot

hf auth login --add-to-git-credential --token <HF_TOKEN>   # 토큰은 커밋하지 말 것
hf auth whoami

export HF_USER="k-chan-l"
export TASK_NAME="pick_and_place5"     # 라운드별 데이터셋 이름
export TASK_DESCRIPTION="Pick up the blue cube and place it in the paper cup."
```

장치 심볼릭 링크(`/dev/so101_*`, `/dev/cam_*`) 설정은 [setup.md](setup.md) 참고.

---

## record — 데이터 수집

**로봇 제어 PC에서 실행합니다.** 5차 수집 명령이며, 라운드가 달라도 `TASK_NAME`만 바뀝니다.

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

- `fourcc: MJPG`는 **2차의 조치**입니다. 없으면 카메라 2대가 USB 대역폭을 넘겨 프레임이 드랍되고, 이미지-액션 시간 정렬이 깨집니다 → [setup.md](setup.md) 2절
- 본 수집 전 `--dataset.num_episodes=5 --dataset.episode_time_s=20`으로 파이프라인을 먼저 확인했습니다

---

## train — 학습

**3차부터는 SSH 원격 머신(RTX 5070 Ti)에서 실행합니다.** 데이터셋은 HF를 경유합니다 → [setup.md](setup.md) 4절

긴 학습은 SSH가 끊기면 같이 죽으므로 tmux로 띄웁니다.

```bash
tmux new -s train
conda activate lerobot
cd ~/lerobot
# ... lerobot-train 실행 ...
# Ctrl+B 누르고 D 로 분리 / 다시 보려면:
tmux attach -t train
```

### 라운드별 설정 차이

정책 아키텍처는 전 라운드 동일하고 아래만 다릅니다.

| 항목 | 1차 | 2차 | 3차 | 4차 | 4b | **5차** |
| --- | --- | --- | --- | --- | --- | --- |
| 데이터셋 | `pick_and_place2` | `pick_and_place` | `pick_and_place3` | `pick_and_place4` | `pick_and_place4` | `pick_and_place5` |
| 초기화 | scratch | scratch | scratch | **3차 체크포인트** | **3차 체크포인트** | **scratch** |
| steps | 100k | 100k | 100k | **40k** | **40k** | 100k |
| batch_size | 8 | 16 | 16 | 16 | 16 | 16 |
| `image_transforms` | false | false | **true** | **false** | **true** | **true** |
| push_to_hub | true | true | false | true | true | true |
| 학습 머신 | 제어 PC (3070) | 별도 랩탑 (4070 추정) | 원격 5070 Ti | 원격 5070 Ti | 원격 5070 Ti | 원격 5070 Ti |

공통: ACT / `vision_backbone=resnet18` / `dim_model=512` / `chunk_size=100` / `n_action_steps=100` / `n_encoder_layers=4` / `n_decoder_layers=1` / `use_vae=true` / lr 1e-5 (백본 포함) / weight_decay 1e-4 / seed 1000 / `num_workers=8` / `device=cuda`

### 5차 (최종) — from scratch

```bash
lerobot-train \
  --dataset.repo_id=k-chan-l/pick_and_place5 \
  --dataset.image_transforms.enable=true \
  --policy.type=act \
  --policy.device=cuda \
  --policy.repo_id=k-chan-l/pick_and_place5_act \
  --policy.push_to_hub=true \
  --job_name=pick_and_place5 \
  --output_dir=outputs/train/act_so101/pick_and_place5 \
  --steps=100_000 \
  --save_checkpoint=true \
  --save_freq=10_000 \
  --batch_size=16 \
  --num_workers=8 \
  --seed=1000 \
  2>&1 | tee train5.log
```

### 4차 — 기존 체크포인트에서 이어 학습

`--policy.type` 대신 **`--policy.path`로 기존 정책을 지정**하면 그 가중치에서 이어집니다.

```bash
lerobot-train \
  --dataset.repo_id=k-chan-l/pick_and_place4 \
  --dataset.image_transforms.enable=false \
  --policy.path=k-chan-l/pick_and_place3_act \
  --policy.device=cuda \
  --policy.repo_id=k-chan-l/pick_and_place4_act \
  --policy.push_to_hub=true \
  --job_name=pick_and_place4 \
  --output_dir=outputs/train/act_so101/pick_and_place4 \
  --steps=40_000 \
  --save_checkpoint=true \
  --save_freq=10_000 \
  --batch_size=16 \
  --num_workers=8 \
  2>&1 | tee train4.log
```

**4b는 위에서 `--dataset.image_transforms.enable=true`로만 바꾼 것**입니다 (`--policy.repo_id=k-chan-l/pick_and_place4b_act`). 이 한 글자 차이가 큐브 근접을 3/10에서 10/10으로 되돌렸습니다 → [4b](docs/rounds/round4b.md)

### 3차

```bash
lerobot-train \
  --dataset.repo_id=k-chan-l/pick_and_place3 \
  --dataset.image_transforms.enable=true \
  --policy.type=act \
  --policy.device=cuda \
  --policy.repo_id=k-chan-l/pick_and_place3_act \
  --policy.push_to_hub=false \
  --job_name=pick_and_place3 \
  --output_dir=outputs/train/act_so101/pick_and_place3 \
  --steps=100_000 \
  --save_checkpoint=true \
  --save_freq=10_000 \
  --batch_size=16 \
  --num_workers=8 \
  2>&1 | tee train.log
```

> `--policy.push_to_hub=false`라 체크포인트가 원격에만 남습니다. 롤아웃은 제어 PC에서 해야 하므로 나중에 수동으로 HF에 push했습니다. **다음부터는 `true`로 두세요.**

1·2차는 위에서 `--dataset.repo_id`/`--policy.repo_id`/`--output_dir`/`--job_name`을 해당 라운드 이름으로 바꾸고, 1차는 `--batch_size=8 --num_workers=4 --wandb.enable=true`를 추가한 것과 같습니다.

### resume — 중단된 학습 이어가기

```bash
lerobot-train \
  --config_path=outputs/train/act_so101/pick_and_place5/checkpoints/last/pretrained_model/train_config.json \
  --resume=true \
  --save_freq=10_000
```

`last` 대신 `020000` 같은 스텝 디렉터리를 지정하면 그 시점부터 재개합니다.

---

## rollout — 평가

**로봇 제어 PC에서 실행합니다.** `--duration=30`이 [protocol.md](docs/protocol.md)의 30초 제한을 강제합니다.

```bash
lerobot-rollout \
  --strategy.type=base \
  --policy.path=${HF_USER}/${TASK_NAME}_act \
  --robot.type=so101_follower \
  --robot.port=/dev/so101_follower \
  --robot.id=follower \
  --robot.cameras='{
    top: {type: opencv, index_or_path: /dev/cam_top, width: 640, height: 480, fps: 30, fourcc: MJPG},
    wrist: {type: opencv, index_or_path: /dev/cam_wrist, width: 640, height: 480, fps: 30, fourcc: MJPG},
  }' \
  --task="${TASK_DESCRIPTION}" \
  --duration=30 \
  --device=cuda \
  --display_data=true
```

- 평가 시에도 **`fourcc: MJPG`를 반드시 넣습니다.** 학습 데이터를 MJPEG로 수집했으므로 빼면 관측 분포가 달라집니다
- 기본 `n_action_steps=100`이 **모든 라운드의 기준선**입니다. 라운드 간 비교는 이 설정에서만 성립합니다

### 추론 설정 변형 (실험 A / B)

재학습 없이 추론 설정만 바꿔 시험한 것입니다. **둘 다 기본값보다 나빴습니다** → [실험 A/B](docs/rounds/experiment-ab.md)

```bash
  --policy.n_action_steps=20                                    # A: 0/3
  --policy.temporal_ensemble_coeff=0.01 --policy.n_action_steps=1 # B: 0/10
```

`temporal_ensemble_coeff`를 쓸 때는 `n_action_steps=1`이어야 합니다.

---

## DAgger — 사람이 개입해 교정 데이터 수집

4차에 쓴 명령입니다. **결과적으로 회귀했으므로 재사용 전 [4차 기록](docs/rounds/round4.md)을 먼저 읽으세요.**

```bash
lerobot-rollout \
  --strategy.type=dagger \
  --strategy.num_episodes=20 \
  --policy.path=${HF_USER}/${TASK_NAME}_act \
  --robot.type=so101_follower \
  --robot.port=/dev/so101_follower \
  --robot.id=follower \
  --robot.cameras='{
    top: {type: opencv, index_or_path: /dev/cam_top, width: 640, height: 480, fps: 30, fourcc: MJPG},
    wrist: {type: opencv, index_or_path: /dev/cam_wrist, width: 640, height: 480, fps: 30, fourcc: MJPG},
  }' \
  --teleop.type=so101_leader \
  --teleop.port=/dev/so101_leader \
  --teleop.id=leader \
  --dataset.repo_id=${HF_USER}/rollout_${TASK_NAME}_dagger \
  --dataset.single_task="${TASK_DESCRIPTION}" \
  --device=cuda
```

키보드 조작:

| 키 | 동작 |
| --- | --- |
| **Space** | 자율 실행 ↔ 일시정지 (리더 팔이 팔로워 위치로 부드럽게 이동) |
| **Tab** | 일시정지 ↔ 교정 녹화 (여기서부터 기록됨) |
| **Enter** | HF 업로드 |
| **ESC** | 종료 |

흐름: 실패 조짐 → `Space` → `Tab` → 리더로 교정 시연 → `Tab`(에피소드 저장) → `Space`(자율 재개)

> **`--strategy.record_autonomous`를 지정하지 않았으므로 기본값 `false`(corrections-only)로 동작합니다.** 이게 맞습니다. `true`로 두면 정책의 자율 실행 프레임까지 기록되는데, `intervention` 컬럼을 `lerobot-train`도 ACT도 읽지 않으므로 **정책이 자기 실패 행동까지 따라 배웁니다.**
>
> `--strategy.num_episodes=20`으로 지정했으나 실제 수집은 **17개**입니다 (중간 종료). 이 값은 corrections-only 모드에서만 정지 조건으로 동작합니다.

수집한 교정 데이터를 기존 데이터셋과 합치려면 **`intervention` 컬럼을 제거한 사본**이 필요합니다. `--dataset.repo_id`는 문자열 하나만 받으므로 미리 병합해야 합니다.

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

# 카메라 포함 — 2차 이후 시연자는 이 화면만 보고 조작합니다
lerobot-teleoperate \
  --teleop.type=so101_leader --teleop.port=/dev/so101_leader --teleop.id=leader \
  --robot.type=so101_follower --robot.port=/dev/so101_follower --robot.id=follower \
  --robot.cameras='{
    top: {type: opencv, index_or_path: /dev/cam_top, width: 640, height: 480, fps: 30, fourcc: MJPG},
    wrist: {type: opencv, index_or_path: /dev/cam_wrist, width: 640, height: 480, fps: 30, fourcc: MJPG},
  }' \
  --display_data=true
```

5차부터는 촬영 전에 카메라 자동 노출·화이트밸런스를 끕니다 → [setup.md](setup.md)
