# Applicability Guard Rules

> 파이프라인 코드(data_models/, modules/, pipeline/) 작성 시 반드시 준수할 3가지 규칙

## R1. 계산 전 적용 가능성 확인 (Applicability Check)

**원칙**: "계산 가능"과 "계산해야 함"은 다르다.

| 금지 | 허용 |
|------|------|
| dtype만 보고 바로 계산 | dtype → **적용 판단** → 계산 or skip |
| string에 min/max 산출 | string → min/max = None |
| ID 컬럼에 categorical_distribution | cardinality 확인 후 skip |

**적용 기준**:
- numeric이 아니면: mean, std, skewness = None
- cardinality > 30% (unique_ratio)이면: categorical_distribution = None
- datetime/timedelta/period이면: categorical_distribution = None

## R2. 모르면 비워라 (No Guessing)

**원칙**: 빈 값이 틀린 값보다 낫다.

| 금지 | 허용 |
|------|------|
| 키워드 매칭으로 domain 추론 | domain = "" |
| unique_count 기반 task_type 추측 | task_type = "" |
| fallback 0.5 / 1.0 하드코딩 | None 반환 |
| 30자 truncation으로 이름 생성 | dataset_name = "" |

**적용 대상**:
- `_extract_domain()` → ""
- `_determine_task_type()` → ""
- `_determine_target_type()` → ""
- `_compute_positive_ratio()` → None (binary + 1이 존재하는 경우만 계산)
- `_generate_dataset_name()` → ""
- target trust fallback → dist_stability = 0.0 (1.0 낙관 금지)

## R3. 역할 기반 필터링 (Role-based Filtering)

**원칙**: 집계 시 target/index/useless 컬럼 제외, feature 컬럼만 대상.

| 금지 | 허용 |
|------|------|
| ALL 컬럼의 null_ratio 평균 | feature only |
| ALL 컬럼의 skewness 추출 | feature only |
| target에 edit_suggestion 생성 | feature only |

**적용 대상**:
- `_extract_null_ratios()` → feature only
- `_extract_skewness_values()` → feature only
- `_extract_outlier_ratios()` → feature only
- `null_ratios_dict`, `feature_importances_dict` → feature only
- `_build_edit_suggestions()` → feature only (target/idx/useless 제외)

## 요약

```
산출 전에 항상 자문하라:
1. 이 컬럼의 dtype/role에 이 지표가 적합한가? (R1)
2. 이 값을 확신할 수 있는가? 아니면 비워야 하는가? (R2)
3. 이 집계에 target/index 컬럼이 포함되어도 되는가? (R3)
```
