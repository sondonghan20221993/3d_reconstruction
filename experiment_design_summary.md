# 3D Mesh 복원 실험 설계 요약

## 실험 목표

실외 건물 이미지 200~300장 → **시각적 품질 최우선 3D mesh 복원**

---

## 실험 구조

### 공통 후반 파이프라인 (A/B 동일)

```
카메라 pose + point cloud
  ↓
2DGS 학습
  ↓
SuGaR (mesh 추출 → refinement)
  ↓
최종 mesh (텍스처 포함)
```

### 실험 A (메인) — hloc + COLMAP

```
이미지 200~300장
  ↓
hloc (SuperPoint+LightGlue) → COLMAP
  ↓
카메라 pose + sparse point cloud
  ↓
공통 후반 파이프라인
```

### 실험 B (비교) — MASt3R-SfM

```
이미지 200~300장 (동일 이미지셋)
  ↓
MASt3R-SfM
  ↓
카메라 pose + dense point cloud
  ↓
공통 후반 파이프라인 (동일 코드 재사용)
```

### 비교 변수

- **유일한 차이**: pose + point cloud 생성 방법만 다름
- **비교 기준**: 최종 mesh 시각적 품질 (텍스처, 노이즈, artifact)
- 수치적 정확도(Chamfer distance 등)는 평가 기준 아님

---

## 서버 환경

| 항목 | 내용 |
|------|------|
| GPU | NVIDIA RTX A6000 (VRAM 48GB) |
| Driver / CUDA | 545.23.08 / CUDA 12.3 |
| OS | Ubuntu (20.04 추정) |
| 사용자 | sdh |
| conda | `/home/sdh/miniforge3` (miniforge3 기준) |
| 가상환경 | `recon3d` (Python 3.10) |

---

## 설치된 패키지 및 경로

| 패키지 | 경로 | 비고 |
|--------|------|------|
| PyTorch | conda env recon3d | 2.1.0+cu118 |
| nvcc | conda env recon3d | 11.8 |
| COLMAP | conda env recon3d | 3.11.1, CUDA 지원 |
| hloc | `~/Desktop/Hierarchical-Localization` | pip -e 설치 |
| 2DGS | `~/Desktop/2d-gaussian-splatting` | setup.py로 빌드 |
| SuGaR | `~/Desktop/SuGaR` | nvdiffrast 포함 |
| MASt3R | `~/Desktop/MAST3R_2` | PYTHONPATH 등록 |
| MASt3R checkpoint | `~/Desktop/MAST3R_2/checkpoints/` | DUSt3R_ViTLarge_BaseDecoder_512_dpt.pth |

---

## 환경 변수 (.bashrc 등록됨)

```bash
# conda 활성화 (매번 필요)
source /home/sdh/miniforge3/etc/profile.d/conda.sh
conda activate recon3d

# MASt3R PYTHONPATH (.bashrc에 영구 등록됨)
export PYTHONPATH=$PYTHONPATH:~/Desktop/MAST3R_2:~/Desktop/MAST3R_2/dust3r
```

---

## 주의사항

| 항목 | 내용 |
|------|------|
| .bashrc 문법 오류 | 3번째 줄 오류로 `source ~/.bashrc` 실패 → 매 셸 시작 시 `source /home/sdh/miniforge3/etc/profile.d/conda.sh` 수동 실행 필요 |
| numpy 버전 | `<2`로 고정 (2DGS 빌드 시 다운그레이드) → opencv-python 충돌 가능성 있으나 현재 무시 |
| MASt3R 설치 | setup.py/pyproject.toml 없는 구버전 → PYTHONPATH로 해결 |

---

## 디렉토리 구조 (권장)

```
3d_reconstruction/
├── images/                  # 원본 이미지 200~300장
├── experiment_A/
│   ├── colmap_output/       # pose + sparse point cloud
│   ├── 2dgs_output/         # .ply
│   └── sugar_output/        # mesh .obj/.glb
├── experiment_B/
│   ├── mast3r_output/       # pose + dense point cloud
│   ├── 2dgs_output/         # .ply
│   └── sugar_output/        # mesh .obj/.glb
└── experiment_design_summary.md
```

---

## 다음 단계

```
Step 1  이미지 준비 (서버에 업로드)

Step 2  실험 A 실행
        hloc feature matching
          → COLMAP pose 추정
          → 2DGS 학습
          → SuGaR mesh 추출 + refinement
          → 최종 mesh 시각적 검토

Step 3  실험 B 실행
        MASt3R-SfM으로 pose + point cloud 생성
          → 이후 2DGS / SuGaR는 실험 A와 동일 설정 재사용
          → 최종 mesh 시각적 검토

Step 4  두 실험 결과 비교
        - 텍스처 선명도
        - 표면 노이즈 / 구멍 유무
        - artifact 유무
        - 전반적 완성도
```
