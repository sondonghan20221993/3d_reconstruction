# 현재 작업 현황 (2026-06-22)

> 새 세션에서 이 파일만 읽으면 지금 뭘 하고 있는지 파악 가능.

## 현재 상태: GOF 학습 중 + 포즈 정확도 평가 완료

- **진행 중**: `real_test__gof` — GOF (Gaussian Opacity Fields, NeurIPS2024) 학습 중 (30000 iter)
- **완료**: `real_test__milo_sor2` — 18000 iter 완료, `mesh_learnable_sdf.ply` (215MB) 추출 완료
- **완료**: GT vs SfM vs IMU 포즈 정확도 비교 (ATE/RPE, 34 frames)
- blue_1은 단색 박스로 막힘 상태 유지.

---

## outlier 제거 실험 (2026-06-22 착수)

### 배경: 직접-위상 방법(MILo/SuGaR/AGS)의 잡음 원인 분석

TSDF 기반(2DGS)은 멀티뷰 평균화로 outlier에 강건하지만, 직접-위상 방법들은 각기 다른 이유로 outlier에 취약:

- **MILo**: 학습 중 매 iteration마다 Gaussian → SDF → 미분 가능한 메시 추출 → occupancy/normal supervision에 사용. outlier가 학습 전 과정 내내 메시 위상을 오염시킴. `mesh_extract_sdf.py`는 학습 후 최종 export만 담당.
- **SuGaR/AGS**: 학습 완료된 Gaussian에서 별도 후처리로 메시화 → outlier가 Gaussian 분포를 왜곡해 결과 메시를 깨뜨림.

가설 검증:

| 가설 | 판정 | 근거 |
|---|---|---|
| H1: 스케일/좌표계 깨짐 | ❌ 기각 | scene_extent real_test 1.99 / blue_1 4.24 = **정상 범위** (100↑/0.1↓ 아님) |
| H2: low-confidence outlier 포인트 | ✅ **핵심 원인** | 점군이 scene_extent의 **4.9x(real_test)/4.6x(blue_1)**까지 퍼짐. MILo "잡음 많음"의 직접 원인 |
| H3: imp_metric outdoor/indoor 미스매치 | ⏸ 미확인 | MILo config 확인 필요 |

### scene extent 측정값 (COLMAP 변환 기준)

| 항목 | real_test | blue_1 |
|---|---|---|
| 카메라 diagonal | 5.14 | 9.06 |
| scene_extent (3DGS radius) | 1.99 | 4.24 |
| points3D full diag | 15.84 | 29.77 |
| points3D p1–p99 diag | 9.84 | 19.39 |
| 점군/extent 비율 | 4.93x | 4.57x |

### 기존 시도 여부 확인
- **기하 outlier 제거(SOR/bbox crop)는 MILo/SuGaR에 적용한 적 없음** (real_test MILo는 raw 1.49M 점군 그대로 사용)
- 존재하던 `mast3r_color_filter.py`는 파란색 추출→Poisson(blue_1 전용), 기하 필터 아님

### 결정: SOR만 적용 (환경 최대 보존 목적, bbox crop 제외)

bbox crop은 주변 환경(지면)까지 잘릴 수 있어 제외. SOR은 공간 경계 없이 고립 floater만 제거 → 환경 보존.

| SOR 설정 | 보존 | full diag | p1–p99 |
|---|---|---|---|
| (원본) | 100% | 15.84 | 9.84 |
| **nb=20, std=2.0 (채택)** | **94.6%** | 12.79 | ~7.98 |
| nb=20, std=1.5 | 92.0% | 11.61 | 7.28 |
| nb=30, std=1.0 | 88.0% | 9.53 | 6.30 |

- 1차 시도: `nb_neighbors=20, std_ratio=2.0` (80,414 pts 제거, 94.6% 보존). 로컬 `real_test_pointcloud_sor.ply` (37MB)
- outlier 위치 확인: `real_test_SOR_outlier_비교.png` → 제거점이 전부 바깥 성긴 가장자리(고립 floater)에 분포 확인

### 검증 실행 결정 (2026-06-22)

효과 유무를 빠르게 확인하기 위해 **강한 SOR + 가장 적합한 모델**로 1회 테스트:

- **SOR 강도**: `nb=30, std=1.0` (88% 보존, full diag 15.84→9.53) — 강하게 적용해 효과 명확히 확인
- **테스트 모델**: **MILo** 선정
  - 이유: ① baseline 명확("형상 양호, 잡음 많음") ② 학습 중 매 iter마다 미분 가능한 메시 추출 → outlier가 학습 전체에 걸쳐 위상 오염 → 가장 민감하여 SOR 효과 가장 뚜렷 ③ 별도 TSDF 없이 learnable SDF에서 직접 메시 export ④ SuGaR보다 빠름(~30분)
- **복원 속도 순위**: 2DGS(robust, 제외) > MILo(~30분) > AGS(depth prep+TSDF) > SuGaR(다단계, 최저)
- **파이프라인**: points3D.txt를 SOR로 필터 → `real_test__mast3r_clean/02_colmap` → MILo 재실행(`real_test__milo_sor`) → 기존 MILo와 비교

### SOR 실험 결과 요약

| 항목 | MILo baseline | MILo SOR1 (nb=30, std=1.0) | MILo SOR2 (nb=30, std=0.5) |
|---|---|---|---|
| 입력 pts | 1,490,920 | 1,308,980 (88%) | 1,218,707 (81.7%) |
| 출력 파일 | `real_test_milo.ply` (288MB) | `real_test_milo_sor.ply` (221MB) | `real_test_milo_sor2.ply` (215MB) ✅ |
| Vertices | 7,203,794 | 5,534,552 (-23%) | 측정 필요 |
| bbox diagonal | 15.20 | 12.35 (-19%) | 측정 필요 |
| 메시 품질 | 가장자리 스파이크 심각 | 가장자리 스파이크 여전히 존재 | 평가 예정 |
| 학습 시간 | — | — | 18000 iter / 6828초 (~114분) |

- **SOR1 결과**: 수치 개선됐으나 3D 뷰어에서 여전히 가장자리 파편 다수 → std=0.5로 강화
- **SOR2 결과**: ✅ 완료. 215MB PLY 로컬 저장 (`현재결과/03_Mesh/sor/real_test_milo_sor2.ply`)
- outlier 위치 시각화: `real_test_SOR_outlier_topview.png` (단일 상단뷰, RGB 컬러 포인트)

### GOF (Gaussian Opacity Fields) — 진행 중

- **목적**: unbounded 야외 씬 특화 최신 모델 (NeurIPS 2024) 적용
- **상태**: ✅ 학습 중 (`tmux gof`, 30000 iter)
- **입력**: SOR2 필터링된 COLMAP (`real_test__milo_sor2/colmap/`, 1,218,707 pts)
- **출력 예정**: `real_test__gof/mesh_0030000.ply`
- **로그**: `/home/sdh/Desktop/gof_run.log`

---

## 포즈 정확도 비교 — GT vs MASt3R-SfM vs IMU (2026-06-22)

### 데이터
- **GT**: AirSim meta JSON `camera.pose` (ground truth, 오차 없음) — 34프레임
- **SfM**: COLMAP `images.txt` → world 위치 복원 (R^T(-t))
- **IMU**: Euler preintegration (gravity 보정, 초기값=GT frame 0)

### 결과

| 방법 | ATE (m) | RPE (m) | 비고 |
|---|---|---|---|
| MASt3R-SfM | **0.023** | **0.024** | Umeyama 정렬 후 (scale 3.89x) |
| IMU preintegration | 6.23 | 2.39 | 초기값 GT, drift 누적 |

> 좌표계: AirSim NED (m), 드론 고도 약 5m (Z ≈ -5)  
> GT trajectory span: 18.88m (X×Y≈13m씩 이동)  
> SfM scale factor 3.89: COLMAP arbitrary unit → meter 변환  
> **결론: MASt3R-SfM 위치 오차 2.3cm (매우 정확), IMU는 6.2m drift 누적**

> ⚠️ 버그 수정 이력: Umeyama `.mean()`→`.sum()/n` (scale 3x 과대추정 → ATE 13m 오류)

- **궤적 비교 플롯**: `현재결과/pose_comparison_imu.png` (상단뷰 + 정면뷰)
- **스크립트**: `/tmp/pose_eval2.py`

---

## real_test 메시 복원 실험 이력 (2026-06-21 완료)

> 데이터: `~/Desktop/data/datasets/rgb/` (34장, 1920×1080, 시뮬레이션)
> 공통 초기화: MASt3R-SfM 점군 1,490,920 pts (전부 동일 PLY 사용 확인)

### Baseline (depth/normal supervision 미적용)

| 방법 | 결과 | 출력 | 로컬 다운로드 |
|---|---|---|---|
| 2DGS (baseline) | ✅ 완료 | `real_test__mast3r__2dgs/.../fuse_post.ply` 6.4M V | `C:\Users\sdh97\Desktop\real_test_2dgs_fuse_post.ply` (315MB) |
| MILo (baseline) | ✅ 완료 | `real_test__milo/mesh_learnable_sdf.ply` | `C:\Users\sdh97\Desktop\real_test_milo.ply` (288MB) |
| AGS-Mesh (baseline) | ✅ 완료 | `real_test__ags_mesh_output/.../fuse_post.ply` 3.5M V | `C:\Users\sdh97\Desktop\real_test_ags_fuse_post.ply` (165MB) |

### Prior 버전 (MASt3R depth map supervision 적용)

| 방법 | 결과 | 출력 | 평가 |
|---|---|---|---|
| MILo + MASt3R depth prior | ✅ 완료 | `real_test__milo_prior/mesh_learnable_sdf.ply` | `C:\Users\sdh97\Desktop\real_test_milo_prior.ply` (268MB) — **전체 형상 양호, 잡음 많음** |
| AGS-Mesh + MASt3R depth prior | ✅ 완료 | `real_test__ags_prior_output/.../fuse_post.ply` 3.1M V | `C:\Users\sdh97\Desktop\real_test_ags_prior_fuse_post.ply` (152MB) — **평가 필요** |

### Prior 구현 내역

- MASt3R depth maps: `~/Desktop/data/experiments/real_test__mast3r_depth/depth_maps/` (`.npy`, float32, m)
- **MILo 패치** (`~/Desktop/milo/milo/regularization/regularizer/depth_order.py`):
  - `initialize_depth_order_supervision()` 상단에 MASt3R `.npy` 로딩 경로 추가
  - config key `mast3r_depth_dir` 존재 시 `.npy` 로드 후 [0,1] 정규화 → depth order supervision에 사용
  - 없으면 기존 DepthAnythingV2 경로 fallback
- **MILo config** (`~/Desktop/milo/milo/configs/depth_order/mast3r_depth.yaml`):
  - `mast3r_depth_dir: '/home/sdh/Desktop/data/experiments/real_test__mast3r_depth/depth_maps'`
  - `weight_update_iters: [600, 1500, 3000, 8000, 13000]` / `weight_update_values: [1, 0.1, 0.01, 0.001, 0.0001]`
- **AGS-Mesh**: MASt3R `.npy` → uint16 PNG (mm) 변환 후 `--depth_supervision`, TSDF `voxel_size=0.004 sdf_trunc=0.02`

### 주의사항 (재실행 시 참고)
- AGS input 디렉토리는 `colmap/` 심링크 필수 (없으면 "Could not recognize scene type!" 에러)
- MASt3R depth format: `.npy` float32 meters. AGS는 uint16 PNG mm 필요 → `(depth * 1000).astype(np.uint16)`
- AGS normal: dummy flat (2DGS normal 저장이 주석처리됨, `utils/mesh_utils.py:294`)

---

## blue_1 현재 상태: 막힘

단색 파란 박스 → 모든 접근법 실패. 근본 원인은 아래 참조.

---

## 근본 실패 구조

```
단색 파란 박스 → texture gradient 없음
    ├─ feature matching 실패 → SfM 희소 포인트 → Poisson 불완전
    ├─ photometric loss ≈ 0 → GS 계열 전부 학습 실패 (2DGS/SuGaR/MeshSplatting)
    └─ depth map 노이즈 심함 → TSDF floater 과다 (AGS-Mesh / MASt3R TSDF)
```

Gaussian splatting을 시도한 이유도 바로 이 때문이었으나, photometric gradient가 없어 학습이 안 됨.

---

## blue_1 메시 복원 실험 이력

| 방법 | 결과 | 실패 원인 |
|---|---|---|
| Screened Poisson (720p) | ❌ | 저해상도 노이즈 |
| Screened Poisson (FHD) | ⚠️ 형태 식별 | 지면 격자 아티팩트 |
| PPSurf | ❌ | 입력 점군 품질 한계 |
| SuGaR | ❌ | photometric gradient 부족 |
| GOF | ❌ | CUDA 11.3 불일치 |
| MILo | ❌ | 1.23M pts → CUDA kernel limit |
| MeshSplatting CVPR2026 | ❌ | photometric loss ≈ 0 |
| 2DGS BASE + dense prior | ✅ 학습 | 메시 미추출 |
| 2DGS CONTROL + dense prior | ✅ 학습 | TSDF floater 심각 |
| 2DGS TSDF (render.py) | ❌ | 559MB, floater 과다 |
| AGS-Mesh-2dgs | ⚠️ 완료 | TSDF 여전히 noisy (13M V) |
| MASt3R direct TSDF | ⚠️ 완료 | 16.7M V, 클러스터 후 noisy |
| MASt3R 컬러 필터 → Poisson (loose) | ❌ | gray점 혼입, 17K V |
| MASt3R 컬러 필터 → Poisson (strict) | ⚠️ 완료 | 법선 방향 불안정 |
| MASt3R 컬러 필터 → Poisson (카메라 정렬) | ⚠️ 완료 | blue_box_poisson2.ply, 평가 중 |

### 컬러 필터 결과 (가장 최근)

- 박스 점군: `blue_box_cluster.ply` (12,374 pts, 1cm 간격, 전면 커버)
- OBB 직육면체: `blue_box_obb.ply` (0.616×0.587×0.222m)
- Poisson (카메라 정렬): `blue_box_poisson2.ply` (2.4MB) — 평가 필요
- 필터 조건: `B>0.40 AND B>R+0.25 AND B>G+0.15`

---

## 남은 선택지

| 방법 | 설명 | 가능성 |
|---|---|---|
| **BPA** | Ball Pivoting. 법선 불필요, 있는 점만 연결 | ⭐ 시도 안 함 |
| **OBB 직육면체 대체** | 정확한 형상 아니지만 크기/위치 정확 | ⭐ 이미 있음 |
| structured light / depth sensor | 소프트웨어 한계 돌파 | 하드웨어 필요 |

---

## AGS-Mesh-2dgs 결과 (참고용)

- 학습: 30000 iter 완료 (2026-06-20 00:00 KST)
- 메시: `~/Desktop/data/experiments/blue_1__ags_mesh_output/train/ours_30000/fuse_post.ply`
  - 13M V, 24M F → 단순화 후 `ags_mesh_simplified.ply` (50만 F, 14MB)
- 다운로드: `C:\Users\sdh97\Desktop\ags_mesh_simplified.ply`
- 다운로드: `C:\Users\sdh97\Desktop\ags_mesh_fuse_post.ply.gz` (246MB)

### 코드 패치 내역 (dn-splatter)

- `scene/dataset_readers.py`: `load_every=5` → `load_every=1`
- `utils/camera_utils.py`: depth/normal/confidence F.interpolate 리사이즈
- `utils/general_utils.py`: `tostring_rgb()` → `buffer_rgba()` (matplotlib 호환)

---

## MASt3R direct TSDF 결과 (참고용)

- 스크립트: `~/Desktop/mast3r_tsdf_fusion.py`
- 결과: `~/Desktop/mast3r_tsdf_mesh.ply` (1.2GB), 클리닝 후 `mast3r_tsdf_clean.ply`
- 다운로드: `C:\Users\sdh97\Desktop\mast3r_tsdf_clean.ply` (20MB)
- 파라미터: voxel=1cm, sdf_trunc=5cm, 125장 fusion

---

## 완료된 환경

| venv/conda | 상태 | 용도 |
|---|---|---|
| `venv-mast3r` | ✅ | MASt3R-SfM |
| `venv-2dgs` | ✅ | 2DGS (Python 3.10) |
| `venv-meshsplat10` | ✅ | MeshSplatting CVPR2026 |
| `conda ags_mesh` | ✅ | AGS-Mesh-2dgs (Python 3.10, PyTorch 2.1.2) |
| `venv-gof` | ✅ | GOF (NeurIPS2024, numpy 1.26.4, setuptools 69.5.1) |
| `/usr/bin/python3` | ✅ | open3d 0.13.0 (시스템) |

- **주의**: 환경 전환 시 `unset LD_LIBRARY_PATH` 필수
- `venv-pointmap`: python3.10이 venv-sdf 심링크 → 깨진 상태
- conda `recon3d`: PIL/libstdc++ 손상, 사용 불가

---

## 유틸리티 스크립트

| 스크립트 | 위치 | 용도 |
|---|---|---|
| `run_mast3r_sfm_with_depth.py` | `~/Desktop/MAST3R_2/` | MASt3R-SfM + depth map 저장 |
| `prepare_ags_data.py` | `~/Desktop/` | AGS-Mesh 데이터 디렉토리 준비 (blue_1용) |
| `prepare_ags_data_realtest.py` | `~/Desktop/` | AGS-Mesh 데이터 디렉토리 준비 (real_test용) |
| `mast3r_tsdf_fusion.py` | `~/Desktop/` | MASt3R depth → TSDF mesh |
| `mast3r_color_filter.py` | `~/Desktop/` | 파란 점 필터 → Poisson mesh |
| `blue_box_obb.py` | `~/Desktop/` | 컬러 필터 + OBB + Poisson |
| `inference_realesrgan.py` | `~/Desktop/Real-ESRGAN/` | 720p → FHD 업스케일 |
| `run_real_test_mesh.sh` | `~/Desktop/` | 2DGS→AGS prep→MILo→AGS 순차 실행 (real_test baseline)

---

## 데이터셋

| 객체 | 장수 | 상태 |
|---|---|---|
| blue_1 | 125 | ⚠️ 막힘 — blue_box_poisson2.ply 평가 후 BPA 또는 OBB 대체 고려 |
| blue_2 | 106 | ⬜ 대기 |
| streetlight_low | 75 | ⬜ 대기 |
| blue_person | 38 | ⬜ 대기 |
