# 현재 작업 현황 (2026-06-17)

> 새 세션에서 이 파일만 읽으면 지금 뭘 하고 있는지 파악 가능. 상세 배경은 `pipeline_strategy_3branches.md` 참고.

## 진행 중인 프로세스

| 터미널 | venv | 작업 | 상태 |
|---|---|---|---|
| 터미널 A | `venv-sdf` | Neuralangelo 학습 (`real_test`, 33장) | 🔄 진행 중, 500k iter 중 ~152k 완료, 총 예상 ~29시간 (06-16 낮1시 시작 → 06-17 저녁 완료 예상) |
| 터미널 B | `venv-mast3r` → `venv-gof` | blue_1 SfM + COLMAP 변환 → GOF 학습(2-1, 2-2) | ✅ SfM/COLMAP 완료, GOF 대기 |

## 신규 데이터셋 (SYMA Z3 드론, 2026-06-16)

| 객체 | 장수 | rgb 폴더 | SfM 상태 |
|---|---|---|---|
| blue_1 | 125 | `~/Desktop/data/datasets/blue_1/rgb/` | ✅ SfM(125개 카메라, 2.3M점) / COLMAP 변환 완료 → GOF 학습 예정 |
| blue_2 | 106 | `~/Desktop/data/datasets/blue_2/rgb/` | ⬜ 대기 (blue_1 완료 후) |
| streetlight_low | 75 | `~/Desktop/data/datasets/streetlight_low/rgb/` | ⬜ 대기 |
| blue_person | 38 | `~/Desktop/data/datasets/blue_person/rgb/` | ⬜ 대기 (장수 부족, 참고용) |

## 다음 단계 (순서대로)

1. **MASt3R-SfM 4개 순차 실행** (venv-mast3r, `--scene_graph complete --shared_intrinsics`) → 각 `~/Desktop/data/experiments/{object}__mast3r/01_sfm/`에 poses.npy/focals.npy/pointcloud.ply
2. **COLMAP 변환** (`mast3r_to_colmap.py`) → `02_colmap/`
3. **GOF 학습** (venv-gof): 객체별로 2-1(베이스라인, COLMAP sparse init) / 2-2(MASt3R prior 통합) 두 버전
4. (이후) 점맵 baseline 메싱 (venv-pointmap), 필요시 Neuralangelo도 신규 객체에 적용

## 유틸리티 스크립트

### `visualize_reconstruction.py` — SfM 결과 시각화

MASt3R-SfM / DUSt3R 출력물(`pointcloud.ply`, `poses.npy`, `focals.npy`)을 trimesh로 3D 시각화.

**기능**
- PLY 포인트클라우드를 씬에 로드
- 카메라 포즈를 **방향 화살표**로 표시 (몸통 cylinder + 머리 cone)
- 화살표 색상: **파랑(첫 프레임) → 빨강(마지막 프레임)** 그라디언트로 촬영 순서 표시
- 카메라 위치를 이어주는 **궤적 선** 표시
- trimesh 인터랙티브 뷰어로 회전/줌/이동 가능

**의존성**
```
pip install trimesh numpy
```

**사용법**
```bash
# 출력 폴더 지정 (pointcloud.ply / poses.npy / focals.npy 자동 탐색)
python visualize_reconstruction.py --output_dir ./ground_first_output

# 파일 개별 지정
python visualize_reconstruction.py --ply a.ply --poses poses.npy --focals focals.npy
```

---

## 완료된 환경 구축

- `venv-mast3r` / `venv-gof` / `venv-sdf` / `venv-pointmap` 4개 모두 ✅ 완료 (시스템 CUDA 12.3 기반 순수 venv)
- conda `recon3d`는 torch/numpy 파일 손상으로 사용 중단, 보존만 함
- **주의**: 환경 전환 시 `unset LD_LIBRARY_PATH` 필수 (안 하면 torch CUDA 깨짐 — `pipeline_strategy_3branches.md` 공통주의사항 참고)
