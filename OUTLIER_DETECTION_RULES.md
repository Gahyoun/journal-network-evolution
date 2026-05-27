# Outlier 저널 탐지·제거 규칙 (v2)
# Outlier Journal Detection & Filtering Rules (v2)

---

## 1. 목적 / Purpose

**KR.** 4분야(Physics / Chemistry / Biology / Math) 교수 데이터에서 OpenAlex 동명이인 오집계로 잘못 붙은 타분야 저널 논문을 제거한다. 본인 분야 내의 새로운 세부분야 진출은 보존한다.

**EN.** Remove journal papers that were incorrectly attributed to a professor due to OpenAlex same-name collisions across disciplines (Physics / Chemistry / Biology / Math), while preserving genuine forays into new sub-fields within the professor's own discipline.

---

## 2. 판정 단위 / Unit of Decision

**KR.** `(prof_id, venue_id)` **쌍** 단위. 저널(venue) 노드 자체는 제거하지 않는다 — 같은 저널을 정당하게 쓰는 다른 교수의 데이터는 보존된다.

**EN.** Per `(prof_id, venue_id)` **pair**. The venue node itself is not removed — data for other professors who legitimately use the same venue is preserved.

---

## 3. 판정 조건 (AND 결합) / Flag Conditions (AND-combined)

### 3-1. `PPR_low` — 저자 분야 커뮤니티로부터 거리 / Distance from author's discipline community

**KR.** 각 안정 해상도(stable gamma)에서, 그 분야(Physics/Chemistry/Biology/Math)의 **primary 커뮤니티 상위 2개**를 시드로 Personalized PageRank를 돌려 venue의 PPR 점수를 구한다. 다음 중 하나면 `PPR_low = True`:

- `z-score < -2`: 그 교수가 게재한 venue들 사이에서 상대적으로 낮음
- `PPR 점수 < 전체 하위 5% 분위`: 절대적으로 커뮤니티 코어에서 멀다

**EN.** At each stable gamma, run Personalized PageRank seeded by the **top-2 primary communities** of the discipline. `PPR_low = True` if either:

- `z-score < -2`: low relative to other venues the professor has published in
- `PPR score < global 5th-percentile cutoff`: absolutely far from any community core

### 3-2. `RARE` — 교수집단 전체 게재 희소성 / Rarity across the entire professor body

**KR.** 4분야 교수 **전체**가 그 venue에 게재한 총 논문 수 < **10편** 이면 `RARE = True`.

**EN.** `RARE = True` if the total number of papers in that venue across the entire 4-discipline professor body is **< 10**.

### 3-3. `FLAG` (단일 gamma 기준) / Per-gamma flag

```
FLAG_gamma = PPR_low AND RARE
```

### 3-4. Consensus (다중 gamma 통합) / Multi-gamma consensus

**KR.** 7개 유효 stable gamma 중 **≥ 50%** 에서 FLAG → 최종 outlier로 확정.

**EN.** Final outlier if FLAG holds for **≥ 50%** of the 7 valid stable gammas.

---

## 4. 처리 매트릭스 / Decision Matrix

| 케이스 / Case | PPR_low | RARE | 결과 / Result |
|---|---|---|---|
| 수학교수 ← 의학 동명이인 (정형외과 저널) / Math prof ← orthopedist namesake | ✅ | ✅ | **drop** ✅ |
| 수학교수 ← 건축 동명이인 (건축 저널) / Math prof ← architect namesake | ✅ | ✅ | **drop** ✅ |
| 수학교수의 신규 세부분야 (매듭이론 저널 1편) / Math prof's new sub-field (Knot Theory) | ❌ | — | keep ✅ |
| 다른 수학교수 동명이인 (수학 저널) / Other math prof namesake (math journal) | ❌ | — | keep ✅ |
| 같은 분야 메이저 저널 (정상 게재) / Same-discipline major venue | ❌ | ❌ | keep ✅ |
| 4분야 내 타분야 메이저 저널 (예: 수학 ← PRL 1편) / Cross-disc major venue (rare in practice) | ✅ | ❌ | keep ⚠️ |

**KR. 마지막 행 주의**: 4분야 안에서 메이저 venue에 오집계된 케이스는 RARE = False 라서 못 잡는다. 다만 실데이터에서 그런 케이스는 **0건** 으로 확인됨 (메이저 venue는 항상 본인 분야 교수들도 함께 쓰는 패턴).

**EN. Note on the last row**: Cross-discipline misattributions to *major* venues within the 4 disciplines are not caught (RARE = False). However, in the actual dataset such cases are **0** (major venues always have at least some same-discipline users).

---

## 5. 결과 요약 (2026 spring) / Result Summary (2026 spring)

| | |
|---|---|
| 전체 평가 (prof, venue) 쌍 / Total pairs evaluated | 133,302 |
| 최종 outlier 쌍 / Final outlier pairs | **5,692** |
| 영향 받는 논문 수 / Papers affected | **7,698** |
| 영향 받는 교수 수 / Profs affected | 1,617 |
| outlier venue 의 평균 global_papers | 4.9 (max 9) |

**분야별 / By discipline**

| Discipline | Outlier pairs | Papers | Profs |
|---|---|---|---|
| Biology | 2,574 | 3,476 | 710 |
| Chemistry | 1,341 | 1,763 | 382 |
| Math | 972 | 1,353 | 249 |
| Physics | 805 | 1,106 | 276 |

---

## 6. 컷오프 (기본값) / Cutoffs (defaults)

| Parameter | Value | 의미 / Meaning |
|---|---|---|
| `--z-thresh` | `-2.0` | 교수 내 z-score 하한 / Within-prof z-score cutoff |
| `--pct-thresh` | `5.0` | 전체 PPR 점수 하위 백분위 / Global PPR percentile cutoff |
| `--min-global-papers` | `10` | RARE 기준: 4분야 합산 게재 수 / RARE threshold across 4 disciplines |
| `--consensus-thr` | `0.5` | gamma 다수결 비율 / Multi-gamma majority ratio |
| `--min-papers` | `2` | 교수당 최소 논문 수 / Minimum papers per prof |
| `--min-nc` | `5` | gamma 스킵 기준 (커뮤니티 수) / Skip gamma if fewer communities |
| `--alpha` | `0.15` | PPR teleport 확률 / PPR teleport probability |
| `--top-k-seeds` | `10` | 커뮤니티별 PPR 시드 수 / PPR seeds per community |
| `--edge-per-node` | `15` | 노드당 유지 엣지 수 / Edges kept per node |

---

## 7. 적용 방식 / Application

**KR.** outlier 쌍은 `build_combined_html.py` 의 works 로드 직후에 한 줄 필터로 제거된다. 그래프 노드 / 엣지 / 인접행렬 / 커뮤니티 구조는 원본 그대로 유지.

**EN.** Outlier pairs are removed by a single-line filter immediately after loading the works data in `build_combined_html.py`. Graph nodes / edges / adjacency matrix / community structure remain untouched.

```bash
python build_combined_html.py \
  --pkl    community_consensus.pkl \
  --npz    cooccurrence_matrix.npz \
  --works  openalex_professor_works_2026spring_api.csv \
  --phys   "../2026 spring Physics.xlsx" \
  --chem   "../2026 spring Chemistry.xlsx" \
  --bio    "../2026 spring Biology.xlsx" \
  --math   "../2026 spring math.xlsx" \
  --outlier-csv ./output/outlier_pairs_v2.csv \
  --out    ./output
```

`--no-drop-outliers` 로 필터를 끌 수 있다 / Use `--no-drop-outliers` to disable.

---

## 8. 서버 업로드 파일 / Files to Upload to Server

### 8-1. 신규 / 수정된 코드 / New / modified code

| 파일 / File | 상태 / Status | 설명 / Description |
|---|---|---|
| `Journal project/detect_outliers_v2.py` | **신규 / NEW** | PPR + RARE AND-결합 outlier 탐지기 / PPR + RARE AND-combined detector |
| `Journal project/output/build_combined_html.py` | **수정 / MODIFIED** | `--outlier-csv` 인자 + works 필터 추가 / Added `--outlier-csv` arg & works filter |
| `Journal project/name_communities.py` | **신규 / NEW** | 759개 커뮤니티 자동 명명 (휴리스틱+OVERRIDES) / Auto-name 759 communities (heuristics + overrides) |
| `Journal project/community_top_journals_named_v2.csv` | **신규 / NEW** | 모든 stable γ × 커뮤니티 영어 라벨 / English labels for every (γ, community) |
| `Journal project/OUTLIER_DETECTION_RULES.md` | **신규 / NEW** | 본 문서 / This document |

### 8-2. 서버에서 생성되는 산출물 (재실행 가능, 동기화 선택) / Server-side outputs (reproducible, sync optional)

| 파일 / File | 크기 / Size | 설명 / Description |
|---|---|---|
| `Journal project/output/outlier_pairs_v2.csv` | ~0.5 MB | 최종 drop 대상 (prof, venue) 쌍 5,692개 / Final 5,692 outlier pairs |
| `Journal project/output/outlier_pairs_v2_detail.csv` | ~70 MB | gamma별 + `ppr_low`/`rare` 컬럼 (디버깅) / Per-gamma debug detail |
| `Journal project/output/outlier_summary_v2.csv` | ~0.4 MB | 교수별 요약 1,617명 / Per-prof summary |

### 8-3. 폐기 권장 / Recommended to discard

| 파일 / File | 사유 / Reason |
|---|---|
| `Journal project/build_filtered_htmls.py` | 초기 venue-node-삭제 접근 — pair-필터 방식으로 전환 후 불필요 / Initial venue-node-deletion approach — superseded by pair-filter |

---

## 9. 재실행 / Reproduce

```bash
cd "Journal project"

# 1. outlier 탐지 / Detect outliers
python detect_outliers_v2.py \
  --pkl    community_consensus.pkl \
  --npz    cooccurrence_matrix.npz \
  --works  openalex_professor_works_2026spring_api.csv \
  --phys   "../2026 spring Physics.xlsx" \
  --chem   "../2026 spring Chemistry.xlsx" \
  --bio    "../2026 spring Biology.xlsx" \
  --math   "../2026 spring math.xlsx" \
  --out    ./output \
  --min-global-papers 10

# 2. 커뮤니티 명명 (한 번만, 새 stable γ 추가될 때만 재실행) / Name communities
python name_communities.py

# 3. HTML 빌드 (outlier 제거 + 커뮤니티 이름 적용) / Build HTML
python output/build_combined_html.py \
  --pkl    community_consensus.pkl \
  --npz    cooccurrence_matrix.npz \
  --works  openalex_professor_works_2026spring_api.csv \
  --phys   "../2026 spring Physics.xlsx" \
  --chem   "../2026 spring Chemistry.xlsx" \
  --bio    "../2026 spring Biology.xlsx" \
  --math   "../2026 spring math.xlsx" \
  --named  ./community_top_journals_named_v2.csv \
  --outlier-csv ./output/outlier_pairs_v2.csv \
  --out    ./output
```

---

## 10. 변경 이력 / Changelog

- **v2 (2026-05-27)**: AND 결합 (PPR_low ∧ RARE) + (prof, venue) 쌍 필터 / AND-combined criterion + pair-level filtering
- **v1 (이전)**: PPR_low 만 (OR) + venue 노드 통째 삭제 — 정상 게재 8,770편 손실 발생 / PPR_low only (OR) + venue node deletion — caused 8,770 legitimate papers lost
