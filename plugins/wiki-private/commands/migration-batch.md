---
description: 레거시 PARA 노트를 도메인 단위로 이관합니다. "레거시 이관", "PARA 마이그레이션", "migration batch", "_legacy 정리" 등의 요청 시 사용합니다.
---

# Migration Batch 워크플로우

**`_legacy/` 하위 PARA 구조 노트를 도메인 단위로 LLM-wiki 구조로 이관합니다.**

---

## Phase A: 이관 계획

### A-1. 현황 파악

```
Use Bash tool:
- command: "find _legacy/ -name '*.md' | wc -l"
```

```
Use Glob tool:
- pattern: "_legacy/**/*.md"
```

전체 레거시 노트 수와 디렉토리 구조를 파악합니다.

### A-2. 도메인 선택

이관 우선순위 (기본):

| 순위 | 도메인 | 예상 규모 | 레거시 경로 |
|------|--------|----------|------------|
| 1 | ai-llm | ~450 | `_legacy/2.Resource/0.LLM` |
| 2 | readings | ~330 | `_legacy/2.Resource/2.읽은것들` |
| 3 | engineering | ~300 | `_legacy/2.Resource/{apps, python, ...}` |
| 4 | finance | ~120 | `_legacy/2.Resource/Economics` + `_legacy/0.Project/{auto-stock, ...}` |
| 5 | work | ~280 | `_legacy/0.Project/CUBIG*` + `_legacy/3.Archive/{wavus, ...}` |
| 6 | career | ~70 | |
| 7 | journal | ~130 | |
| 8 | life | ~40 | |
| 9 | data-ml | ~40 | |
| 10 | research | | |

사용자에게 대상 도메인 확인 후 진행합니다.

### A-3. 배치 분할

- **세션당 30~50 노트**
- 대상 도메인의 `_legacy/` 노트를 스캔하여 배치 구성
- 배치 계획 제시 → 사용자 승인

---

## Phase B: 순차 이관

각 노트에 대해 `wiki-migrate` skill을 호출합니다.

```
Use Skill tool (각 노트마다):
- skill: "wiki-private:wiki-migrate"
- args: "<_legacy/경로>"
```

**배치 간 규칙:**
- 10~15노트마다 중간 보고
- 타입 판정 애매한 노트는 `_inbox/` 배치 후 보고
- 에러 발생 시 건너뛰고 기록

---

## Phase C: Sanity Check

배치 완료 후 해당 도메인 점검:

```
Use Skill tool:
- skill: "wiki-private:wiki-lint"
- args: "<domain>"
```

특히 확인:
- 고아 페이지 (새로 이관된 페이지 중 링크 누락)
- 깨진 wikilink (미이관 대상으로의 링크)
- index 등재 누락

---

## Phase D: 완료 처리

도메인 전체 이관 완료 시:

1. `log_<domain>.md`에 `## [YYYY-MM-DD] migrate_complete | <domain>` 마커 기록
2. 완료 기준 확인:
   - [ ] 해당 `_legacy/` 경로 노트 모두 이관 또는 `_inbox/` 배치
   - [ ] `index_<domain>.md` 카탈로그 최신화
   - [ ] 고아 페이지 < 전체의 5%
   - [ ] `log_<domain>.md`에 `migrate_complete` 이벤트

---

## 보고

```
# Migration Report — <domain> ([YYYY-MM-DD])

## Summary
| 항목 | 수 |
|------|---|
| 총 대상 노트 | N |
| 이관 완료 | N |
| _inbox 배치 | N |
| 건너뜀 (에러) | N |

## 타입별 분포
| Type | Count |
|------|-------|
| entity | N |
| concept | N |
| source-summary | N |
| comparison | N |
| note (_inbox) | N |

## 주의 사항
- ...
```

---

## 주의사항

- 한 세션에 2개 도메인 동시 이관 금지
- `99.attached/` 경로 변경 금지
- `_legacy/` 원본 삭제 금지 (Phase 4 전까지)
- 전역 find/replace로 wikilink 일괄 치환 금지
