# 현재 작업 현황 (2026-06-29)

> 새 세션에서 이 파일만 읽으면 지금 뭘 하고 있는지 파악 가능.

## 현재 상태: 실제 드론 데이터 복원 진행 중 (2026-06-29)

---

## 7. 실제 드론 데이터 복원 (2026-06-29)

### 데이터

| 항목 | 값 |
|---|---|
| 드론 | DJI, 실제 야외 비행 |
| 고도별 폴더 | `3m_1` (55장), `5m_1` (55장), `7m_1` (57장) |
| 해상도 | 1920×1080 |
| GT | ❌ 없음 (시각적 평가만 가능) |
| 서버 | sysai3, `/home/sdh/Desktop/data/drone_real/` |

### 실험 이력

#### ① 3m+7m combined (순차 순서) — 박스 분리 문제 발견

| 항목 | 값 |
|---|---|
| 이미지 | 3m_1(55) + 7m_1(57) = 112장, 프리픽스 `3m_frame_`, `7m_frame_` |
| MASt3R-SfM | ✅ 완료 → `drone_real_sfm/combined_3m7m/` (focal=1364.30px) |
| COLMAP 변환 | ✅ 완료 → `drone_real_combined__colmap/` |
| 2DGS 학습 | ✅ 완료 (30k iter) → `drone_real_combined__2dgs/` |
| 메시 추출 | ✅ 완료 → `fuse_post.ply` (96MB, 1.95M V) |
| **문제** | **박스가 두 개로 분리** — 3m↔7m inter-orbit scale mismatch |

> 원인: swin-5 그래프가 순차 정렬 시 경계 1곳만 cross-altitude 연결 →  
> 3m 그룹과 7m 그룹이 서로 다른 스케일로 복원 → 동일 박스가 2개로 보임.  
> 시뮬레이션에서도 7m ATE=27.6cm로 이미 확인된 구조적 문제.

#### ② 3m+5m combined (순차 순서) — 동일 문제

| 항목 | 값 |
|---|---|
| 이미지 | 3m_1(55) + 5m_1(55) = 110장 |
| MASt3R-SfM | ✅ 완료 → `drone_real_sfm/combined_3m5m/` |
| **문제** | **박스 분리 동일** — 고도 차 2m도 순차 정렬 시 동일 증상 |

#### ②-1 5m_1 단독 GS-2M 추가 (2026-06-30)

| 항목 | 값 |
|---|---|
| 배경 | 시뮬 4m에서 GS-2M이 큐브 세밀/조밀(F@1cm·vertex 6.7×) 우위 → 실제 5m도 검증 |
| 학습 | ✅ GS-2M 30k 완료 (PSNR 29.33) → `drone_real_5m_1__gs2m` |
| 메시 | ✅ `tsdf_post.ply` (152M, 3.0M V) / 2DGS는 기존 `fuse_post.ply` (27M) |
| 평가 | ❌ GT 없음 → 시각 비교만 (박스 영역 조밀도 확인) |
| 다운로드 | 로컬 `Desktop/real_5m_meshes/` (2DGS, GS-2M) |
| 주의 | 5m 고도 → 박스 이미지 점유 작음·측면 미관측 → GS-2M 3M V 중 지면/배경 floater 다수 가능 |

#### ③ 3m+5m 인터리빙 (현재 진행 중) — 해결 시도

| 항목 | 값 |
|---|---|
| 핵심 아이디어 | 같은 각도의 3m·5m 이미지를 교대 배치 → swin-5가 매 쌍마다 cross-altitude 연결 생성 |
| 이미지 순서 | `img_0000_3m, img_0001_5m, img_0002_3m, img_0003_5m, ...` (110장) |
| 경로 | `drone_real/combined_3m5m_interleaved/` |
| MASt3R-SfM | ⏳ 진행 중 (tmux: mast3r_3m5m) |
| 2DGS | ⏳ SfM 완료 후 즉시 시작 예정 |

> **검증 방법**: pointcloud.ply 또는 fuse_post.ply를 CloudCompare로 열어 박스가 하나로 합쳐지는지 확인.

### 다운로드 현황 (로컬 바탕화면)

| 파일 | 경로 | 내용 |
|---|---|---|
| `sfm_poses.npy` | `Desktop/` | 3m+7m combined 포즈 |
| `sfm_pointcloud.ply` | `Desktop/` | 3m+7m SfM 점군 |
| `sfm_3m5m/` | `Desktop/sfm_3m5m/` | 3m+5m combined SfM 결과 |
| `simulation_3m/`, `simulation_7m/` | `Desktop/` | 시뮬레이션 이미지 34장 |
| `simulation_sfm/` | `Desktop/simulation_sfm/` | 시뮬레이션 MASt3R-SfM 결과 |
| `2dgs_combined/fuse_post.ply` | `Desktop/2dgs_combined/` | 3m+7m 2DGS 메시 (96MB) |

---

---

## 8. 시뮬레이션 4m old 임시 검증 실험 (2026-06-29)

> ⚠️ **임시 실험**: 기존 시뮬레이션(3m+7m)에 5m 고도 데이터가 없어, old `real_test` 데이터 중
> **frame 0-16 (~4.3m, 17장)** 을 현재 파이프라인으로 재실행. AirSim/Unreal 씬 동일 → GT 메시 재사용 가능.
> 향후 AirSim에서 5m 균일 촬영 시 이 실험으로 대체 예정.

### 실험 조건

| 항목 | 값 |
|---|---|
| 데이터 | old `real_test` frame 0-16, 고도 ~4.3m, **17장** |
| 이미지 경로 | `datasets/real_test_4m_old/rgb/` (000000~000016.png) |
| 해상도 | 1920×1080 |
| GT 메시 | 동일 (`gt_scene_clean.ply`, `gt_cubes.ply`) |
| prior | **① ply 초기화** 전 모델 공통 적용 (MASt3R pointcloud) |
| 외부 depth | ❌ 없음 (공정 비교) |

### 포즈 품질 (Umeyama Sim3, 2026-06-29)

| 항목 | 4m old (17장) | 3m uniform (참고) |
|---|---|---|
| ATE RMSE | **0.029 cm** | 0.76 cm |
| ATE Max | 0.095 cm | — |
| RPE 회전 Mean | 0.58° | 0.23° |
| Scale | 3.865 | 3.08 |

> 단일 고도 단독 실행 → scale mismatch 없음 → ATE 매우 우수.

### 학습 진행 상황 (2026-06-29 갱신)

| 모델 | 환경 | 상태 |
|---|---|---|
| **2DGS** | venv-2dgs | ✅ 학습+메시 완료 |
| **3DGS** | miniforge3/gs3d | ✅ 학습 완료 |
| **GS-2M** | miniforge3/gs2m | ✅ 학습(30k, PSNR 31.74) + 메시 완료 |
| **MILo** | venv-milo | ✅ 학습(18k) + 메시 완료 |

> **메시 산출물 (2026-06-29, 4모델 전부 완료)**
> - GS-2M: `real_test_4m_old__gs2m/train/ours_30000/mesh/tsdf_post.ply` (193M, 3.94M V)
> - MILo: `real_test_4m_old__milo/mesh_learnable_sdf.ply` (146M, vertex color 포함)
> - 메시 추출 명령: GS-2M `render.py -m <out> --extract_mesh --skip_test` / MILo `mesh_extract_sdf.py -s <colmap> -m <out> --rasterizer radegs` (milo/milo/ 에서)
> - 메시 추출 명령(추가): 2DGS `render.py -m <out> -s <colmap> --voxel_size 0.01 --depth_trunc 6.0 --sdf_trunc 0.04 --num_cluster 1` / 3DGS `/tmp/run_3dgs_tsdf_4m.py` (open3d TSDF)

### 메시 CD/F-score 평가 결과 (2026-06-29, GT 큐브 Cube8-12 기준)

> 정렬: MASt3R poses ↔ GT meta(NED) **Umeyama Sim3 1회** (단일 17장, ATE 1.86cm, s=3.865).
> 평가 스크립트: `/tmp/eval_mesh_4m_old.py` (combined_uniform 평가의 4m old 버전).
> GT를 COLMAP 공간으로 역변환 후 큐브 주변 crop → CD/F@thr (threshold는 COLMAP scale 적용).

| 순위 | 모델 | CD↓ | F@1cm↑ | F@5cm↑ | F@10cm↑ |
|---|---|---|---|---|---|
| 🥇 | **2DGS** | **3.63cm** | 0.2483 | **0.5408** | **0.8556** |
| 🥈 | **GS-2M** | 3.69cm | 0.2743 | 0.5280 | 0.8496 |
| 🥉 | **MILo** | 3.77cm | **0.2746** | 0.5284 | 0.8350 |
| — | 3DGS | 측정불가 | — | — | — |

> **3DGS 측정불가**: TSDF 메시가 GT 큐브 영역 crop 후 표면 0개 → 큐브 복원 실패(floater/지면 위주).
> combined_uniform에서도 3DGS 환경 F@10=0.000 → **동일 현상**.

#### 기존 3m+7m(combined_uniform) 물체 결과와 비교

| 모델 | 4m old CD | 4m F@10 | combined 물체 CD | combined 물체 F@10 |
|---|---|---|---|---|
| 2DGS | 3.63cm | 0.856 | (최고) | 0.914 |
| GS-2M | 3.69cm | 0.850 | 5.15cm | 0.789 |
| MILo | 3.77cm | 0.835 | 7.32cm | 0.659 |
| 3DGS | 실패 | — | 실패 | 0.000 |

> **결론: 순위·경향 기존과 동일** — 2DGS ≥ GS-2M > MILo ≫ 3DGS, 3DGS 메시 붕괴도 재현.
> **차이점**: 4m 단독(17장)은 scale mismatch가 없어 CD 절대값이 전반적으로 더 낮고(3.6~3.8cm),
> **모델 간 격차가 매우 작음**(0.14cm 차). combined는 3m↔7m inter-orbit 오차로 모델 간 격차가 컸음(5→7cm).
> → 데이터 품질(단일고도 균일)이 좋을수록 모델 선택의 영향이 줄어든다는 점을 정량 확인.

#### 박스(큐브)만 tight crop 결과 (±5cm, 지면 제외, 2026-06-30)

> 스크립트: `/tmp/eval_box_crop_4m.py`. GT 큐브 5개 각각 bbox±5cm만 pred에서 crop → CD/F-score.
> 지면 floater 제외로 큐브 복원 정확도만 순수 측정.

| 모델 | CD↓ | F@1cm↑ | F@5cm↑ | F@10cm↑ |
|---|---|---|---|---|
| **2DGS** | **2.11cm** | 0.3850 | **0.7893** | 1.000 |
| **GS-2M** | 2.14cm | **0.4324** | 0.7793 | 1.000 |
| **MILo** | 2.13cm | 0.4309 | 0.7881 | 1.000 |
| 3DGS | 실패(0개) | — | — | — |

> **해석**: CD는 2DGS가 미세하게 유리(floater 없음), F@1cm는 GS-2M이 높음(조밀 vertex → 근접 커버리지 우위).
> CD와 F@1cm의 상충은 각 지표의 특성 차이: CD는 outlier(먼 점)에 민감, F@1cm는 근접 커버리지에 민감.

#### 논문 DTU 벤치마크 CD vs 우리 시뮬 4m 비교

| 모델 | 우리 4m CD (박스 crop) | 논문 DTU CD | 씬 조건 |
|---|---|---|---|
| **2DGS** | **2.11cm** | **~0.48mm** | DTU: 텍스처 풍부, 49+장, 물체 스케일 |
| **GS-2M** | 2.14cm | ~0.80mm | DTU: GS-2M 논문 자체 보고 |
| **MILo** | 2.13cm | ~0.62mm | DTU 기준 추정 |
| **PGSR** | 미실험 | 0.52mm | DTU 최고 성능 참고값 |

> **절대값 차이(~25×)**: DTU는 cm 스케일 물체·정밀 3D 스캐너 GT. 우리는 4m 고도 항공 씬·17장. 씬 스케일 상이 → 직접 비교 불가.
> **순위 비교**: 2DGS ≥ GS-2M 경향은 DTU·우리 씬 모두 유사. 단, 우리 씬은 저텍스처+sparse view로 GS-2M BRDF 분해가 ill-posed → 모델 간 격차가 DTU보다 훨씬 작음(0.03cm 차).

### 실행 명령 (4m old — 재현용)

```bash
# GS-2M (30k)
source /home/sdh/miniforge3/bin/activate gs2m && \
python3 /home/sdh/Desktop/models/GS-2M/train.py \
  -s /home/sdh/Desktop/data/experiments/real_test_4m_old__colmap \
  -m /home/sdh/Desktop/data/experiments/real_test_4m_old__gs2m \
  --iterations 30000 --port 6012

# MILo (18k, outdoor+radegs 필수, milo/milo/ 에서 실행)
source /home/sdh/Desktop/venvs/venv-milo/bin/activate && cd /home/sdh/Desktop/models/milo/milo && \
python3 train.py \
  -s /home/sdh/Desktop/data/experiments/real_test_4m_old__colmap \
  -m /home/sdh/Desktop/data/experiments/real_test_4m_old__milo \
  --iterations 18000 --port 6013 --imp_metric outdoor --rasterizer radegs
```

> ⚠️ 주의: MILo는 `--imp_metric outdoor --rasterizer radegs` 빠뜨리면 안 됨.
> COLMAP/PLY prior 경로: `real_test_4m_old__colmap`, `real_test_4m_old__mast3r/pointcloud.ply`.

---

# 🎯 최종 시뮬레이션 환경 검증 실험 계획 (2026-06-28)

> **위치 부여**: 이 실험은 **시뮬레이션(AirSim/Unreal) 환경에서의 마지막 검증**이다.
> 여기서 MASt3R-SfM 기반 pose-free 파이프라인(포즈 추정 → ply 초기화 → 3D 복원)이
> 시뮬레이션 GT 대비 정량적으로 검증되면, 이후 실제 드론 데이터로 넘어간다.
> 실제 드론에는 GT가 없으므로, **GT로 정량 검증이 가능한 것은 이 시뮬레이션 단계가 마지막**이다.

## 0. 데이터셋 / 환경

| 항목 | 값 |
|---|---|
| **데이터셋** | `real_test_3m_uniform`(17장) + `real_test_7m_uniform`(17장) = **34장** ← **최종 확정** |
| Google Drive | `1-RAGc9v-JDg6tcDdHrCDLmn3zTxm0smA` (`real_test_3-7m_uniform.zip`, 135MB) |
| 서버 경로 | `/home/sdh/Desktop/data/datasets/real_test_3m_uniform/`, `real_test_7m_uniform/` |
| 해상도 | 1920×1080 |
| 출처 | AirSim/Unreal 시뮬레이션 (orbit, **균일 촬영**) |
| 서버 | sysai3, RTX A6000 48GB |
| 포즈 | ✅ MASt3R-SfM 완료 → `real_test_combined_uniform__mast3r/poses.npy` (34,4,4) |
| 초기 점군 | ✅ `pointcloud.ply` 1,804,901 pts → 500k 다운샘플 → COLMAP points3D.txt |

> ⚠️ **데이터셋 교체 사유 (2026-06-28)**: 이전 `real_test_3m/7m`은 웨이포인트 기반 비행으로
> 각도 간격 std=9.4° (14°~43°), 타임스탬프 간격 3.8~7.9s로 **비균일 촬영**이었음.
> 신규 데이터셋은 각도 간격 std=1.5~2.5° (18°~31°)로 **거의 균일**하게 개선.
> 이전 실험 결과 전체 폐기, 신규 데이터로 처음부터 재시작.

### 포즈 품질 (per-group Umeyama Sim(3) 정렬, 2026-06-28 측정)

| 항목 | 3m (17장) | 7m (17장) | 전체 |
|---|---|---|---|
| ATE RMSE | **0.76 cm** | **27.6 cm** | 19.52 cm |
| RPE 회전 | 0.23° | 0.67° | — |
| focal (1920px 기준) | 969.40 px | 969.40 px | (shared_intrinsics) |

> ⚠️ 7m ATE 27.6cm는 swin-5 그래프 경계효과: 3m↔7m 간 cross-altitude 연결로
> 7m frame 0 (103cm 이상치) 발생. 7m 단독 실행 시 ATE=2.03cm였으나 합산 후 악화.
> 근본 원인: 고도 차이로 feature matching 불충분 → inter-orbit scale mismatch ~11.6%.
> **3m+7m 합산 복원 유지 이유**: 지형 커버리지 확보 목적.

## 1. 대원칙: GT는 평가에만, 학습엔 절대 안 씀

| 단계 | GT 사용? |
|---|---|
| 포즈 추정 / ply 초기화 / 모델 학습 / 메시 추출 | ❌ **GT 0% (100% MASt3R 추정값)** |
| 메시 정렬 + CD/F-score 평가 | ✅ GT 메시를 **정답지로만** (DTU/TNT 표준 관행과 동일) |
| 메시 정렬용 Sim(3) | ✅ GT 카메라 포즈를 **평가 시점 정렬에만** (학습 leakage 아님) |

> GT를 학습에 넣으면 leakage(부정행위)지만, 평가 정답지로 쓰는 것은 모든 surface
> reconstruction 논문(DTU Chamfer, TNT F-score)의 표준이다.

## 2. prior 정의 (혼동 방지 — 명확히 분리)

prior는 두 가지가 있으며 **서로 다른 것**이다:

| prior | 정체 | 역할 |
|---|---|---|
| **① ply 초기화** | MASt3R-SfM `pointcloud.ply` (1.8M점, 500k 다운샘플 후 COLMAP 투입) | 가우시안 시작 위치 (input). **모든 모델 공통** |
| **② depth supervision** | MASt3R 렌더 depth → invdepth L1 loss (`--depths`) | 학습 중 외부 깊이 정규화 |

- **"외부 depth"**(②)와 **"모델 내부가 자체 렌더링하는 depth"**(2DGS depth-distortion, TSDF fusion용 depth 등)는 다름. 후자는 알고리즘 본질이라 끌 수 없고 당연히 씀.
- 우리가 통제하는 변수는 ②(외부 MASt3R depth supervision)뿐.

## 3. 실험 트랙 (2개)

### 트랙 ① 메시 복원 (CD/F-score) — surface method 비교

| 설정 | 값 |
|---|---|
| 학습 | `eval=False`, **34장 전체** (held-out 없음 — 복원은 전량 투입이 정석) |
| 통일 prior | **① ply 초기화만** (외부 depth supervision ② 없음 → 공정) |
| 정규화 | 각 모델 **native 방식** (끌 수 없는 본연의 것) |
| 평가 | 추출 메시 vs **언리얼 export GT 메시**, CD↓ / F-score↑ |

| 모델 | 메시 추출 방식 | ply-init | 외부 depth |
|---|---|---|---|
| **3DGS + TSDF** | 별도 후처리 (open3d TSDF) | ✅ | ❌ |
| **2DGS** | native (surfel→TSDF) | ✅ | ❌ |
| **GS-2M** | native (TSDF) | ✅ | ❌ |
| **MILo** | native (learnable SDF→marching cubes) | ✅ | ❌ |

> 3DGS는 surface method가 아님(볼류메트릭 타원체) → TSDF 돌려도 noisy.
> baseline으로 포함해 "왜 2DGS/MILo가 필요한가"를 정량으로 보임. 표에 "별도 TSDF" 명시.

### 트랙 ② NVS (test PSNR) — depth supervision ablation

| 설정 | 값 |
|---|---|
| 학습 | `eval=True`, llffhold=8 → **29 train / 5 test** |
| test 5장 | `3m_000000, 3m_000008, 3m_000016, 7m_000007, 7m_000015` (자동 선택, llffhold=8) |
| 평가 | **held-out test 뷰** PSNR↑/SSIM↑/LPIPS↓ |

> 헤드라인 메시지: "② depth supervision이 **안 본 각도(test)** 품질을 얼마나 올리나".
> ⚠️ train-view PSNR은 암기 점수이므로 **성능으로 제시 금지**. 보여야 하면 "training-view
> reconstruction fidelity"로 라벨 명시.

**모델별 외부 depth supervision 지원 여부 (2026-06-29 코드 확인):**

| 모델 | 외부 depth 지원 | 방식 | 비고 |
|---|---|---|---|
| **3DGS** | ✅ | `--depths` 플래그, invdepth L1 loss (`depth_l1_weight` 지수감소) | 메인 ablation 대상 |
| **GS-2M** | ❌ | `depths` 인수 존재하나 train.py에 미사용. `depth_normal_loss`는 내부 일관성용 | 외부 depth 불가 |
| **2DGS** | ❌ | `depth_ratio`는 내부 depth-normal 일관성용. 외부 depth 파이프라인 없음 | 외부 depth 불가 |
| **MILo** | ✅ | `mast3r_depth_dir` config. depth ordering supervision (rank-based, metric 아님) | 3m_1에서 구현 완료 |

> **결론**: 외부 MASt3R depth supervision은 **3DGS와 MILo에만 적용 가능**.
> 2DGS/GS-2M은 각 모델 고유의 내부 supervision 방식을 사용하므로 depth ablation 대상 외.

**진행 상황 (2026-06-29 01:14 KST):**

| 모델 | prior | 7k test PSNR | 30k test PSNR | 상태 |
|---|---|---|---|---|
| **3DGS** (eval=True) | ① ply-init만 | **20.54** | 19.61 | ✅ 완료 |
| **3DGS + depth** (eval=True) | ① + ② MASt3R depth | 20.15 | 19.05 | ✅ 완료 |
| **MILo** (eval=True, 선택) | ① ply-init만 | — | — | ⏳ 낮은 우선순위 |

> **결과 해석**: depth supervision이 오히려 **−0.4dB 악화** (7k 기준).
> 두 모델 모두 7k에서 peak → 30k에서 과적합 (3DGS 과적합 gap ~1dB).
> 3m_1 결과(depth +0.09dB 미미)보다도 나쁜 결과 → MASt3R depth와 COLMAP이 동일 metric 스케일임에도
> 이 씬에서 depth prior는 도움이 되지 않음.
> **근본 원인 추정**: 34장 중 29장으로만 학습(eval=True) 시 지형 커버리지 부족 + 3m/7m 고도 차이
> scale mismatch가 depth loss 방향을 혼란시킴.

> ⚠️ 기존 `real_test_combined_uniform__3dgs`는 **eval=False**로 학습됨 (train PSNR=30.80dB).
> Track ②용: `real_test_combined_uniform__3dgs_eval` (eval=True 재학습).

**depth map 생성 완료 (2026-06-29):**
- `/tmp/gen_depth_combined.py`: MASt3R pointcloud → 카메라별 투영 → 34장 depth.npy
  - `R_w2c = R_c2w.T`, `t_w2c = -R_c2w.T @ t_c2w`, focal=969.40px (1920px 기준)
  - 유효 픽셀: 690k~890k/frame, 깊이 범위 [0.92, 13.2]m ✅
- `/tmp/convert_depth_combined.py`: .npy → uint16 invdepth PNG + depth_params.json
  - COLMAP/MASt3R depth 스케일 비율=**0.9899** (동일 metric ✅)
  - 출력: `real_test_combined_uniform__colmap/depths_png/` (34 PNG)
  - `sparse/0/depth_params.json` (키: `3m_000000` 형식, 확장자 없음)
- 3DGS `--depths depths_png` → `source_path/depths_png/` 자동 로드 ✅

**test 프레임** (llffhold=8, 34장 정렬 기준):
`3m_000000, 3m_000008, 3m_000016, 7m_000007, 7m_000015` (5장)

## 4. 메시 평가 절차 (트랙 ①, 언리얼 GT 받은 후)

```
0단계  handedness 일치: 언리얼(left-handed) → 한 축 flip(예: Y→−Y) → right-handed
        ※ Sim(3)는 det(R)=+1만 허용 → 거울상은 ICP로도 안 겹침. flip이 최우선.
1단계  Sim(3) 정렬: GT 카메라 1:1 대응으로 결정(s≈3.49) + ICP 미세정렬
2단계  cropping 통일: 복원 메시를 GT bbox(+margin)로 crop, 4개 모델 동일 적용
        ※ CD 양방향 → 복원 메시의 GT-외 floater가 accuracy 부풀림 방지 (TNT foreground 논리)
3단계  CD(양방향) + F-score @1cm/5cm/10cm 산출
```

### 4-1. GT 메시 (언리얼 export 완료, 2026-06-28 수령)

| 항목 | 값 |
|---|---|
| **원본 파일** | `gt_scene_full.fbx` (4.0MB, Kaydara FBX Binary, 언리얼 export) |
| 로컬 | `C:\Users\sdh97\Desktop\학교\캔위성\2차 발표자료\현재결과\gt_mesh\gt_scene_full.fbx` |
| 서버 | `sdh@sysai3:/home/sdh/Desktop/data/gt_mesh/gt_scene_full.fbx` |
| 출처 다운로드 | `C:\Users\sdh97\Downloads\my_box.fbx` (중복본 `my_box (1).fbx`는 동일 md5 → 삭제) |

**FBX 내 메시 노드 (Geometry 10개):**
- `Cube8`~`Cube12` — GT 큐브 5개 (기존 `Cube{8..12}.ply`와 동일)
- `Cube` — 추가 큐브 1개 (기존 PLY엔 없던 것)
- `Landscape` — **지면/지형 메시** (= 주변 환경)
- `MI_LayerGround` — 지면 머티리얼
- `UCX_*` — 언리얼 충돌 메시 (평가 제외 대상)

> ⭐ **의의**: 이 FBX엔 큐브뿐 아니라 **`Landscape`(지면)**가 포함됨 → 기존 "지면 오염 편향"
> (GT에 큐브만 있어 crop 안 지면 복원점이 Precision/CD를 왜곡) **해결 가능**.
> 이제 "물체만(큐브)" 평가와 "환경 포함(큐브+지면)" 평가를 **둘 다** 산출 가능.

### 4-2. FBX → PLY 변환 완료 (2026-06-29, 검증 통과)

**도구**: 서버에 blender/assimp/pymeshlab 전무 → 격리 venv(`/tmp/venv-fbx`)에 `pyassimp` 설치.
pymeshlab은 11개 메시를 1개로 병합해버려 부적합 → pyassimp로 노드별 접근.

**핵심 발견 — Y축 handedness 반전**:
- FBX 노드 월드변환 적용 후 큐브 좌표가 PLY와 **X·Z는 일치, Y만 부호 반전**
  - 예: FBX `Cube8` Y[5063,5103] vs PLY Y[−5103,−5063]
- 원인: 언리얼(left-handed) FBX export → assimp 임포트 시 Y축 부호. STATUS 0단계 handedness 그 자체
- **조치**: `Y → −Y` flip 적용 → 기존 검증된 `Cube{8..12}.ply`(카메라 ATE 0.6cm 정렬)와
  **maxΔ=0.000cm 완전 일치 확인** → 변환 신뢰성 검증됨

**메시 구조 (pyassimp, 노드 월드변환 + Y-flip 적용)**:
- `Cube8`~`Cube12`: 평가용 큐브 5개 (각 144V/48F)
- `UCX_Cube*`: 충돌 메시 → **제외**
- `Landscape`: **z=0 완전 평탄 지면**, ±252m (1.52M V/508k F)
- `Cube` 노드: 메시 없는 그룹 노드 (무시)

**산출 PLY** (서버 `/home/sdh/Desktop/data/gt_mesh/`, 로컬 `gt_mesh/`, Unreal cm 프레임):
| 파일 | 내용 | 크기 | 용도 |
|---|---|---|---|
| `gt_cubes.ply` | 큐브 5개 | 720V/240F (12K) | **물체-only 평가** |
| `gt_landscape.ply` | 지면 z=0 | 1.52M V (24M) | 지면 단독 |
| `gt_scene_clean.ply` | 큐브+지면 (UCX 제외) | 1.52M V (24M) | **환경 포함 평가** (지면 오염 해소) |

**변환 스크립트**: `/tmp/fbx_to_ply.py` (월드변환+Y-flip+UCX제외+큐브 검증 내장)

> ⚠️ **환경 평가 시 주의**: `gt_landscape`는 ±252m 무한지면 → Recall 계산 시 GT를
> **카메라 가시영역으로 crop** 필요 (안 그러면 미복원 원거리 지면이 Recall≈0 만듦).
> 물체-only 평가는 큐브 bbox crop으로 충분.

### 4-3. 메시 평가 결과 (2026-06-29, NED 공간 직접 CD)

**평가 방법** (`/tmp/eval_env.py`):
- 정렬: **3m Umeyama** (COLMAP→NED), s=3.0798, ATE=0.60cm (큐브가 3m 카메라 바로 아래)
- NED(m) 공간 직접 CD/F-score, **밀도 일관 샘플링** (crop 영역 face만 80k 동일)
- 환경 평가: 카메라 지면 footprint(2.1×2.1m)+1m crop, z∈[−1.2,0.3]
- 물체 평가: 큐브 bbox+5cm crop

**환경 포함 (gt_scene_clean = 큐브+지면)** — 지면 오염 해소된 정식 평가:

| 순위 | 모델 | CD↓ | F@1cm | F@5cm | F@10cm |
|---|---|---|---|---|---|
| 🥇 | **2DGS** | **6.85cm** | 0.028 | **0.156** | **0.914** |
| 🥈 | GS-2M | 7.47cm | 0.020 | 0.123 | 0.897 |
| 🥉 | MILo | 11.17cm | 0.008 | 0.080 | 0.366 |
| 4 | 3DGS+TSDF | 89.80cm | 0.000 | 0.000 | 0.000 |

**물체-only (gt_cubes = 큐브 5개)** — 참고:

| 순위 | 모델 | CD↓ | F@5cm | F@10cm |
|---|---|---|---|---|
| 🥇 | **2DGS** | **4.93cm** | **0.635** | **0.800** |
| 🥈 | GS-2M | 5.15cm | 0.622 | 0.789 |
| 🥉 | MILo | 7.32cm | 0.448 | 0.659 |
| 4 | 3DGS+TSDF | 큐브 bbox 내 surface 0개 (복원 실패) | — | — |

> **해석** (최종, 4모델):
> - **순위: 2DGS > GS-2M > MILo ≫ 3DGS** (물체·환경 일관)
> - **2DGS ≈ GS-2M**: F@10 0.914 vs 0.897, 거친 형상 모두 양호. 1~5cm 정밀도에서 2DGS 근소 우위.
> - **MILo 환경 F@10=0.366** (2DGS의 절반↓): MILo 메시의 **가장자리 스파이크 노이즈**가 지면
>   평탄도를 깸 (이전 real_test에서도 관찰된 MILo 고질). 물체-only(F@10=0.659)는 상대적으로 양호.
> - **3DGS는 surface method 아님** → TSDF 메시가 수직 2.7m 두께 노이즈, 지면이 GT보다
>   0.16~2.86m 위에 분포 → 환경 CD 89.8cm, 큐브 bbox 내 surface 전무. baseline으로서
>   "왜 2DGS/GS-2M/MILo가 필요한가"를 정량 입증 (3DGS 원본 가우시안 top-view 시각자료도 확보).

> ⚠️ **검증 이력 (편향 제거)**: 초기 평가는 ① 전체메시 균일샘플 → crop 영역 밀도 불일치,
> ② margin 과대(11cm) → 지면 오염으로 CD 부풀림. 밀도 일관 샘플 + GT에 지면 포함으로 수정.
> Recall@5cm은 margin 무관하게 robust (2DGS 1.0 vs 3DGS 0.096 — 초기부터 일관).

### 4-4. MILo SOR 출력메시 실험 (2026-06-29, 재학습 없이 출력 메시에 SOR 적용)

**방법**: `mesh_learnable_sdf.ply` (5,029,391V) → open3d SOR → 3강도 메시 저장 → 재평가  
**환경**: `recon3d` env (open3d 0.19.0, libstdc++ 호환)

| 강도 | nb | std | 보존V(%) | 출력 파일 |
|---|---|---|---|---|
| weak   | 30 | 2.0 | 96.7% (4,862,809V) | `mesh_sor_weak.ply` |
| mid    | 30 | 1.0 | 91.9% (4,624,059V) | `mesh_sor_mid.ply` |
| strong | 30 | 0.5 | 86.6% (4,354,600V) | `mesh_sor_strong.ply` |

**물체-only (gt_cubes) 평가:**

| 모델 | CD↓ | F@1cm | F@5cm | F@10cm |
|---|---|---|---|---|
| MILo 원본 | 7.32cm | 0.081 | 0.448 | 0.659 |
| SOR_weak   | 7.24cm | 0.082 | **0.449** | **0.661** |
| SOR_mid    | **7.17cm** | 0.070 | 0.437 | 0.659 |
| SOR_strong | 7.57cm | 0.039 | 0.368 | 0.631 |

**환경포함 (gt_scene_clean) 평가:**

| 모델 | CD↓ | F@1cm | F@5cm | F@10cm |
|---|---|---|---|---|
| MILo 원본 | 11.17cm | 0.008 | 0.080 | 0.366 |
| SOR_weak   | 10.99cm | 0.008 | 0.082 | 0.375 |
| SOR_mid    | **10.81cm** | 0.008 | 0.078 | **0.390** |
| SOR_strong | **10.81cm** | 0.005 | 0.064 | 0.406 |

> **결론**: 출력 메시 SOR은 효과가 미미하다.
> - **SOR_mid**: 환경 F@10 0.366 → 0.390 (+0.024, +6.6%) 개선이 가장 좋은 tradeoff.
> - **SOR_strong**: 환경 F@10 0.406이지만 물체 F@10 0.631로 크게 하락 (큐브 표면 정점 과도 제거).
> - **전체 결론**: 출력 메시 SOR은 이전 COLMAP prior SOR 실험과 동일하게 효과 미미.
>   MILo 한계(환경 F@10≈0.4)는 edge spike 노이즈 구조 문제 → 알고리즘 수준 해결 필요.
> - **최종 순위(환경 F@10) 변경 없음**: 2DGS(0.914) > GS-2M(0.897) ≫ MILo_SOR_mid(0.390) ≫ 3DGS(0.000)

## 5. 시뮬레이션 실험 최종 결론 (2026-06-29 확정)

### 핵심 결론

| 항목 | 결론 |
|---|---|
| **최고 모델 (메시)** | **2DGS** — CD 6.85cm, F@10 0.914, 모든 지표 1위 |
| **2위** | GS-2M — F@10 0.897, 근접하지만 전 지표 2위 |
| **NVS depth prior** | 효과 없음 (−0.4dB 악화). depth가 아닌 데이터 품질이 병목 |
| **GS-2M < 2DGS 이유** | BRDF 추정이 photometric gradient 부재 씬에서 ill-posed → 기하 왜곡 |
| **다른 모델도 못 이김** | 이 씬의 한계(텍스처 빈약+sparse view)는 알고리즘이 아닌 데이터 문제. 복잡한 방법일수록 역효과 |

### 조건부 명제 (일반화 주의)

> **"텍스처가 빈약한 미터 스케일 sparse-view 야외 씬에서는 BRDF 기반 방법이 단순 surfel 기반보다 불리하다"**
>
> GS-2M을 이기려면 더 좋은 알고리즘이 아니라 **더 좋은 데이터(텍스처, dense view)**가 필요.
> 실제 드론 데이터(자연 텍스처 존재)에서는 결과가 달라질 수 있음.

### F-score 스케일 해석 기준

- F@1cm 낮음(0.028): 결함 아님. 미터 스케일 씬에서 1cm는 의도적으로 tight한 임계값
- F@10cm 기준(0.914): 이 씬의 실용적 정확도 지표
- CD 6.85cm: NED meter 기준 ~2m orbit 씬에서 합리적 수치

---

## 6. 의존성 / 시작 순서

| 트랙 | 필요 입력 | 상태 |
|---|---|---|
| **MASt3R-SfM** (포즈 + ply) | `real_test_3m_uniform` + `real_test_7m_uniform` | ✅ **완료** |
| **COLMAP 변환** (`build_colmap_uniform.py`) | poses.npy + pointcloud.ply | ✅ **완료** → `real_test_combined_uniform__colmap/` |
| **3DGS** (30k, gs3d env) | COLMAP | ✅ 학습+TSDF메시 완료 |
| **2DGS** (30k, venv-2dgs) | COLMAP | ✅ 학습+메시 완료 |
| **GS-2M** (30k, miniforge3/gs2m) | COLMAP | ✅ 학습+메시 완료 |
| **MILo** (18k, venv-milo, outdoor+radegs) | COLMAP | ✅ 학습+메시 완료 |
| 메시 추출 (4모델) | 학습 완료 체크포인트 | ✅ **완료** |
| 메시 CD/F-score 평가 | GT + 4모델 메시 | ✅ **완료** (4-3 결과표) |
| NVS depth ablation (3DGS eval=True) | MASt3R depth map 34장 | ✅ 완료 (peak PSNR 20.54 @7k) |
| NVS depth ablation (3DGS+depth eval=True) | MASt3R depth map 34장 | ✅ 완료 (peak PSNR 20.15 @7k) |

### 메시 추출 명령 (학습 완료 후)
| 모델 | 명령 |
|---|---|
| 2DGS | `render.py --voxel_size 0.01 --depth_trunc 6.0 --sdf_trunc 0.04 --num_cluster 1` |
| GS-2M | `render.py --extract_mesh --skip_test` |
| MILo | `mesh_extract_sdf.py` |
| 3DGS | 별도 open3d TSDF 후처리 |

---

## 문제 이력 — real_test_combined_uniform 복원 중 발생 (2026-06-28)

### 1. 데이터셋 비균일성 (근본 교체 사유)
- **증상**: HTML 시각화에서 프레임 간격이 불규칙해 보임
- **원인**: 이전 `real_test_3m/7m`는 웨이포인트 기반 비행 → 각도 간격 std=9.4° (14°~43°)
- **조치**: 새 균일 데이터셋(`real_test_3m_uniform/7m_uniform`, std=1.5~2.5°) Google Drive에서 재다운로드, 기존 실험 전량 삭제

### 2. venv-mast3r 경로 오류
- **증상**: `source /home/sdh/venv-mast3r/bin/activate` → `No such file`
- **원인**: 실제 경로는 `/home/sdh/Desktop/venvs/venv-mast3r/`
- **조치**: 경로 수정

### 3. MASt3R-SfM `--weights` 인자 누락
- **증상**: `error: the following arguments are required: --weights`
- **조치**: `--weights /home/sdh/Desktop/models/MAST3R_2/checkpoints/MASt3R_ViTLarge_BaseDecoder_512_catmlpdpt_metric.pth` 추가

### 4. 합산 복원 시 7m ATE 악화 (inter-orbit scale mismatch)
- **증상**: 3m+7m 합산 MASt3R-SfM에서 7m ATE=27.6cm (단독 2.03cm 대비 13.6× 악화)
- **원인**: swin-5 그래프 경계(frame 12-16 ↔ frame 17)에서 cross-altitude feature matching 불충분 → 7m frame 0 이상치(103cm). 순서 바꿔도(7m 먼저) 동일하게 88cm 이상치 → 구조적 한계
- **조치**: 합산 유지 (지형 커버리지 우선). per-group Umeyama로 평가.

### 5. HTML 좌표계 틸트
- **증상**: 시각화에서 궤적이 ~18° 기울어짐
- **원인**: MASt3R world frame ≠ NED/ENU 정렬
- **조치**: per-group Umeyama로 GT NED 정렬 후 NED→ENU 변환 (`ned2enu = [y, x, -z]`)

### 6. conda 환경 이중 설치 혼란 (sysai3)
- **증상**: `conda activate gs2m` → `EnvironmentNameNotFound`
- **원인**: sysai3에 conda 두 곳: `~/miniconda3/` (MonoGS만 있음) vs `~/miniforge3/` (gs2m, gs3d, mast3r310, milo 등 전부)
- **조치**: GPU ML 환경은 모두 `source /home/sdh/miniforge3/bin/activate <env>` 사용

### 7. 3DGS `diff_gaussian_rasterization` 패키지 미발견
- **증상**: `gs_env`에서 import error
- **원인**: `gs_env`에 바닐라 3DGS 설치 안 됨
- **조치**: `miniforge3/envs/gs3d` 사용

### 8. MILo `FileNotFoundError: ./configs/fast`
- **증상**: `python3 /home/sdh/Desktop/models/milo/train.py` 실행 시 config 파일 못 찾음
- **원인**: config 경로가 상대경로(`./configs/`) 기준 → working directory가 `milo/milo/`이어야 함
- **조치**: `cd /home/sdh/Desktop/models/milo/milo/ && python3 train.py ...`

### 9. combined 디렉토리에 68장 (old symlink 잔류)
- **증상**: 34장이어야 할 `real_test_combined_uniform_rev/rgb/`에 68장 존재
- **원인**: 이전 데이터 삭제 후 디렉토리 재생성 시 구 symlink 미정리
- **조치**: `rm -rf real_test_combined_uniform_rev/` 후 재생성

---

## 문제 이력 — real_test (구 데이터) 복원 중 발생

### 10. 점군 outlier 문제 → SOR 실험 → 효과 없음 결론

**배경**: MILo/AGS 등 직접-위상 방법에서 메시 가장자리 스파이크/잡음 심각.
점군이 scene_extent의 4.9×까지 퍼진 것 확인 (COLMAP points3D full diag 15.84, scene_extent 1.99).

**가설 검증:**
- H1 스케일/좌표계 이상 → ❌ 기각 (scene_extent 정상 범위)
- H2 low-confidence outlier → ✅ 핵심 원인으로 판단 → SOR 실험 진행

**SOR 실험 경과:**

| 단계 | 설정 | 보존율 | 결과 |
|---|---|---|---|
| SOR1 | nb=20, std=2.0 | 94.6% | 가장자리 스파이크 여전히 존재 |
| SOR2 | nb=30, std=1.0 | 88.0% | MILo 재학습 → 수치 소폭 개선 |
| SOR3 | nb=30, std=0.5 | 81.7% | `real_test_milo_sor2.ply` 완료 |

**메시 품질 비교 (5개 큐브 GT 기준):**

| 모델 | CD (cm) | F@10cm |
|---|---|---|
| MILo baseline | 37.20 | 0.018 |
| MILo SOR2 (nb=30, std=0.5) | 30.72 | 0.024 |
| 차이 | △-6.5cm | △+0.006 |

**결론**: SOR로 floater 제거해도 CD 개선 미미, F-score 거의 변화 없음.
**근본 원인은 outlier가 아니라 texture 부족** (단색 배경 + 큐브 → photometric gradient 없음).
→ 알고리즘 튜닝으로는 해결 불가, 데이터 자체의 한계.

### 11. GOF (Gaussian Opacity Fields) 메시 추출 hanging
- **증상**: 학습 30k 완료 후 메시 추출 시 binary search step 7에서 멈춤, 무한대기
- **원인**: unbounded 야외 씬에서 GOF binary search가 occupancy threshold 수렴 실패로 추정
- **조치**: 타임아웃 후 포기, GOF 메시 결과 없음

### 12. depth prior가 오히려 악화 (blue_1)
- **증상**: `MILo+prior` CD=11.35cm > `MILo baseline` CD=9.40cm
- **원인**: 단색 파란 박스에서 MASt3R depth map 자체가 노이즈 심함 → 잘못된 depth 정보가 학습 왜곡
- **결론**: texture gradient 없는 씬에서 외부 depth supervision은 역효과. real_test에서도 prior 효과 미미 (21.44cm vs 21.54cm).

### 13. 3m_1 focal 버그 (첫 번째 시도 전량 폐기)
- **증상**: 학습 PSNR 16dB (정상 22dB+), 카메라가 어안렌즈처럼 동작
- **원인**: MASt3R는 512px 기준으로 focal 저장 → COLMAP 변환 시 원본 해상도 스케일백 누락
  - 잘못된 값: 359.88px (FOV 139°) → 올바른 값: `359.88 × (1920/512)` = **1349.55px** (FOV 71°)
- **조치**: `build_colmap.py`에 `focal = focal_512 * (W / 512)` 수정, 실험 전량 재실행
  - uniform 데이터에서는 focal=969.40px (`258.51 × 3.75`)

### 14. AGS input `colmap/` 심링크 누락
- **증상**: `"Could not recognize scene type!"` 오류
- **원인**: AGS는 입력 디렉토리 안에 `colmap/` 심링크 필수
- **조치**: `ln -s /path/to/colmap ./input_dir/colmap`

---

## 현재 상태: 3m_1 전 모델 NVS + 메시 추출 완료 (2026-06-27)

### 3m_1 NVS 결과 (test 7장, llffhold=8 → 48 train / 7 test, 2026-06-27 완료)

**30k 기준 PSNR/SSIM/LPIPS 전체 비교**

| 순위 | 모델 | PSNR↑ | SSIM↑ | LPIPS↓ | iter |
|---|---|---|---|---|---|
| PSNR 🥇 | **MILo** | **22.106** | 0.5853 | 0.3543 | 18k |
| SSIM 🥇 | **3DGS+depth** | 21.900 | **0.5949** | **0.2641** | 30k |
| LPIPS 🥇 | **3DGS+depth** | 21.900 | 0.5949 | **0.2641** | 30k |
| — | GS-2M | 21.915 | 0.5889 | 0.2823 | 30k |
| — | 2DGS | 21.906 | 0.5903 | 0.2741 | 30k |
| — | 3DGS baseline | 21.811 | 0.5878 | 0.2642 | 30k |

**best iter 기준 (peak PSNR)**

| 모델 | best test PSNR | iter | 비고 |
|---|---|---|---|
| **3DGS+depth** | **22.23** | 7k | depth prior 효과 (7k peak, 이후 하락) |
| **MILo** | **22.11** | 18k | 과적합 gap 최소 (1.7dB) |
| GS-2M | 21.93 | 25k | 30k 소폭 하락 |
| 3DGS | 22.07 | 7k | 30k 과적합 (gap 6.1dB) |
| 2DGS | 21.91 | 30k | 수렴 안정 |

**지표별 해석:**
- **PSNR**: MILo 1위 — 픽셀 정확도 가장 높음
- **SSIM/LPIPS**: 3DGS+depth 동시 1위 — 지각적 품질(텍스처·구조)은 depth prior Gaussian이 우수
- 모든 모델 PSNR ~21.8~22.1dB 수렴 → 알고리즘 아닌 **데이터 천장**이 지배
- depth prior: PSNR +0.09dB 미미, LPIPS −0.0001 미미하지만 **SSIM +0.007** 유의미

### 3m_1 메시 추출 현황 (2026-06-27 완료)

| 모델 | 메시 파일 | 크기 | 버텍스 수 | 방식 |
|---|---|---|---|---|
| ✅ MILo | `3m_1__mast3r__milo/mesh_learnable_sdf.ply` | 80MB | 2,003,258 | Learnable SDF |
| ✅ GS-2M | `3m_1__mast3r__gs2m/train/ours_30000/mesh/tsdf_post.ply` | 27MB | 523,613 | TSDF fusion |
| ✅ 2DGS | `3m_1__mast3r__2dgs/train/ours_30000/fuse_post.ply` | 33MB | 671,316 | TSDF fusion |
| — | 3DGS / 3DGS+depth | — | — | point cloud만 (별도 TSDF 필요) |

TSDF 파라미터 (GS-2M / 2DGS 공통): `voxel_size=0.01, depth_trunc=6.0, sdf_trunc=0.04, num_cluster=1`

### 핵심 발견
1. **모든 모델 ~21.8–22.2dB에 수렴** → 알고리즘 차이보다 **데이터 천장**이 지배
2. 천장 원인 = **포즈 정확도(MASt3R 누적오차) + 잔디/야외 장면** (geometry가 아님)
3. depth prior는 geometry만 제약 → +0.17dB로 천장 못 뚫음
4. 3DGS는 자유도 높아 sparse view에서 과적합 (train-test gap 6.1dB)
5. 천장 돌파 레버: **COLMAP BA 포즈 재정렬** > 프레임 조밀화(10→5)

### MASt3R-SfM → 3DGS depth prior 파이프라인 (공식 경로, 신규 구축)
- depth 추출: `gen_3m1_depth_maps.py` (버그 2개 수정: reshape, world→camera Z 변환). 출력 `3m_1__mast3r_depth/depth_maps/*.npy` (55장, 288×512, metric 1.4~4m)
- 역깊이 PNG 변환: `/tmp/convert_depth_3dgs.py` → `depths_png/` + `sparse/0/depth_params.json`
- **스케일 검증**: COLMAP/MASt3R 깊이 median 비율 **1.0019** (동일 metric 스케일 → scale=0.998 offset=0 직접 사용, make_depth_scale 불필요)
- ⚠️ MASt3R COLMAP export에 2D-3D 대응점 없음 → make_depth_scale.py 사용 불가 (metric 일치로 우회)
- 학습 env: `gs3d` (gs2m clone + 바닐라 diff-gaussian-rasterization/simple-knn/fused-ssim 설치). gs2m rasterizer는 GS-2M 커스텀(feature_count 필드)이라 바닐라 3DGS 비호환
- 출력: `3m_1__mast3r__3dgs{,_depth}/point_cloud/iteration_{7000,30000}/`

### 서버 경로 요약 (sysai3)
```
/home/sdh/Desktop/data/experiments/
├── 3m_1__mast3r__milo/        mesh_learnable_sdf.ply (80MB)
├── 3m_1__mast3r__gs2m/        train/ours_30000/mesh/tsdf_post.ply (27MB)
├── 3m_1__mast3r__2dgs/        train/ours_30000/fuse_post.ply (33MB)
├── 3m_1__mast3r__3dgs/        point_cloud/iteration_30000/point_cloud.ply (702MB)
├── 3m_1__mast3r__3dgs_depth/  point_cloud/iteration_30000/point_cloud.ply (735MB)
└── 3m_1__mast3r__colmap/      02_colmap/ (COLMAP 입력)
```

### 미완 / 다음
- 메시 3개(MILo/GS-2M/2DGS) 로컬 다운로드 미완료
- 5m_1, 7m_1 동일 파이프라인 미실행
- COLMAP BA 포즈 재정렬 미시도

---

## real_test 결과 요약 (시뮬레이션, 34장, AirSim)

> **데이터**: AirSim 시뮬레이션 렌더링 (실제 드론 아님), 34장, 1920×1080, GT 메시 있음  
> **3m_1과 다른 데이터셋** — 비교 혼용 금지

### NVS PSNR (real_test) — 재학습 예정

> ❌ **문제**: 2DGS/MILo 모두 `eval=False`로 학습됨 → 34장 전부 train에 사용  
> AGS만 `eval=True` (4장 held-out). 직접 수치 비교 불가 → **2DGS·MILo 재학습 필요**

**현재 측정값 (참고용, 공정 비교 아님)**

| 모델 | PSNR | 뷰 | 비고 |
|---|---|---|---|
| 2DGS (30k) | 32.31 | train-34 | ❌ eval=False, 재학습 필요 |
| MILo baseline (18k) | 28.07 | train-34 | ❌ eval=False, 재학습 필요 |
| MILo+prior (18k) | 28.13 | train-34 | ❌ eval=False, 재학습 필요 |
| MILo SOR2 (18k) | 27.53 | train-34 | ❌ eval=False, 재학습 필요 |
| GS-2M (30k) | 31.21 | train (추정) | 모델 없음 |
| AGS baseline (7k) | 17.49 | **test-4** | ✅ eval=True |
| AGS+prior (30k) | 17.28 | **test-4** | ✅ eval=True, 30k 과적합 |

**재학습 계획 (eval=True, llffhold=8 → 30 train / 4 test)**

| 모델 | 우선순위 | 예상 시간 | 상태 |
|---|---|---|---|
| 2DGS (30k) | 🔴 높음 | ~1시간 | ⏳ 대기 |
| MILo baseline (18k) | 🔴 높음 | ~2시간 | ⏳ 대기 |
| MILo+prior (18k) | 🟡 낮음 (ablation) | ~2시간 | ⏳ 대기 |
| MILo SOR2 (18k) | 🟡 낮음 (ablation) | ~2시간 | ⏳ 대기 |
| AGS baseline (7k) | — | — | ✅ 완료 |

### 메시 품질 (real_test, GT 큐브 5개 기준, Umeyama 정렬 후 CD/F-score)

| 순위 | 모델 | CD (cm) | F@1cm | F@5cm | F@10cm |
|---|---|---|---|---|---|
| 🥇 | **AGS baseline** | **25.41** | 0.019 | 0.057 | **0.127** |
| 🥈 | GS-2M | 27.79 | 0.004 | 0.029 | 0.057 |
| 🥉 | MILo SOR2 | 30.72 | 0.003 | 0.013 | 0.024 |
| 4 | MILo baseline | 37.20 | 0.003 | 0.010 | 0.018 |
| 5 | 2DGS | 37.25 | 0.003 | 0.019 | 0.036 |

> ⚠️ CD가 높은 건 전체 씬 메시를 GT 소형 큐브(5개)랑 비교하기 때문 — 절대값보다 상대 순위가 의미 있음  
> 모든 방법 실패에 가까움 → 근본 원인: **texture 부족** (단색 배경 + 큐브)

---

## (이전) 3m_1 평가 파이프라인 설계 완료 → 재실행 대기 (2026-06-25)

- **완료**: `real_test__milo_sor2` — 18000 iter, `mesh_learnable_sdf.ply` (215MB)
- **완료**: `blue_1__milo_fhd_prior` — 20MB PLY (depth prior 적용)
- **완료**: 메시 정량 평가 (Chamfer Distance + F-score) — 모든 데이터셋에서 texture 부족이 병목
- **보류**: `real_test__gof` — 메시 추출 hanging (binary search step 7 후 멈춤)
- **완료 (2026-06-24)**: GS-2M (Eurographics 2026) 학습 + TSDF 메시 추출 — `/mnt/c/Users/sdh97/Desktop/3d_results/real_test/gs2m_output/`
- **완료 (2026-06-24)**: 실제 드론 3개 고도 폴더 MASt3R-SfM — `sysai3:~/Desktop/data/drone_real_sfm/{3m_1,5m_1,7m_1}/`
- **완료 (2026-06-24)**: GS-2M 메시 품질 정량 평가 (재평가, Umeyama 기반 COLMAP 공간 비교)
- **❌ 취소**: 3m_1 첫 번째 학습 시도 (GS-2M/2DGS/MILo) — focal 버그로 전량 폐기
- **⏳ 대기**: focal 수정 후 3m_1 재실행 (파이프라인 설계 완료)

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

### GOF (Gaussian Opacity Fields) — 학습 완료, 메시 추출 실패

- **목적**: unbounded 야외 씬 특화 최신 모델 (NeurIPS 2024) 적용
- **학습**: ✅ 30000 iter 완료
- **메시 추출**: ❌ binary search step 0~7 완료 후 hanging (최종 메시 생성 단계서 멈춤)
- **입력**: SOR2 필터링된 COLMAP (`real_test__milo_sor2/colmap/`, 1,218,707 pts)

---

## 메시 품질 정량 평가 (2026-06-24)

### 평가 방법

- **GT**: UE5 scene cube extract (정확한 좌표 + 메시)
- **지표**: Chamfer Distance (CD), F-score @1cm/@5cm/@10cm
- **정렬**: 스케일 정규화 (bbox diagonal 기준)

### blue_1 결과 (1개 큐브)

| 방법 | CD (cm) | F@1cm | F@5cm | F@10cm | 평가 |
|---|---|---|---|---|---|
| MILo baseline | **9.40** | 0.022 | 0.178 | **0.556** | ✅ 최고 |
| MILo+prior | 11.35 | 0.014 | 0.135 | 0.365 | ❌ depth prior 악화 |
| AGS-Mesh | 9.67 | 0.023 | **0.221** | 0.523 | 유사 |
| MASt3R TSDF | 10.08 | 0.016 | 0.153 | 0.470 | — |
| 2DGS | 10.63 | 0.008 | 0.114 | 0.446 | — |

**발견**: depth prior가 오히려 해악 (단색 박스에서 depth map 노이즈가 학습 왜곡)

### real_test 결과 (5개 큐브, 구 평가 방법)

> ⚠️ 아래 수치는 원본 eval 스크립트 결과 (스크립트 분실로 재현 불가). GS-2M 비교는 상단 "GS-2M 학습+평가" 섹션 참조.

| 방법 | CD (cm) | F@1cm | F@5cm | F@10cm | 평가 |
|---|---|---|---|---|---|
| MILo baseline | **21.54** | 0.000 | 0.025 | 0.088 | — |
| **MILo SOR2** | 21.44 | 0.000 | 0.010 | **0.101** | ⚠️ SOR 무의미 |
| MILo+prior | 21.42 | 0.000 | 0.014 | 0.081 | ⚠️ prior 무의미 |
| AGS baseline | 23.52 | 0.000 | 0.013 | 0.052 | — |
| AGS+prior | 24.31 | 0.000 | 0.009 | 0.102 | — |
| 2DGS | 21.52 | 0.001 | 0.012 | 0.100 | — |

**발견**: SOR, depth prior 모두 효과 없음 → 모든 방법 ~21-24cm (texture 부족이 병목)

### 결론

**근본 원인: Texture/photometric gradient 부족**
- blue_1: 단색 파란 박스 → 색상 정보 전무
- real_test: 주로 단색 배경 + 5개 큐브 → signal 극도로 부족

**시사점**:
1. Outlier 제거(SOR) → 실제 메시 품질 향상 무
2. Depth prior → blue_1에서 악화, real_test에서 무의미
3. 모든 방법(2DGS/MILo/AGS) 비슷하게 실패 → **알고리즘이 아니라 데이터 문제**

---

## 포즈 정확도 비교 — GT vs MASt3R-SfM (2026-06-23)

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

---

## 실제 드론 촬영 데이터 (2026-06-24 추가)

- **출처**: 실제 드론으로 촬영한 영상 프레임 (시뮬레이션 아님)
- **Google Drive**: https://drive.google.com/file/d/1y_HwAsE0eA3lCimyzNVqh3dLiGPdrs-e/view?usp=sharing
- **파일**: ZIP (197MB), 총 1,656장 JPG
- **로컬**: `C:\Users\sdh97\Desktop\drone_images\{3m_1,5m_1,7m_1}\`

| 폴더 | 장수 | 설명 |
|---|---|---|
| `3m_1` | 547장 | 고도 3m 촬영 |
| `5m_1` | 544장 | 고도 5m 촬영 |
| `7m_1` | 565장 | 고도 7m 촬영 |

- **비고**: 기존 STATUS.md의 `real_test` 항목에 "시뮬레이션"으로 잘못 기재된 부분 있음. real_test 34장도 실제 드론 촬영 데이터임.

### 드론 영상 MASt3R-SfM 결과 (2026-06-24 완료)

- **서버**: sysai3, 환경 `venv-mast3r`, 스크립트 `~/Desktop/run_drone_mast3r.sh`
- **설정**: scene_graph=swin winsize=5, 10프레임마다 1장 샘플링 (~55장/폴더), shared_intrinsics

| 폴더 | 입력 장수 | 출력 | 경로 |
|---|---|---|---|
| `3m_1` | 55장 | poses.npy (55,4,4), focals.npy, pointcloud.ply | `~/Desktop/data/drone_real_sfm/3m_1/` |
| `5m_1` | 55장 | poses.npy (55,4,4), focals.npy, pointcloud.ply | `~/Desktop/data/drone_real_sfm/5m_1/` |
| `7m_1` | 57장 | poses.npy (57,4,4), focals.npy, pointcloud.ply | `~/Desktop/data/drone_real_sfm/7m_1/` |

---

## GS-2M (Eurographics 2026) 학습 + 평가 (2026-06-24)

### 학습 정보

- **입력**: `real_test` 데이터 (34장, 1920×1080), COLMAP 초기화
- **모델**: GS-2M — material-aware Gaussian Splatting + TSDF mesh extraction
- **환경**: `conda gs2m` (GCC11, CUDA11.8, NumPy<2)
- **출력 디렉토리**: `C:\Users\sdh97\Desktop\3d_results\real_test\gs2m_output\`

| 항목 | 값 |
|---|---|
| 학습 iter | 30,000 |
| L1 loss (최종) | 0.0199 |
| PSNR (최종) | 31.21 dB |
| TSDF 메시 | `train/ours_30000/mesh/tsdf_post.ply` ✅ |

### 메시 품질 평가 (재평가, Umeyama 카메라 포즈 정렬 기반)

> **평가 방법**: Umeyama 변환 (카메라 포즈 기반, scale=3.89×, ATE=2.3cm) → GT 큐브를 COLMAP 공간으로 역변환 → COLMAP 공간에서 직접 CD/F-score 계산
>
> **GT**: UE5 Cube8~12.ply (5개 큐브, X=1.235~1.587 COLMAP units)
>
> **참고**: 이전 평가(21-24cm)는 스크립트 미확인으로 재현 불가. 아래는 동일 방법으로 전 방법 재평가한 공정 비교임.

| 방법 | CD (cm) | F@1cm | F@5cm | F@10cm | 비고 |
|---|---|---|---|---|---|
| AGS baseline | **25.41** | 0.019 | 0.057 | **0.127** | ✅ CD 최저 |
| **GS-2M (ours)** | **27.79** | 0.004 | 0.029 | 0.057 | ← **NEW** |
| MILo SOR2 | 30.72 | 0.003 | 0.013 | 0.024 | — |
| MILo baseline | 37.20 | 0.003 | 0.010 | 0.018 | — |
| 2DGS | 37.25 | 0.003 | 0.019 | 0.036 | — |

> **해석**: CD 기준으로 GS-2M은 MILo/2DGS 대비 25-27% 개선. 5개 큐브 전체 씬에서 CD가 높은 것은 전체 씬 메시를 GT 소형 큐브와 비교하기 때문 (대부분의 메시 포인트가 큐브 표면과 거리가 멈).

### 스크립트

- **학습**: `/tmp/GS-2M/train.py` — `source ~/miniconda3/bin/activate gs2m && export CC=~/miniconda3/envs/gs2m/bin/gcc ...`
- **메시 추출**: `/tmp/GS-2M/render.py --extract_mesh --skip_test`
- **평가**: `/tmp/eval_gs2m.py` (Umeyama + COLMAP 공간 CD/F-score)

---

## 3m_1 드론 데이터 평가 파이프라인 설계 (2026-06-25 확정)

### 목적
실제 드론 촬영 데이터(GT 없음) → GS-2M / 2DGS / MILo 3모델 비교
- **정량**: PSNR / SSIM / LPIPS (NVS, held-out test views)
- **정성**: 메시 추출 후 시각 비교 (GT 없으므로 CD 불가)

### ❌ 첫 번째 시도 폐기 사유: focal 버그

MASt3R는 이미지를 **512px로 리사이즈**하여 처리하고 focal을 그 해상도 기준으로 저장.
변환 스크립트에서 원본 해상도(1920)로 스케일백을 누락.

| 항목 | 잘못된 값 | 올바른 값 |
|------|----------|----------|
| MASt3R 처리 해상도 | 512×288 | — |
| 저장된 focal | 359.88px | — |
| 스케일 팩터 | 누락 | 1920/512 = **×3.75** |
| 1920px 기준 focal | ❌ 359.88 (FOV 139°) | ✅ **1349.55** (FOV 71°) |

→ 카메라가 사실상 어안렌즈로 설정되어 학습 → PSNR 16dB (정상 22dB+)

### 데이터 준비 (재실행 필요)

- **소스**: sysai3 `~/Desktop/data/drone_real/3m_1/` (55장, 매 10프레임)
- **포즈**: `drone_real_sfm/3m_1/poses.npy` (55,4,4) c2w
- **COLMAP 변환 수정 사항**:
  - `focal = focals_npy[0] * (1920 / 512)` = **1349.55px**
  - cx=960, cy=540 (변경 없음)
  - images.txt: poses.npy 순서 = 이미지 정렬 순서 ✅ 확인됨
  - points3D.ply: 200k 다운샘플 (0pt 버그 수정 유지)

### 학습 설정 (확정)

| 항목 | 설정 | 비고 |
|------|------|------|
| 해상도 | **1920×1080** (코드 패치 필요) | `camera_utils.py` 1600 캡 제거 |
| eval | `--eval` (llffhold=8) | 7장 held-out test |
| iter | GS-2M/2DGS: 30k, MILo: 18k | 각자 수렴 스케줄 |
| points3D | 200k 다운샘플 PLY | |

> ⚠️ **1920 코드 패치**: GS-2M/2DGS/MILo 모두 `utils/camera_utils.py`에 `if orig_w > 1600: rescale` 하드코딩됨
> → 각 모델의 `camera_utils.py`에서 해당 분기 제거 또는 1920으로 임계값 상향 필요

### 평가 파이프라인 (3단계)

```
1. train.py --eval        → 학습 + PSNR 출력 (train 로그에서)
2. render.py              → test 뷰 렌더링
3. metrics.py             → SSIM / LPIPS 계산
```

> ⚠️ SSIM/LPIPS는 train 로그에 **안 나옴** — 반드시 render→metrics 실행 필요

### 메시 추출 방식 (모델별 상이, 정상)

| 모델 | 방식 | 실행 |
|------|------|------|
| GS-2M | depth 렌더 → TSDF fusion → Marching Cubes | `render.py --extract_mesh` |
| 2DGS | depth 렌더 → TSDF fusion → Marching Cubes | `render.py --mesh` |
| MILo | 학습된 occupancy/SDF → Marching Tetrahedra | `mesh_extract_sdf.py` |

> TSDF `voxel_size = max_depth/1024`, `sdf_trunc = 4×voxel` — 씬 스케일 자동 적응 (수동 튜닝 불필요)

### NVS 결과 (실행 후 업데이트 예정)

| 방법 | iter | PSNR (dB) | SSIM | LPIPS | 메시 |
|------|------|-----------|------|-------|------|
| GS-2M | 30k | - | - | - | TSDF |
| 2DGS | 30k | - | - | - | TSDF |
| MILo | 18k | - | - | - | SDF |
