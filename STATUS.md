# 현재 작업 현황 (2026-06-18)

> 새 세션에서 이 파일만 읽으면 지금 뭘 하고 있는지 파악 가능. 상세 배경은 `pipeline_strategy_3branches.md` 참고.

## 진행 중인 프로세스

없음 (blue_1 FHD MILo 학습 완료, 메시 추출 완료)

## blue_1 실험 결과 요약 (2026-06-18 기준)

### 720p 원본 이미지 기반 — 전부 실패
| 방법 | 결과 | 실패 원인 |
|---|---|---|
| Screened Poisson (MASt3R 점군 직접) | ❌ 울퉁불퉁 | 720p 저해상도 → MASt3R 포즈 오차 → 점군 노이즈 |
| PPSurf (CGF 2024, F1=0.92 SOTA) | ❌ 울퉁불퉁 | 동일 원인 (입력 점군 품질 한계) |
| SuGaR (기본 COLMAP sparse init) | ❌ 표면 붕괴 | photometric gradient 부족 |
| SuGaR + MASt3R prior (200k pts, PSNR 23.28) | ❌ 표면 붕괴 | prior 교체해도 GS 한계 동일 |
| MILo (SIGGRAPH Asia 2025) | ❌ 미시도 | FHD 先시도로 결정 |
| MeshSplatting (CVPR 2026) | ❌ 환경 구축 실패 | compile.sh setuptools<49.4 오류, 미해결 |
| GOF (SIGGRAPH 2024) | ❌ 환경 폐기 | Python 3.8 + CUDA 11.3 요구, 서버 환경과 불일치 |

### FHD 업스케일 이미지 기반 (2026-06-18)
| 방법 | 결과 | 출력 경로 |
|---|---|---|
| **MILo (SIGGRAPH Asia 2025)** | ✅ **학습+메시 추출 완료** (18000 iter, ~29분) | `~/Desktop/data/experiments/blue_1__milo_fhd/mesh_learnable_sdf.ply` |

> ⚠️ FHD MILo 메시 품질은 아직 육안 검사 전. MeshLab/CloudCompare로 확인 필요.

**근본 원인 분석:**
- 시뮬레이션(FHD)에서는 같은 파란 드론 잘 복원됨 → 단색 자체가 문제가 아님
- 실측 실패 원인: **①해상도 낮음(720p)** → MASt3R 포즈 오차 증가 → 점군 노이즈
- 추가 원인: 실제 조명 변화 / specular 반사 / 배경 노이즈
- **해결: Real-ESRGAN ×1.5 업스케일 후 FHD로 전체 파이프라인 재실행**

## 신규 데이터셋 (SYMA Z3 드론, 2026-06-16)

| 객체 | 장수 | 해상도 | SfM 상태 |
|---|---|---|---|
| blue_1 | 125 | 720p → ✅ FHD 업스케일 완료 | ✅ SfM 완료 / COLMAP 변환 완료 / MILo 학습+메시 추출 완료 |
| blue_2 | 106 | 720p | ⬜ 대기 |
| streetlight_low | 75 | 720p | ⬜ 대기 |
| blue_person | 38 | 720p | ⬜ 대기 (장수 부족, 참고용) |

## 다음 단계

1. **mesh_learnable_sdf.ply 품질 확인** — MeshLab/CloudCompare로 열어서 평가
2. **blue_2, streetlight_low 업스케일 + 파이프라인 진행** (동일 순서)
3. **MeshSplatting 시도** (venv-meshsplat, compile.sh setuptools 문제 해결 후)

## 유틸리티 스크립트 및 경로

### Real-ESRGAN — 이미지 업스케일 (신규, 2026-06-18)
- 위치: `~/Desktop/Real-ESRGAN/`
- venv: `~/Desktop/Real-ESRGAN/venv-esrgan/`
- 모델: `RealESRGAN_x2plus` (720p × 1.5 → FHD)
- 입력: `~/Desktop/data/datasets/blue_1/rgb/`
- 출력: `~/Desktop/data/datasets/blue_1/rgb_fhd/`
```bash
source ~/Desktop/Real-ESRGAN/venv-esrgan/bin/activate
cd ~/Desktop/Real-ESRGAN
python3 inference_realesrgan.py -n RealESRGAN_x2plus \
    -i ~/Desktop/data/datasets/blue_1/rgb/ \
    -o ~/Desktop/data/datasets/blue_1/rgb_fhd/ \
    --outscale 1.5
```

### `convert_pts.py` — MASt3R 점군 → COLMAP points3D.bin 교체
- 위치: `~/Desktop/convert_pts.py`
- MASt3R PLY(2.3M pts) → 200k 랜덤 다운샘플 → points3D.bin 교체
- 백업: `points3D.bin.bak`

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

- `venv-mast3r` / `venv-pointmap` 2개 ✅ 완료 (시스템 CUDA 12.3 기반 순수 venv)
- `venv-meshsplat` 🔄 구축 예정 (**MeshSplatting CVPR 2026 SOTA**: MILo 대비 PSNR +0.69dB, F1 +5.8%, 학습 2.2배 빠름, 메모리 2.5배 절감)
- `venv-milo` ✅ **완료** (Python 3.10, blue_1 FHD 학습 완료)
- `venv-sdf` ❌ **폐기** (2026-06-17, Neuralangelo 학습 시간 너무 김)
- `venv-gof` ❌ **삭제** (2026-06-17 SOTA 전환으로 GOF → MeshSplatting)
- conda `recon3d`는 torch/numpy 파일 손상으로 사용 중단, 보존만 함
- **주의**: 환경 전환 시 `unset LD_LIBRARY_PATH` 필수 (안 하면 torch CUDA 깨짐 — `pipeline_strategy_3branches.md` 공통주의사항 참고)
