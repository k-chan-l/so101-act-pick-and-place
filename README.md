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

## 프로젝트 요약

다섯 라운드에 걸쳐 학습시켰고 최종 성공률은 **21회 중 7회(33%)** 입니다. 이 저장소는 그 수치보다, 매 라운드 **무엇을 가설로 세우고 어떻게 반증했는지**를 남기기 위한 기록입니다. 장비 반납으로 5차가 마지막 실험입니다.

### 다섯 라운드가 검증한 것

각 라운드는 이전 라운드의 실패 유형 분포에서 가설을 세우고, 한 번에 하나씩 조건을 바꿔 확인했습니다.

| 라운드 | 검증한 가설 | 판정 | 성공률 |
| --- | --- | --- | --- |
| [1차](docs/rounds/round1.md) | 베이스라인 확립 | — | 0% (허공 파지) |
| [2차](docs/rounds/round2.md) | 배경 잡음이 causal confusion을 일으킨다 | **확인** | 2/10 (20%) |
| [3차](docs/rounds/round3.md) | 시연자 특권 정보와 프레임 드랍이 정밀 파지를 막는다 | **확인** | 4/20 (20%) |
| [4차](docs/rounds/round4.md) | DAgger 교정 데이터가 파지 정밀도를 올린다 | **반증** | 0/10 |
| [4b](docs/rounds/round4b.md) | 4차 회귀의 원인은 증강 제거다 | **확인** | 근접 3/10 → 10/10 |
| [A/B](docs/rounds/experiment-ab.md) | 추론을 폐루프화하면 정밀도가 오른다 | **반증** | 0/3, 0/10 |
| [**5차**](docs/rounds/round5.md) | 조건을 통제하고 from scratch로 재수집하면 개선된다 | **확인** | **7/21 (33%)** |

반증된 가설을 지우지 않고 남긴 것은 음성 결과가 다음 라운드 설계의 근거였기 때문입니다. A/B의 실패는 "폐루프로 갈수록 나빠진다"는 진단을 낳았고, 그것이 추론 설정 조정을 포기하고 데이터를 다시 모으기로 한 결정의 근거가 됐습니다.

### 무엇을 배웠는가

**병목이 파지 단계 하나로 좁혀졌습니다.** 컵 위치를 고정하자 파지 이후 구간(이송 → 투입 → 복귀)이 안정화됐습니다. 3차에서는 파지 성공 9건 중 5건이 컵 단계에서 깨졌는데, 5차는 **파지 후 실패가 0건**입니다.

**남은 병목의 원인은 시각적 위치 판별입니다.** 학습 데이터를 사후 분석한 결과 7개 큐브 지점의 파지 자세가 서로 겹쳐 있었습니다 — 베이스 회전각 기준 7지점 전체가 12.8° 안에 있는데 지점 하나의 시연 산포가 ±3.3°입니다(분리비 1.36). 인접 지점이 데이터에서 구분되지 않으므로 정책은 겹친 지점들의 평균 궤적을 출력합니다. 여기에 `n_action_steps=100`(30fps에서 **3.33초 open-loop**)이 겹쳐, 어긋난 궤적을 끝까지 되돌리지 못합니다.

운영자는 이를 "정책이 위치를 암기했다"고 관찰했지만, 평가는 학습한 7지점에 정확히 배치했으므로 **암기했다면 오히려 성공했어야 합니다.** 데이터가 가리키는 것은 암기가 아니라 애초에 구분 불가입니다 → [데이터셋 사후 분석](docs/dataset-analysis.md)

### 무엇이 남았는가

33%는 실용 수준이 아닙니다. 지점당 3회 평가로는 **지점별 편차의 유의성조차 확인할 수 없었습니다**(p=0.169). 검증하지 못한 채 남은 질문과 다음 단계는 아래 Future Work에 정리했습니다.

---

## Future Work

장비 반납으로 실행하지 못한 항목입니다. 각 항목에 **4차에서 겪은 회귀**를 근거로 한 리스크를 함께 적었습니다.

### (a) 파지 실패 복구 장면 추가 수집

**왜** — 5차 실패 14건이 전부 파지 단계이고, 지점 2·3·4에서 재시도 행동이 관찰됐습니다. 복구 시연이 있으면 이 행동을 의도적 능력으로 굳힐 수 있습니다.

**리스크** — 같은 시도를 DAgger로 했다가 [0/10으로 회귀](docs/rounds/round4.md)했습니다. 교정 데이터가 (i) 실패 상태에서 시작해 홈 포지션과 접근 구간이 한 프레임도 없고, (ii) 거의 같은 관측에 재시도·더듬기로 서로 다른 액션이 붙어 있었습니다. 행동복제 모델은 여기서 평균 액션으로 수렴합니다.

**다시 한다면** — 복구 시연도 **홈 포지션에서 시작하는 전체 에피소드**로 찍고, 복구 동작은 매번 같은 방식으로 한 번에 수행합니다. 더듬은 에피소드는 버립니다. 그리고 fine-tune이 아니라 **from scratch**로 학습합니다. 4차는 기존 체크포인트에서 40k steps를 이어 학습해 좁은 추가분에 끌려갔습니다.

### (b) 미학습 위치 커버리지 확대

**왜** — 7개 지점의 파지 자세가 데이터에서 분리되지 않았습니다(분리비 j1 1.36 / j3 1.00). 지점을 늘리는 것보다 **지점 간 간격을 넓히고 지점 내 산포를 줄이는** 것이 먼저입니다. 지금 상태로 지점을 추가하면 겹침만 심해집니다.

**구체적으로** — (i) 7지점이 베이스 회전 12.8°에 몰려 있으므로 작업공간 전체로 재배치, (ii) 이산 지점 대신 **연속 랜덤 배치**로 전환해 정책이 위치를 분류가 아닌 회귀로 학습하도록 유도, (iii) 지점당 시연 수를 늘리기 전에 배치부터 수정.

**리스크** — 위치 다양성을 늘리면 지점당 표본이 줄어 초기 성능이 오히려 떨어질 수 있습니다. 에피소드 수를 함께 늘리지 않으면 [1차의 "표본 부족"](docs/rounds/round1.md) 상태로 되돌아갑니다.

### (c) 빈 컵 / 큐브 없음 엣지 케이스 보강

**왜** — 5차에서 큐브가 없어도 탐색을 멈추지 않았습니다. "큐브 없음" 정지 데이터가 4개로 전체의 **8%** 뿐이었습니다. 2차(40%)·3차(20%)에서는 정지 동작이 정상 학습됐던 것에 비하면 회귀입니다.

**다만 근본 원인은 (b)와 같을 수 있습니다** — 큐브 위치를 신뢰성 있게 잡지 못하면 유무도 구분하지 못합니다. 정지 데이터만 늘려서는 해결되지 않을 가능성이 있습니다.

**리스크** — 정지 비율을 높이면 정지 상태로의 mode collapse가 발생합니다. 2차에서 40% → 20%로 줄인 것이 그 조치였습니다. 8% → 20% 수준까지가 안전 범위로 보입니다.

### (d) 검증하지 못한 채 남은 질문

| 질문 | 확인에 필요한 것 |
| --- | --- |
| 재시도 행동이 청크 경계(3.33초) 효과인가 | 재시도 시점이 3.33초의 배수에 정렬되는지 롤아웃 로그 확인 |
| 지점별 성공률 편차가 실재하는가 | 지점당 9회 이상 (3회로는 p=0.169) |
| `image_transforms`의 어떤 항목이 기여했는가 | 채도·색조·RandomAffine을 개별로 껐다 켜며 비교 |

---

## Hugging Face

계정: [k-chan-l](https://huggingface.co/k-chan-l)

| 라운드 | 데이터셋 | 학습된 정책 |
| --- | --- | --- |
| 1차 | [`k-chan-l/pick_and_place2`](https://huggingface.co/datasets/k-chan-l/pick_and_place2) | [`k-chan-l/pick_and_place2_act`](https://huggingface.co/k-chan-l/pick_and_place2_act) |
| 2차 | [`k-chan-l/pick_and_place`](https://huggingface.co/datasets/k-chan-l/pick_and_place) | [`k-chan-l/pick_and_place_act`](https://huggingface.co/k-chan-l/pick_and_place_act) |
| 3차 | [`k-chan-l/pick_and_place3`](https://huggingface.co/datasets/k-chan-l/pick_and_place3) | [`k-chan-l/pick_and_place3_act`](https://huggingface.co/k-chan-l/pick_and_place3_act) |
| 4차 | `k-chan-l/pick_and_place4` (3차+교정 병합) | `k-chan-l/pick_and_place4_act` |
| 4b | `k-chan-l/pick_and_place4` (4차와 동일) | `k-chan-l/pick_and_place4b_act` |
| **5차** | [`k-chan-l/pick_and_place5`](https://huggingface.co/datasets/k-chan-l/pick_and_place5) | [`k-chan-l/pick_and_place5_act`](https://huggingface.co/k-chan-l/pick_and_place5_act) |

> **이름 주의** — 데이터셋 이름의 숫자가 실험 순서와 일치하지 않습니다. **`pick_and_place2`가 1차, 숫자 없는 `pick_and_place`가 2차**, `pick_and_place3`이 3차입니다. 정책 이름도 데이터셋을 따라가므로 `pick_and_place2_act`가 1차 정책, `pick_and_place_act`가 2차 정책입니다.

## 문서

- [docs/experiments.md](docs/experiments.md) — **실험 기록 목차.** 여기서 시작하세요
- [docs/rounds/](docs/rounds/) — 라운드별 상세 기록 (설계 → 결과 → 실패 분석 → 다음 라운드 결정 근거)
- [docs/dataset-analysis.md](docs/dataset-analysis.md) — 데이터셋 사후 분석. 로봇 없이 재현 가능
- [docs/protocol.md](docs/protocol.md) — 평가 프로토콜, 실패 유형 정의, 5차 통제 조건
- [commands.md](commands.md) — 실제로 사용한 LeRobot 명령어
- [setup.md](setup.md) — udev 규칙, 카메라 MJPEG·USB 대역폭 해결 기록, 설치
- [CLAUDE.md](CLAUDE.md) — 저장소 작업 규칙과 LeRobot 관련 검증된 사실
- [media/](media/) — 롤아웃 영상
