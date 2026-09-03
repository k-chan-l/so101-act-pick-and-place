# 영상

라운드별 롤아웃·시연 영상을 둡니다. **아직 채워지지 않은 칸이 많습니다** — 없는 라운드는 나중에 해당 행을 지우면 됩니다.

## 파일 규칙

```
media/<라운드>/<라운드>_<지점>_<시행번호>.<확장자>
예: media/round5/round5_p3_t2.mp4
    media/round5/round5_overview.gif
```

- **원본 영상을 그대로 커밋하지 마세요.** GitHub 파일 제한은 100MB이고, 큰 파일은 히스토리에서 지우기 번거롭습니다
- README에 넣을 것은 **5MB 아래 GIF**로 압축합니다
- 학습용 시연 영상 원본은 HF 데이터셋 안에 있습니다 (`videos/` 디렉터리). 여기 복사하지 말고 링크로 대체하세요

```bash
# mp4 -> GIF 압축 예시
ffmpeg -i input.mp4 -vf "fps=10,scale=480:-1:flags=lanczos" -t 4 output.gif
```

## 현황

| 라운드 | 성공률 | 영상 | 상태 |
| --- | --- | --- | --- |
| [1차](../docs/rounds/round1.md) | 0% | [`round1/`](round1/) | 미기록 |
| [2차](../docs/rounds/round2.md) | 2/10 | [`round2/`](round2/) | 미기록 |
| [3차](../docs/rounds/round3.md) | 4/20 | [`round3/`](round3/) | 미기록 |
| [4차](../docs/rounds/round4.md) | 0/10 | [`round4/`](round4/) | 미기록 |
| [4b](../docs/rounds/round4b.md) | 0/10 | [`round4b/`](round4b/) | 미기록 |
| [5차](../docs/rounds/round5.md) | **7/21** | [`round5/`](round5/) | 미기록 (휴대폰 촬영분 있음) |

## 만들 그림 (미완)

| 우선순위 | 그림 | 만드는 법 | 들어갈 곳 |
| --- | --- | --- | --- |
| 1 | **7지점 분리도 산점도** — j1×j3 평면에 42개 파지 지점을 지점별 색으로 | [dataset-analysis.md](../docs/dataset-analysis.md)의 파지 검출 코드에 플롯 추가 | `docs/dataset-analysis.md` 3절 |
| 2 | **라운드별 성공률 + 실패 유형 누적 막대** | [experiments.md](../docs/experiments.md) 요약표 수치 | `README.md` 결과 |
| 3 | **태스크 소개 GIF** — `cam_top`+`cam_wrist` 2~3초 | 휴대폰 영상 또는 데이터셋 `videos/` | `README.md` 최상단 |

1번이 이 프로젝트의 핵심 발견(분리비 1.36)을 표보다 잘 보여줍니다.
