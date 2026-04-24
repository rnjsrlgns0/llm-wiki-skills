---
name: wiki-lint
description: Use when the user says "위키 점검해줘", "lint <domain>", "위키 건강검진", "vault 정리", or after a large ingest batch (>20 sources). Runs the Karpathy LLM-wiki lint checklist on one domain — detects contradictions, stale claims, orphan pages, missing cross-refs, data gaps, index drift — and records findings in log_<domain>.md.
---

# Wiki Lint Skill

Karpathy LLM-wiki 패턴(https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)의 **lint 워크플로우**를 실행한다.

> "The LLM is good at suggesting new questions to investigate and new sources to look for. This keeps the wiki healthy as it grows." — Karpathy

lint는 단순 검증이 아니라 **위키가 건강하게 성장하도록** 다음 ingest·query의 방향을 제안하는 단계다.

## When to use

- 주기 점검: 도메인당 월 1회
- 대량 ingest 직후 (> 20 소스) 해당 도메인
- 사용자 명시 요청: `/wiki-lint <domain>` 또는 "위키 점검"

## Source of truth

- `.claude/rules/60_lint.md` — 상세 체크리스트
- `.claude/rules/10_frontmatter.md` — frontmatter 유효성 기준
- `.claude/rules/20_tags.md` — 태그 네임스페이스 검증

## Scope

- **기본**: 한 번에 한 도메인 (`wiki/<domain>/`)
- **전역 lint**: 페이지 총수 ~2000 이하일 때만 권장
- 시작 전에 대상 도메인 사용자에게 확인 (여러 후보일 때)

## Procedure — 8개 체크리스트

### Check 1. 고아 페이지
- **정의**: 인바운드 wikilink 0 인 페이지
- **탐지**: 도메인 내 전체 .md 대상으로 `[[<id>]]` 참조 수 집계
- **조치 제안**:
  - 관련 페이지에 링크 추가 권유
  - 오래되고 가치 낮으면 `status: archived` 강등
  - 삭제 권유 (사용자 승인)

### Check 2. 모순 (Karpathy 1순위)
- **정의**: 동일 entity/concept에 대한 상충 주장을 가진 페이지 쌍
- **탐지 힌트**:
  - 같은 entity를 인용하는 여러 source-summary 비교
  - entity/concept 본문 내 모순 키워드("그러나", "반대로")와 인용 충돌
- **조치 제안**: 최신 소스 기준 병합 또는 "논쟁 중" 섹션 신설

### Check 3. Outdated (진부한 주장)
- `status: outdated` 페이지 전부 수집
- + `status: active` 이면서 `updated` 6개월 이상 정체된 페이지
- raw/ 에 더 새로운 소스가 있으면 업데이트 후 `active`

### Check 4. 누락 교차참조
- entity A 본문에 entity B가 **텍스트로 언급**되나 `[[B]]` wikilink 없음
- comparison 페이지가 언급한 개체 중 대응 entity/concept 페이지 부재
- **자동 수정 가능**: wikilink 추가 (판단 명확한 경우)

### Check 5. 데이터 갭 (Karpathy 핵심)
- `index_<domain>.md` 내 "TODO" / "오픈 질문" 섹션 항목 수집
- 언급되나 자체 페이지 없는 개념 목록
- 조치: **새 ingest 후보 또는 웹검색 후보로 사용자에게 보고**

### Check 6. 인덱스 정합성
- `index_<domain>.md` 등재 페이지 ↔ 실제 파일시스템 일치
- 신규 파일 중 index 누락 → index 보강
- index에 있으나 파일 없음 → index 정리

### Check 7. `_inbox/` 적체
- `wiki/<domain>/_inbox/` 파일 수 집계
- **> 10 이면 긴급 재분류 트리거**: 각 파일 타입 판정(entity/concept/source-summary) → 적절 경로로 이동

### Check 8. Frontmatter 유효성
`10_frontmatter.md` 기준:
- 필수 필드 누락: `id`, `title`, `domain`, `type`, `status`, `tags`, `sources`, `created`, `updated`
- `id` ≠ 파일명 (확장자 제외)
- 존재하지 않는 태그 네임스페이스 (prefix가 `project/`, `tech/`, `person/`, `company/`, `status/` 중 없음)
- `type` 값이 5종(entity/concept/source-summary/comparison/note) 외

## 수정 적용 정책

- **자동 적용** (보고만 하고 즉시 수정):
  - 누락 wikilink (판단 명확한 경우)
  - frontmatter 필드 누락 (`updated` 추가 등)
  - index 정합성 (신규 파일 등재, 없는 파일 제거)
- **사용자 승인 필요**:
  - 고아 페이지 삭제 / archived 강등
  - 모순 해결 (병합 방향)
  - `_inbox/` 대량 이동

## log_<domain>.md 기록 — **포맷 엄수**

```markdown
## [YYYY-MM-DD HH:MM] lint | <domain>
findings:
  orphans: N
  conflicts: N
  outdated: N
  missing_xrefs: N
  data_gaps: N
  index_mismatch: N
  inbox_backlog: N
  frontmatter_errors: N
actions:
  - auto-fixed: <요약>
  - pending_user: <요약>
suggestions:
  - <새 ingest 후보 1>
  - <후속 질문 1>
```

- 접두사 `## [YYYY-MM-DD HH:MM] lint | ` 변형 금지
- 8개 findings 카테고리는 **모두 기록** (0이어도)

## Karpathy principles to honor

1. *"orphan pages, missing cross-references, data gaps"* — 단순 검증이 아니라 다음 방향 제시
2. *"suggesting new questions and new sources"* — suggestions 섹션 빈 채로 두지 말 것
3. *"keeps the wiki healthy as it grows"* — lint는 파괴적이지 않게; 삭제는 항상 사용자 승인

## Hard rules (금지)

- ❌ 동시 2개 이상 도메인 lint (집중력 분산, 로그 복잡)
- ❌ 사용자 승인 없는 페이지 삭제
- ❌ 자동 수정을 요약 없이 일괄 적용 (변경 목록 항상 보고)
- ❌ log 접두사 포맷 변형
- ❌ 8개 체크 중 일부 생략

## Report template (사용자에게 제시)

```
# Lint Report — <domain> ([YYYY-MM-DD HH:MM])

## Summary
| Category         | Count | Auto-fixed | Pending |
|------------------|-------|------------|---------|
| Orphans          | N     | -          | N       |
| Conflicts        | N     | -          | N       |
| Outdated         | N     | N          | N       |
| Missing xrefs    | N     | N          | N       |
| Data gaps        | N     | -          | N       |
| Index mismatch   | N     | N          | N       |
| _inbox backlog   | N     | -          | N       |
| Frontmatter errs | N     | N          | N       |

## Auto-fixed
- ...

## Needs your decision
- [orphan] wiki/<domain>/<x>.md — archive / delete / link-to?
- [conflict] wiki/<domain>/<a>.md vs <b>.md — 병합 방향?

## Suggestions (next ingest / query)
- ...
```

## Verification checklist

- [ ] 대상 도메인 확정됨
- [ ] 8개 체크 모두 실행됨
- [ ] 자동 수정 사항 변경 목록으로 보고됨
- [ ] 사용자 결정 필요 항목 분리 제시됨
- [ ] `log_<domain>.md` 에 규정 포맷으로 append
- [ ] Suggestions 최소 1개 이상 제시
