# 사용한 명령어

셸 히스토리에서 추출한, 실제로 실행한 LeRobot 명령어입니다.

## 공통 환경변수

```bash
conda activate lerobot
cd ~/lerobot

hf auth login --add-to-git-credential --token <HF_TOKEN>   # 토큰은 저장소에 커밋하지 말 것
hf auth whoami

export HF_USER="k-chan-l"
export TASK_NAME="<dataset>"          # 라운드별 데이터셋 이름 (히스토리상 pick_and_place2 / pick_and_place3 사용)
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

> 최초 학습을 시작한 `lerobot-train` 명령은 셸 히스토리에 남아 있지 않습니다 (히스토리에는 아래 resume 명령만 존재). 아래는 위 record 설정과 체크포인트 경로에 맞춰 **재구성한 템플릿**이며, 실제 실행 시 사용한 하이퍼파라미터로 갱신해야 합니다.

```bash
lerobot-train \
  --dataset.repo_id=${HF_USER}/${TASK_NAME} \
  --policy.type=act \
  --policy.device=cuda \
  --output_dir=outputs/train/act_so101/${TASK_NAME} \
  --job_name=act_so101_${TASK_NAME} \
  --policy.push_to_hub=true \
  --policy.repo_id=${HF_USER}/${TASK_NAME}_act \
  --save_freq=10_000 \
  --wandb.enable=true
```

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

`--config_path`로 체크포인트의 `train_config.json`을 지정하고 `--resume=true`를 붙입니다. 4차에서 3차 체크포인트에 복구 시연을 얹어 이어서 학습할 때도 이 형태를 사용합니다.

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
