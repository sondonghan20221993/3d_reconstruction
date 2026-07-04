# 파이프라인 개선 계획 (2026-07-05 확정)

> 배경: 시뮬레이션 검증 완료 후, 전체 파이프라인(SfM → 학습 → 메시 → 평가) 개선이 다음 목표.
> 이 문서는 후보 검토 결과와 확정된 실행 순서를 기록한다.

---

## 검토 결과 요약

| # | 후보 | 판정 | 근거 |
|---|---|---|---|
| ① | MASt3R 포즈 → COLMAP BA 재정렬 | ✅ **채택** | STATUS.md에 "천장 돌파 레버 1위"로 기록, 미시도. 하류 전 모델이 이득 |
| ② | scene graph retrieval 전환 | ✅ **채택 (승격)** | MASt3R-SfM 논문 Table 4가 직접 입증 (아래 상세) |
| ③ | hloc+COLMAP 분기 | ❌ 제외 | 오래된 기술, baseline 소개용으로만 언급 |
| ④ | 프레임 조밀화 (55→110장) | ❌ 제외 | 효과 자명, 실험 가치 낮음 |
| ⑤ | 실외 조명 대응 (appearance modeling) | ✅ 채택 (스터디 선행) | WildGaussians/bilateral grid 계열 조사 후 2DGS 적용 |
| ⑥ | PGSR | ❌ 제외 — **GS-2M 유지** | GS-2M이 PGSR 기반(개선 버전)임을 확인 → PGSR 계열 대표는 GS-2M으로 (아래 상세) |
| ⑨ | GT-free 평가 체계 | ✅ **채택** | 이미지만으로 가능 확인 (아래 지표표) |
| ⑩ | 파이프라인 자동화 + /tmp 스크립트 repo 이관 | ✅ 채택 | ⑨와 묶어 진행 |

---

## ② 근거 — MASt3R-SfM 논문 Table 4 (T&T 200뷰)

| 그래프 방식 | ATE↓ | RTA@5↑ | 쌍 수 |
|---|---|---|---|
| Complete | 0.0126 | 75.9 | 39,800 |
| **Local window (= swin)** | **0.0251** | **33.1** | 2,744 |
| Random | 0.0156 | 55.2 | 2,754 |
| **Retrieval (ASMK)** | **0.0124** | **70.9** | 2,758 |

- 같은 쌍 수 예산에서 **swin이 최하위 (무작위보다도 나쁨)** → 입력 순서 의존성이 논문 수치로 입증됨
- 논문의 "unordered collections 지원" 주장의 근거가 retrieval 그래프 = **retrieval 전환은 논문 기본 구성으로의 복귀**
- 현재 진행 중인 3m+5m 인터리빙 실험 판정 기준 (별도 실험 불필요):
  1. pointcloud.ply에서 박스 정합 여부 (CloudCompare)
  2. cross-altitude 매칭 쌍 수 (SfM 캐시에서 3m↔5m 쌍 카운트)
  3. 그룹 간 scale 비율 (시뮬에서 11.6% mismatch 측정한 방식 재사용)
- **무작위 > swin인 이유 (논문엔 설명 없음, 그래프 이론 해석)**: swin은 사슬 그래프 → 두 프레임 간 제약이
  수십 홉을 거치며 오차가 곱셈적으로 누적(drift, RTA@5 33.1이 증거). 무작위는 같은 엣지 수로도
  장거리 엣지가 생겨 그래프 지름이 O(log N)으로 줄고 loop closure 역할 → 오차가 상쇄됨.
  retrieval은 여기에 "겹침 보장"까지 더한 것. **박스 분리 = 경계 엣지 1개짜리 사슬 실패의 전형**이며,
  인터리빙(수동)과 retrieval(자동)은 같은 원리로 cross-altitude 장거리 엣지를 만드는 방법.
- 출처: [MASt3R-SfM (arXiv 2409.19152)](https://arxiv.org/abs/2409.19152)

## ② 실행 결과 (2026-07-05, 서버 sysai3에서 실제 실행 완료)

> ⚠️ **이 프로젝트는 지금까지 단 한 번도 retrieval을 쓴 적이 없었다** (swin-5만 사용, 2026-07-05 서버 로그/bash_history/배포 스크립트 전수 확인으로 검증됨).
> 아래는 2026-07-05에 처음으로 붙인 것이며, 실행 시각/경로/파라미터를 명확히 구분해 기록한다. 헷갈리지 말 것.

### 설치 (venv-mast3r, 서버 sysai3)
- `asmk` (github.com/jenicek/asmk, PyPI에 없어 소스 clone 후 `pip install . --no-build-isolation`) 설치 완료
- `faiss-cpu` 설치 완료
- retrieval 체크포인트: `~/Desktop/models/MASt3R-SLAM/checkpoints/MASt3R_ViTLarge_BaseDecoder_512_catmlpdpt_metric_retrieval_{trainingfree.pth,codebook.pkl}` → `~/Desktop/models/MAST3R_2/checkpoints/`로 복사 (동일 backbone, 파일명 규칙 일치 확인)
- `run_mast3r_sfm.py` 패치: `dust3r.image_pairs.make_pairs` → `mast3r.image_pairs.make_pairs`로 교체(retrieval 지원), `--scene_graph retrieval`, `--retrieval_model`, `--retrieval_na`(기본 20), `--retrieval_k`(기본 1) 인자 추가. 원본은 `run_mast3r_sfm.py.bak_before_retrieval`로 서버에 백업됨

### 실험 매트릭스 (서버 경로 기준, 절대 헷갈리지 말 것)

| 데이터셋 | scene_graph | 서버 출력 경로 | 상태 |
|---|---|---|---|
| 시뮬(3m+7m uniform, 34장) | swin-5 (기존) | `data/experiments/real_test_combined_uniform__mast3r/` | 기존 |
| 시뮬(3m+7m uniform, 34장) | retrieval-20-**1** | `data/experiments/real_test_combined_uniform__mast3r_retrieval/` | ❌ 7m 그룹 파탄 |
| 시뮬(3m+7m uniform, 34장) | retrieval-20-**5** | `data/experiments/real_test_combined_uniform__mast3r_retrieval_k5/` | ✅ **채택 후보** |
| 5m_1 (실제 드론, 55장) | swin-5 (기존) | `data/drone_real_sfm/5m_1/` | 기존 |
| 5m_1 (실제 드론, 55장) | retrieval-20-**1** | `data/drone_real_sfm_retrieval/5m_1/` | ✅ 문제 없음(궤도 매끄러움 확인) |
| 5m_1 (실제 드론, 55장) | retrieval-20-**5** | `data/drone_real_sfm_retrieval_k5/5m_1/` | 실행됨 (통일용, 결과는 아래) |

### 시뮬레이션 정량 결과 — per-group Umeyama (`tools/python/eval_retrieval_group_scale.py`)

| 지표 | swin-5 | retrieval-20-1 | **retrieval-20-5** |
|---|---|---|---|
| cross-altitude 쌍 비율 | 17.6% (30/170) | 36.8% (75/204) | 35.3% (94/266) |
| 3m ATE RMSE | 0.76cm | 0.63cm | 0.53cm |
| 7m ATE RMSE | 27.6cm | 384.33cm (파탄, 이상치 2개: 1317cm/767cm) | **2.56cm** |
| 7m ATE mean | (미측정) | 215.39cm | **2.15cm** |
| 그룹 간 scale mismatch | ~11.6% | 2.61% | **0.19%** |

**결론**: `retrieval-20-1`(k=1)은 그룹 간 scale mismatch는 고치지만 그룹 내부 로컬 밀도 부족으로 개별 프레임이 파탄남.
`retrieval-20-5`(k=5, swin의 winsize와 동일 밀도)가 두 문제를 모두 해결 — **swin 대비 7m ATE 10배 이상 개선, scale mismatch 사실상 해소**.
→ **retrieval-20-5를 앞으로 기본값으로 채택**.

### 5m_1(실제 드론) 결과 — 궤도 매끄러움 체크 (`tools/python/eval_trajectory_smoothness.py`, GT 없음)

5m_1은 55장 단일 궤도라 3m+7m 같은 그룹 분할이 없어 k=1의 약점이 애초에 안 드러남:

| 지표 | swin-5 | retrieval-20-1 |
|---|---|---|
| step 평균/표준편차 | 0.1901 / 0.0844 | 0.1945 / 0.0860 |
| step 최대 | 0.3840 | 0.3923 (동일 지점) |
| 중앙값 5배 초과 튐 | 0개 | 0개 |

k=1도 이미 정상. k=5는 파이프라인 파라미터 통일 목적으로 추가 실행.

## ⑥ 근거 — GS-2M은 PGSR 기반 → GS-2M으로 유지 (2026-07-05 확정)

- GS-2M 논문 원문: *"we construct GS-2M from PGSR to maintain SoTA reconstruction performance"*
- PGSR의 unbiased depth rendering + multi-view constraint를 그대로 상속, material(albedo/roughness) 분해를 추가한 것이 GS-2M
- **결정**: PGSR 계열 대표는 개선 버전인 GS-2M으로 유지, PGSR 별도 실행은 하지 않음
- (참고) 굳이 돌린다면 "GS-2M − material 분해" ablation 의미 — "BRDF 분해가 저텍스처 씬에서 ill-posed" 가설 직접 검증용
- 출처: [GS-2M (arXiv 2509.22276)](https://arxiv.org/abs/2509.22276), [PGSR (arXiv 2406.06521)](https://arxiv.org/abs/2406.06521)

## ⑨ GT-free 평가 지표 세트 (이미지+메시만으로 산출)

| 지표 | 입력 | 측정 대상 |
|---|---|---|
| held-out NVS PSNR/SSIM/LPIPS | 이미지 (llffhold=8) | 렌더링 품질 (시뮬에서 쓰던 방식 그대로) |
| 지면 planarity RMS | 메시 | 지면 RANSAC 평면 피팅 잔차 — "지면은 평평" 사전지식이 GT 역할 |
| floater 비율 | 메시 + 추정 포즈 | 카메라 가시영역 밖 vertex 비율 |
| 박스 분리 감지 | 점군/메시 | connected component 클러스터링 — 다고도 실험 성패 자동 판정 |
| 모델 간 상호 CD | 메시 2개+ | 방법 간 합의도 — outlier 방법 탐지 |

> ⚠️ NVS PSNR 단독은 기하를 못 잡음 (3DGS: PSNR 양호·메시 붕괴 사례) → **NVS + planarity + floater 세트**로 판정.
> 전부 open3d로 구현 가능.

---

## 실행 순서

1. **인터리빙 결과 판정** (진행 중, 위 ② 판정 기준 3개 적용)
2. **② `scene_graph=retrieval` 실행** — 3m+5m 동일 데이터로 swin-5 인터리빙과 비교 (논문 근거 확보됨, 인터리빙 결과와 무관하게 진행)
3. **① COLMAP BA 재정렬** — MASt3R 포즈/점군 → COLMAP DB → `point_triangulator` + `bundle_adjuster`. 효과 검증은 시뮬 데이터(GT)로 ATE 전후 비교
4. **⑨+⑩ 평가 스크립트 + 자동화** — `tools/eval_*` 로 repo에 정리, `/tmp` 산발 스크립트 이관, 원커맨드 파이프라인
5. **⑤ appearance modeling** — 스터디(WildGaussians, bilateral grid, GLO embedding) 후 2DGS 적용 판단
