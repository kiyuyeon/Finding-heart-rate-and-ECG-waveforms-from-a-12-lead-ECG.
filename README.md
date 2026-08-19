# ecg-hr-analyzer

12-lead ECG 신호에서 **PQRST 피크를 검출하고 심박수(HR)를 계산**하는 파이썬 도구입니다.
NeuroKit2 기반이며, 병원 계측기 출력과 같은 형식인 `.npy` 배열을 그대로 입력으로 받습니다.

---

## 동작 방식

한 리드(lead)의 1차원 신호가 들어오면 아래 순서로 처리합니다.

| 단계 | 함수 | 설명 |
|---|---|---|
| 1. 전처리 | `nk.ecg_clean(method='neurokit')` | 기저선 변동(baseline wander)과 고주파 잡음 제거 |
| 2. R 피크 검출 | `nk.ecg_peaks()` | QRS 복합체의 R 정점 위치(sample index) 추출 |
| 3. 파형 분할 | `nk.ecg_delineate(method='peak')` | R 피크를 기준으로 P, Q, S, T 피크 위치 추출 |
| 4. RR 간격 | `np.diff(r_peaks) / sr` | 인접 R 피크 간 시간 간격(초) |
| 5. 심박수 | `60 / mean(RR)` | 분당 심박수(bpm) |

검출 실패한 피크는 NaN으로 나오므로 `~np.isnan(...)`으로 걸러낸 뒤 정수 인덱스로 변환합니다.
R 피크가 2개 미만이면 RR 간격을 만들 수 없으므로 HR은 `NaN`을 반환합니다.

---

## 데이터

- 출처: [PTB-XL Electrocardiography Database (Kaggle)](https://www.kaggle.com/datasets/bjoernjostein/ptbxl-electrocardiography-database)
- 원본은 MATLAB(`.mat`) 형식이지만, 실제 병원 데이터가 `.npy`로 저장되어 있어 동일한 파이프라인을 쓰기 위해 `.npy`로 변환해 사용했습니다.
- 배열 형태: `(12, 5000)` — 12개 리드 × 5000 샘플
- 샘플링 주파수: **500 Hz** → 한 레코드당 **10초**
- 이 저장소에는 예시 레코드 2개(`09089_hr.npy`, `12750_hr.npy`)가 포함되어 있습니다.

리드 순서는 PTB-XL 표준을 따릅니다: `I, II, III, aVR, aVL, aVF, V1, V2, V3, V4, V5, V6`

---

## 설치

```bash
pip install numpy neurokit2
```

Python 3.8 이상을 권장합니다.

---

## 사용법

```python
from ecg_analyzer import ECGAnalyzer   # 노트북 코드 기준

# .npy 파일들이 들어 있는 디렉터리를 지정
analyzer = ECGAnalyzer("./", sr=500)

# 특정 레코드의 12개 리드를 모두 분석
results = analyzer.analyze_all_leads(sample_index=0)

for lead, (p, q, r, s, t, hr) in results.items():
    print(f"{lead}: R peaks={len(r)}, HR={hr:.1f} bpm")

# 특정 리드 하나만 분석 (0 = Lead I)
p, q, r, s, t, hr = analyzer.analyze_lead(sample_index=0, lead_index=0)
```

### 출력 예시

```
Lead 1:
P 피크: [ 558 1032 1536 2018 2559 3112 3645 4171]
Q 피크: [ 592 1096 1590 2083 2607 3166 3709 4232]
R 피크: [ 638 1116 1612 2102 2631 3187 3728 4251 4770]
S 피크: [ 702 1146 1649 2144 2669 3226 3770 4296]
T 피크: [ 779 1253 1753 2244 2769 3329 3872 4391]
심박수 (HR): 58.08
```

피크 값은 **샘플 인덱스**입니다. 초 단위로 바꾸려면 `sr`(500)로 나누면 됩니다.
예: R 피크 638 → 1.276초

---

## API

### `ECGAnalyzer(data_dir, sr=500)`

`data_dir` 안의 모든 `.npy` 파일을 읽어 `(N, 12, 5000)` 배열로 적재합니다.

| 메서드 | 반환 | 설명 |
|---|---|---|
| `load_npy_data(data_dir)` | `ndarray` | 디렉터리 내 `.npy` 전체를 float32 배열로 로드 |
| `calculate_rr_intervals(r_peaks)` | `ndarray` | R 피크 인덱스 → RR 간격(초) |
| `find_pqrst_and_calculate_hr(signal)` | `(p, q, r, s, t, hr)` | 1차원 신호 하나에 대한 피크 검출 + HR |
| `analyze_lead(sample_index, lead_index)` | `(p, q, r, s, t, hr)` | 특정 레코드의 특정 리드 분석 |
| `analyze_all_leads(sample_index)` | `dict` | `{'Lead 1': (...), ..., 'Lead 12': (...)}` |

---

## 저장소 구조

```
.
├── ecg_hr_analysis.ipynb   # ECGAnalyzer 구현 + 실행 예제
├── 09089_hr.npy            # 예시 ECG 레코드 (12 × 5000)
├── 12750_hr.npy            # 예시 ECG 레코드 (12 × 5000)
└── README.md
```

---

## 주의사항

- **리드마다 HR이 다르게 나올 수 있습니다.** R 피크가 뚜렷한 Lead I·II·V2 계열은 안정적이지만, Lead III·aVL처럼 진폭이 작은 리드는 잡음을 R 피크로 오인해 HR이 과대 추정될 수 있습니다. 임상적으로 신뢰할 값이 필요하면 여러 리드의 결과를 비교하거나 Lead II를 기준으로 쓰는 것을 권합니다.
- `sample_index`는 파일명이 아니라 **`os.listdir()`이 반환한 순서**를 따릅니다. 특정 파일을 확실히 지정하려면 파일 목록을 정렬해 두고 인덱스를 확인하세요.
- `ecg_delineate`는 검출에 실패한 지점을 NaN으로 채우므로, P/Q/S/T 피크의 개수가 R 피크 개수와 다를 수 있습니다.
- 이 코드는 **연구·학습용**입니다. 진단 목적으로 사용해서는 안 됩니다.

---

## 참고

- [NeuroKit2 문서](https://neuropsychology.github.io/NeuroKit/)
- Wagner et al., *PTB-XL, a large publicly available electrocardiography dataset*, Scientific Data (2020)
