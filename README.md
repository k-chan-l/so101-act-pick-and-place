# SO-101 Pick-and-Place with ACT: iterative failure analysis

SO-101 로봇팔로 "파란 큐브를 집어 종이컵에 넣는" 태스크를 ACT 모방학습으로 학습시키면서, 라운드마다 롤아웃 실패 유형을 분석하고 데이터 수집 방식을 고쳐 나간 실험 기록 저장소입니다.

> 코드 저장소가 아니라 **실험 기록 저장소**입니다. 학습/추론 코드는 [LeRobot](https://github.com/huggingface/lerobot) 업스트림을 그대로 사용하고, 이 저장소에는 데이터 구성·촬영 방식·평가 결과·실패 가설·다음 조치만 남깁니다.

## 스택

### 하드웨어
| 항목 | 내용 |
| --- | --- |
| 로봇팔 | SO-101 follower + leader (leader teleoperation 방식 시연) |
| 액추에이터 | Feetech STS3215 (ID 1~6) |
| 카메라 | `cam_top` (전체뷰), `cam_wrist` (손목캠) — 각 640×480 / 30fps / MJPEG |
| 워크스테이션 GPU | NVIDIA RTX 3070 |

### 소프트웨어
| 항목 | 내용 |
| --- | --- |
| 프레임워크 | LeRobot (`lerobot-record` / `lerobot-train` / `lerobot-rollout`) |
| 정책 | ACT (Action Chunking Transformer) |
| 환경 | conda `lerobot` / Python 3.12 / Ubuntu |
| 로깅 | Weights & Biases, rerun (`--display_data=true`) |

## 라운드별 결과 요약

| 라운드 | 데이터 구성 | 성공률 | 핵심 조치 |
| --- | --- | --- | --- |
| 1차 | 책상 위 잡동사니가 남아 있는 환경에서 시연 촬영 | 0% (전 시도 실패, 허공 파지) | 책상 정리 후 타겟 물체만 배치, 정지(no-op) 데이터 추가 |
| 2차 | 50 에피소드 (정지 20, 40%) / 팔을 직접 보며 시연 | **2/10 (20%)**, 30초 제한 | 카메라 스트림 압축 전송, 카메라 화면 보며 시연, 체계적 촬영 프로토콜 도입, 정지 데이터 비율 40%→20% |
| 3차 | 50 에피소드 (정지 10, 20%) / 2차 조치 반영 | *학습 중 — 결과 미기록* | (평가 후 기록 예정) |
| 4차 (계획) | 3차 데이터셋 + 복구(recovery) 시연 15~20개 | — | 3차 실패 유형 분포에 맞춘 복구 시연 추가 후 기존 체크포인트에서 이어서 학습 |

상세 기록은 [docs/experiments.md](docs/experiments.md) 참고.

## Hugging Face

| 항목 | 링크 |
| --- | --- |
| 계정 | [k-chan-l](https://huggingface.co/k-chan-l) |
| 1차 데이터셋 | `k-chan-l/<dataset>` *(TBD)* |
| 2차 데이터셋 | `k-chan-l/<dataset>` *(TBD)* |
| 3차 데이터셋 | `k-chan-l/<dataset>` *(TBD)* |
| 학습 정책 (ACT) | `k-chan-l/<dataset>_act` *(TBD)* |

> 실제 실행 시에는 `TASK_NAME` 환경변수로 데이터셋 이름을 넘겼습니다 (`--dataset.repo_id=${HF_USER}/${TASK_NAME}`). 확정된 이름을 위 표에 채워 넣으세요.

## 문서

- [docs/experiments.md](docs/experiments.md) — 라운드별 상세 실험 기록 (데이터 구성 / 촬영 방식 / 결과 / 실패 원인 가설 / 다음 라운드 조치)
- [docs/protocol.md](docs/protocol.md) — 평가 프로토콜과 실패 유형 정의
- [commands.md](commands.md) — 실제로 사용한 LeRobot 명령어 (record / train / resume / rollout)
- [setup.md](setup.md) — udev 규칙, 카메라 대역폭 문제, 모터 이슈 해결 기록
- [media/](media/) — 롤아웃 영상
