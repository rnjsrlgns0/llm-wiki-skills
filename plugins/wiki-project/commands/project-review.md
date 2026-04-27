---
description: 프로젝트 위키 전체 점검을 수행합니다. "프로젝트 리뷰", "project review", "위키 점검", "프로젝트 건강검진" 등의 요청 시 사용합니다.
---

# Project Review 워크플로우

**프로젝트 위키의 건강 상태를 종합 점검합니다 (lint + sync).**

---

## Phase A: 대상 확인

### A-1. 프로젝트 선택

```
Use Glob tool:
- pattern: "project-wiki/*/index.md"
```

사용 가능한 프로젝트 위키 목록을 표시합니다.
사용자가 미지정 시 가장 최근 활동(log.md 마지막 항목 기준)이 오래된 프로젝트를 후보로 제안합니다.

### A-2. 현황 스캔

```
Use Glob tool:
- pattern: "project-wiki/<project>/**/*.md"
```

타입별 페이지 수 요약:
| Type | Count |
|------|-------|
| adr | N |
| module | N |
| glossary | N |
| runbook | N |
| incident | N |
| changelog | N |
| _inbox | N |

---

## Phase B: Sync 실행

코드 변경이 위키에 미치는 영향을 먼저 확인합니다.

```
Use Skill tool:
- skill: "wiki-project:project-sync"
- args: "<project>"
```

sync 결과 outdated 마킹된 페이지를 확인합니다.

---

## Phase C: Lint 실행

```
Use Skill tool:
- skill: "wiki-project:project-lint"
- args: "<project>"
```

8개 체크리스트 전체 실행:
1. 고아 페이지
2. ADR-코드 모순
3. 모듈 문서 최신성
4. 누락 교차참조
5. Superseded ADR 체인
6. 인덱스 정합성
7. _inbox 적체
8. Frontmatter 유효성

---

## Phase D: 종합 보고

Sync + Lint 결과를 통합하여 보고합니다.

```
# Project Review — <project> ([YYYY-MM-DD])

## Sync Results
| 항목 | 수 |
|------|---|
| 변경된 소스 파일 | N |
| 영향받는 위키 페이지 | N |
| Outdated 마킹 | N |
| 미문서화 파일 | N |

## Lint Results
| Category           | Count | Auto-fixed | Pending |
|--------------------|-------|------------|---------|
| ...                |       |            |         |

## Priority Actions
1. [가장 긴급한 항목]
2. [...]

## Suggestions
- [다음 ingest 후보]
- [문서화 필요 모듈]
```

---

## Phase E: 조치 실행

### E-1. 자동 수정

lint + sync 자동 수정 항목 적용 (변경 목록 보고).

### E-2. 사용자 결정

수동 결정 필요 항목을 목록으로 제시:
- ADR 상태 변경
- 고아 페이지 처리
- 모순 해결 방향

---

## 주의사항

- 한 번에 한 프로젝트만 점검
- Phase B(sync) → C(lint) 순서 고정 (sync가 outdated 마킹 → lint가 이를 포함하여 검사)
- 사용자 승인 없는 페이지 삭제 금지
