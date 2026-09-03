# SO-101 Pick-and-Place with ACT

SO-101 로봇팔로 "파란 큐브를 집어 종이컵에 넣는" 태스크를 ACT 모방학습으로 학습시킨 **실험 기록**입니다. 라운드마다 실패 유형을 분석하고 데이터 수집 방식을 고쳐 나갔습니다.

> 코드 저장소가 아닙니다. 학습·추론은 [LeRobot](https://github.com/huggingface/lerobot) 업스트림을 그대로 쓰고, 여기에는 **데이터 구성·평가 결과·실패 가설**만 남깁니다.
> 장비 반납으로 **5차를 마지막으로 종료**했습니다.

## 결과

| 라운드 | 데이터 | 검증한 가설 | 판정 | 성공률 |
| --- | --- | --- | --- | --- |
| [1차](docs/rounds/round1.md) | 50 ep, 정지 없음 | 베이스라인 | — | **0%** |
| [2차](docs/rounds/round2.md) | 50 ep, 정지 40% | 배경 잡음이 causal confusion을 일으킨다 | 확인 | **2/10 (20%)** |
| [3차](docs/rounds/round3.md) | 50 ep, 정지 20% | 특권 정보·프레임 드랍이 정밀 파지를 막는다 | 확인 | **4/20 (20%)** |
| [4차](docs/rounds/round4.md) | 3차 + DAgger 17 ep | DAgger 교정이 파지 정밀도를 올린다 | **반증** | **0/10** |
| [4b](docs/rounds/round4b.md) | 4차와 동일, 증강만 on | 4차 회귀는 증강 제거 탓이다 | 확인 | 0/10 (근접 10/10) |
| [A/B](docs/rounds/experiment-ab.md) | 재학습 없음 | 폐루프화하면 정밀도가 오른다 | **반증** | 0/3, 0/10 |
| [**5차**](docs/rounds/round5.md) | 신규 50 ep, from scratch | 조건을 통제하고 재수집하면 개선된다 | 확인 | **7/21 (33%)** |

성공률은 전부 `n_action_steps=100` 기준입니다. **추론 설정이 다르면 비교되지 않습니다.**

## 최종 평가 (5차)

7지점 × 3회 = 21회, 시도당 30초. 성공 기준은 파지 → 컵에 투입 → 홈 복귀를 30초 안에 전부 만족.

| 항목 | 값 |
| --- | --- |
| 성공 | **7 / 21 (33%)** |
| 파지 실패 | 14 |
| **파지 후 실패** | **0** |
| 무반응 | 0 |

| 지점 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 성공 | 0/3 | 2/3 | 2/3 | 2/3 | 0/3 | 0/3 | 1/3 |

**병목이 파지 단계 하나로 좁혀졌습니다.** 컵을 고정하자 파지 이후 구간(이송 → 투입 → 복귀)이 안정화돼 파지 후 실패가 0건입니다. 3차에서는 파지 성공 9건 중 5건이 컵 단계에서 깨졌습니다.

**남은 원인은 시각적 위치 판별입니다.** 학습 데이터를 사후 분석하니 7개 큐브 지점의 파지 자세가 서로 겹쳐 있었습니다 — 7지점 전체가 베이스 회전 12.8° 안에 있는데 지점 하나의 시연 산포가 ±3.3°(분리비 1.36)입니다. 지점당 3회로는 지점별 편차의 유의성도 확인할 수 없었습니다(p=0.169).

→ [분석과 남은 과제](docs/findings.md) · [데이터셋 사후 분석](docs/dataset-analysis.md)

## 스택

| | |
| --- | --- |
| 로봇 | SO-101 follower + leader (leader teleoperation), Feetech STS3215 ×6 |
| 카메라 | `cam_top` + `cam_wrist` — 640×480 / 30fps / **MJPEG** |
| 정책 | ACT, ResNet-18, `chunk_size=100`, `n_action_steps=100` |
| 환경 | LeRobot / conda `lerobot` / Python 3.12 / Ubuntu |
| 머신 | 수집·평가: 제어 PC (RTX 3070 Laptop) / 학습: SSH 원격 (RTX 5070 Ti, 3차~) |

## 명령어 요약

전체는 [commands.md](commands.md)에 있습니다.

```bash
export HF_USER="k-chan-l"
export TASK_NAME="pick_and_place5"
export TASK_DESCRIPTION="Pick up the blue cube and place it in the paper cup."
```

**수집** (제어 PC) — 50 에피소드, 에피소드당 30초

```bash
lerobot-record --teleop.type=so101_leader --robot.type=so101_follower \
  --robot.cameras='{top: {..., fourcc: MJPG}, wrist: {..., fourcc: MJPG}}' \
  --dataset.repo_id=${HF_USER}/${TASK_NAME} --dataset.num_episodes=50 \
  --dataset.episode_time_s=30 --dataset.push_to_hub=true --display_data=true
```

**학습** (SSH 원격, tmux) — 5차는 from scratch 100k

```bash
tmux new -s train
lerobot-train --dataset.repo_id=${HF_USER}/${TASK_NAME} \
  --dataset.image_transforms.enable=true --policy.type=act --policy.device=cuda \
  --policy.repo_id=${HF_USER}/${TASK_NAME}_act --policy.push_to_hub=true \
  --steps=100_000 --batch_size=16 --num_workers=8 --seed=1000 | tee train5.log
# Ctrl+B, D 로 분리 / tmux attach -t train 로 복귀
```

**평가** (제어 PC) — `--duration=30`이 30초 제한을 강제

```bash
lerobot-rollout --strategy.type=base --policy.path=${HF_USER}/${TASK_NAME}_act \
  --robot.type=so101_follower --robot.cameras='{...fourcc: MJPG...}' \
  --task="${TASK_DESCRIPTION}" --duration=30 --device=cuda --display_data=true
```

`fourcc: MJPG`는 수집과 평가 **양쪽 모두**에 넣어야 합니다. 없으면 USB 대역폭 초과로 프레임이 드랍되고 이미지-액션 시간 정렬이 깨집니다.

## 문서

| 문서 | 내용 |
| --- | --- |
| [docs/experiments.md](docs/experiments.md) | **실험 기록 목차.** 여기서 시작 |
| [docs/rounds/](docs/rounds/) | 라운드별 상세 (설계 → 결과 → 실패 분석 → 다음 라운드 근거) |
| [docs/findings.md](docs/findings.md) | 무엇을 배웠는가 + Future Work |
| [docs/dataset-analysis.md](docs/dataset-analysis.md) | 데이터셋 사후 분석. 로봇 없이 재현 가능 |
| [docs/protocol.md](docs/protocol.md) | 평가 프로토콜, 실패 유형 정의 |
| [commands.md](commands.md) | 실행한 LeRobot 명령어 전체 + LeRobot 관련 확인된 사실 |
| [setup.md](setup.md) | udev, 카메라 MJPEG·USB 대역폭 해결, 설치 |

## Hugging Face

계정: [k-chan-l](https://huggingface.co/k-chan-l)

| 라운드 | 데이터셋 | 정책 |
| --- | --- | --- |
| 1차 | `pick_and_place2` | `pick_and_place2_act` |
| 2차 | `pick_and_place` | `pick_and_place_act` |
| 3차 | `pick_and_place3` | `pick_and_place3_act` |
| 4차 / 4b | `pick_and_place4` | `pick_and_place4_act` / `pick_and_place4b_act` |
| **5차** | [`pick_and_place5`](https://huggingface.co/datasets/k-chan-l/pick_and_place5) | [`pick_and_place5_act`](https://huggingface.co/k-chan-l/pick_and_place5_act) |

> **이름 주의** — 데이터셋 숫자가 실험 순서와 다릅니다. `pick_and_place2`가 1차, 숫자 없는 `pick_and_place`가 2차입니다.
