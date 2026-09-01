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
| 로봇 제어 PC | 노트북, NVIDIA RTX 3070 Laptop (8GB) — 데이터 수집·롤아웃 전담 |
| 학습 GPU | 1차: 위 RTX 3070 Laptop (로컬) / 2차: 별도 랩탑 (RTX 4070 추정) / **3차부터: SSH 원격 접속한 RTX 5070 Ti** |

### 소프트웨어
| 항목 | 내용 |
| --- | --- |
| 프레임워크 | LeRobot (`lerobot-record` / `lerobot-train` / `lerobot-rollout`) |
| 정책 | ACT (Action Chunking Transformer), ResNet-18 백본, chunk_size 100 |
| 환경 | conda `lerobot` / Python 3.12 / Ubuntu |
| 로깅 | Weights & Biases, rerun (`--display_data=true`) |

## 라운드별 결과 요약

> 1~4차는 **탐색 단계**(라운드마다 변수를 여럿 바꿈), 5차부터 **통제 실험**입니다. 성공률은 모두 `n_action_steps=100` 기준이며, 추론 설정이 다르면 비교되지 않습니다.

| 라운드 | 데이터 구성 | 성공률 | 핵심 조치 |
| --- | --- | --- | --- |
| 1차 | 50 ep (17,480 프레임), 정지 없음 / 책상에 잡동사니 | 0% (허공 파지) | 책상 정리, 정지(no-op) 데이터 추가 |
| 2차 | 50 ep (14,315 프레임), 정지 20 (40%) / 팔을 직접 보며 시연 | 2/10 (20%), 10회 축약 | MJPEG 압축, 카메라 화면 보며 시연, 6단계 프로토콜, 정지 40%→20% |
| 3차 | 50 ep (27,147 프레임), 정지 10 (20%) / 증강 on | **4/20 (20%)** | 병목이 파지 정밀도(10)와 컵 단계(5)로 갈림 → 복구 시연 보강 |
| 4차 | 3차 + DAgger 교정 17 ep = 67 ep (33,547 프레임) / 3차 가중치에서 40k fine-tune, 증강 off | **0/10 (회귀)** | 교정 데이터가 좁은 관절 영역에 뭉쳐 평균화 → 허공 잡음. 17 ep 보류 |
| 4b | 4차와 동일 조건, 증강만 on | *진행 중* | 회귀가 증강 제거 탓인지 DAgger 노이즈 탓인지 분리 |
| 5차 (준비) | 신규 50 ep (정상 45 + 정지 5), from scratch | — | 8지점 × 0°/45°, 카메라 노출 고정, 파지 후 멈춤 없이 곧장 컵 |

추론 설정 실험 (재학습 없음, 3차 정책 대상):

| 실험 | 설정 | 결과 |
| --- | --- | --- |
| 기준 | `n_action_steps=100` | **4/20 (20%)** |
| A | `n_action_steps=20` | 0/3 — 진동하다 정지, 3회로 중단 |
| B | `temporal_ensemble_coeff=0.01` | 0/10 — 그리퍼 개폐가 뭉개짐 |

**100 > TE > 20.** 관측을 자주 넣을수록 나빠진다는 것은 정책이 관측을 거의 쓰지 못한 채 초기 궤적에 커밋해 버티고 있다는 뜻이며, 추론 설정으로는 해결되지 않습니다 → 5차 데이터 재수집.

상세 기록은 [docs/experiments.md](docs/experiments.md) 참고.

## Hugging Face

계정: [k-chan-l](https://huggingface.co/k-chan-l)

| 라운드 | 데이터셋 | 학습된 정책 |
| --- | --- | --- |
| 1차 | [`k-chan-l/pick_and_place2`](https://huggingface.co/datasets/k-chan-l/pick_and_place2) | [`k-chan-l/pick_and_place2_act`](https://huggingface.co/k-chan-l/pick_and_place2_act) |
| 2차 | [`k-chan-l/pick_and_place`](https://huggingface.co/datasets/k-chan-l/pick_and_place) | [`k-chan-l/pick_and_place_act`](https://huggingface.co/k-chan-l/pick_and_place_act) |
| 3차 | [`k-chan-l/pick_and_place3`](https://huggingface.co/datasets/k-chan-l/pick_and_place3) | [`k-chan-l/pick_and_place3_act`](https://huggingface.co/k-chan-l/pick_and_place3_act) |
| 4차 | `k-chan-l/pick_and_place4` (3차+교정 병합) | `k-chan-l/pick_and_place4_act` |

> **이름 주의** — 데이터셋 이름의 숫자가 실험 순서와 일치하지 않습니다. **`pick_and_place2`가 1차, 숫자 없는 `pick_and_place`가 2차**, `pick_and_place3`이 3차입니다. 정책 이름도 데이터셋을 따라가므로 `pick_and_place2_act`가 1차 정책, `pick_and_place_act`가 2차 정책입니다.

## 문서

- [docs/experiments.md](docs/experiments.md) — 라운드별 상세 실험 기록 (데이터 구성 / 촬영 방식 / 결과 / 실패 원인 가설 / 다음 라운드 조치)
- [docs/protocol.md](docs/protocol.md) — 평가 프로토콜, 실패 유형 정의, **5차 통제 실험 조건**
- [commands.md](commands.md) — 실제로 사용한 LeRobot 명령어 (record / train / resume / rollout)
- [setup.md](setup.md) — udev 규칙, 카메라 MJPEG·USB 대역폭 문제 해결 기록, 소프트웨어 설치
- [media/](media/) — 롤아웃 영상
- [CLAUDE.md](CLAUDE.md) — 저장소 작업 규칙과 LeRobot 관련 검증된 사실
