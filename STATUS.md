# 현재 작업 현황 (2026-06-19)

> 새 세션에서 이 파일만 읽으면 지금 뭘 하고 있는지 파악 가능. 상세 배경은 `pipeline_strategy_3branches.md` 참고.
> real_test 복원 상세: `real_test_reconstruction.md` 참고.

## 진행 중인 프로세스

없음 (blue_1 FHD 표면 복원 실험 1라운드 완료)

---

## real_test 포즈 비교 실험 (2026-06-19 완료)

| 항목 | 상태 |
|---|---|
| MASt3R-SfM | ✅ 완료 (34장, 149만 pts) |
| VGGT 포즈 + 포인트 클라우드 | ✅ 완료 (160만 pts) |
| 포즈 정확도 비교 (RPE/ATE) | ✅ 완료 |
| 결과 PLY 로컬 다운로드 | ✅ `D:\3d_reconstruction\` |

**결과 요약 (RPE 기준):**
- 회전: MASt3R 21.9° vs VGGT 29.3° → **MASt3R 우세**
- 위치: MASt3R 2.68 vs VGGT 2.57 → **VGGT 근소 우세 (무의미한 차이)**
- 상세: `D:\3d_reconstruction\pose_comparison_result.md`

> AirSim(NED)↔OpenCV 좌표계 차이로 ATE 회전 오차(~120°)는 무의미. RPE 기준으로 판단.

**VGGT 성능 저하 원인:**
1. 비정사각형 입력 (294×518 ≠ 학습 기준 518×518)
2. object-centric 드론 장면 → 학습 분포 밖
3. Bundle Adjustment 없음

---

## blue_1 실험 결과 요약 (2026-06-19 업데이트)

### 720p 원본 이미지 기반 — 전부 실패
| 방법 | 결과 | 실패 원인 |
|---|---|---|
| Screened Poisson (MASt3R 점군 직접) | ❌ 울퉁불퉁 | 720p 저해상도 → MASt3R 포즈 오차 → 점군 노이즈 |
| PPSurf (CGF 2024, F1=0.92 SOTA) | ❌ 울퉁불퉁 | 동일 원인 (입력 점군 품질 한계) |
| SuGaR (기본 COLMAP sparse init) | ❌ 표면 붕괴 | photometric gradient 부족 |
| SuGaR + MASt3R prior (200k pts, PSNR 23.28) | ❌ 표면 붕괴 | prior 교체해도 GS 한계 동일 |
| GOF (SIGGRAPH 2024) | ❌ 환경 폐기 | Python 3.8 + CUDA 11.3 요구, 서버 환경 불일치 |

### FHD 업스케일 이미지 기반 (2026-06-18~19)

| 방법 | 결과 | 로컬 파일 | 비고 |
|---|---|---|---|
| **MILo (SIGGRAPH Asia 2025)** | ✅ 완료 | `blue_1_fhd_milo.ply` (18MB) | 육안 품질 검사 대기 |
| **Screened Poisson (FHD 점군)** | ✅ 완료, 형태 식별 가능 | `mesh_poisson_fhd.ply` (55MB) | 박스 윤곽 나옴, 지면 격자 아티팩트 |
| **MeshSplatting (CVPR 2026)** | ✅ 학습 완료, ❌ 메시 파편화 | `blue_1_fhd_meshsplat.ply` (901MB) | PSNR 21.4, 결과 사용 불가 |

#### MeshSplatting 실패 원인 분석
1. **Photometric Loss Failure** — 단색 파란 박스에서 표면 위치와 무관하게 색상 오차 ≈ 0 → 기하학적 gradient 없음
2. **TSDF 후처리** — depth map 일관성 부족 시 TSDF fusion이 파편 생성
3. **GS 기반 구조적 한계** — object-centric + 텍스처리스 장면 = MeshSplatting/SuGaR 공통 실패 패턴

> **결론: GS 기반 메시 추출(MeshSplatting, SuGaR)은 이 데이터셋 유형에 구조적으로 부적합. Screened Poisson 또는 MILo(SDF 병행)가 현실적.**

---

## 신규 데이터셋 (SYMA Z3 드론, 2026-06-16)

| 객체 | 장수 | 해상도 | 상태 |
|---|---|---|---|
| blue_1 | 125 | 720p → FHD 업스케일 완료 | ✅ SfM / COLMAP / MILo 학습+메시 추출 완료 |
| blue_2 | 106 | 720p | ⬜ 대기 |
| streetlight_low | 75 | 720p | ⬜ 대기 |
| blue_person | 38 | 720p | ⬜ 대기 (장수 부족, 참고용) |

---

## blue_1 VGGT 실험 (2026-06-19)

> **VGGT는 결과 불량 — 이후 사용 안 함**

| 실험 | 출력 파일 | 포인트 수 | 로컬 파일명 |
|---|---|---|---|
| VGGT 720p | `experiments/blue_1__vggt/pointcloud_vggt.ply` | 361만 pts | `blue_1_720p_vggt.ply` (52MB) |
| VGGT FHD | `experiments/blue_1__vggt_fhd/pointcloud_vggt.ply` | 728만 pts | `blue_1_fhd_vggt.ply` (105MB) |

GT 포즈 없는 실측 데이터라 정량 비교 불가. 포인트 클라우드 품질만 육안 확인 가능.

---

## 다음 단계

1. **blue_1 MILo FHD 메시 품질 확인** — `blue_1_fhd_milo.ply` MeshLab/CloudCompare로 육안 검사 (Screened Poisson과 비교)
2. **blue_2, streetlight_low** 업스케일 + 파이프라인 진행
3. **2DGS** — GS 기반 중 표면 정렬에 특화, 시도 검토 중

---

## 완료된 환경 구축

| venv | 상태 | 용도 |
|---|---|---|
| `venv-mast3r` | ✅ | MASt3R-SfM |
| `venv-pointmap` | ✅ | 포인트 클라우드 처리 / Screened Poisson |
| `venv-milo` | ✅ | MILo 메시 복원 |
| `venv-vggt` | ✅ (2026-06-19 신규) | VGGT 포즈+포인트 클라우드 |
| `venv-meshsplat` | ❌ Python 3.8 / effrdel 호환 불가 | 구버전, 미사용 |
| `venv-meshsplat10` | ✅ (2026-06-19 신규, Python 3.10) | MeshSplatting CVPR 2026 — 빌드 완료 |
| `venv-sdf` | ❌ 폐기 | Neuralangelo (학습 시간 너무 김) |
| `venv-gof` | ❌ 삭제 | GOF → MeshSplatting으로 대체 |

- conda `recon3d`: torch/numpy 파일 손상으로 사용 중단, 보존만 함
- **주의**: 환경 전환 시 `unset LD_LIBRARY_PATH` 필수
- `venv-meshsplat10` 의존성: effrdel의 `tetutils.py`, `intersect.py`를 site-packages에 수동 복사 필요 (find_packages 누락)

---

## 유틸리티 스크립트

| 스크립트 | 위치 | 용도 |
|---|---|---|
| `run_mast3r_sfm.py` | `~/Desktop/MAST3R_2/` | MASt3R-SfM 실행 |
| `mast3r_to_colmap.py` | `~/Desktop/MAST3R_2/` | COLMAP 형식 변환 |
| `run_vggt_poses.py` | `~/Desktop/` | VGGT 포즈+PLY 추출 |
| `compare_poses.py` | `~/Desktop/` | GT vs MASt3R vs VGGT 포즈 비교 → MD 출력 |
| `convert_pts.py` | `~/Desktop/` | MASt3R PLY → COLMAP points3D.bin |
| `visualize_reconstruction.py` | `~/Desktop/` | SfM 결과 3D 시각화 |
| `inference_realesrgan.py` | `~/Desktop/Real-ESRGAN/` | 720p → FHD 업스케일 |
