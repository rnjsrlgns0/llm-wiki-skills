---
name: project-ingest
description: Use when the user adds project artifacts (PR summary, issue resolution, meeting notes, architecture decision, code review, deployment record) to the project wiki, or says "이거 프로젝트 위키에 넣어줘", "ADR 작성해줘", "인시던트 기록", "모듈 문서화", "project ingest". Executes the project wiki ingest workflow — determines page type, creates/updates pages, adds cross-references, and logs the event.
---

# Project Ingest Skill

> "결정의 맥락이 코드보다 오래 산다" — The context behind decisions outlives the code itself.

소프트웨어 프로젝트 아티팩트(PR 요약, 이슈 해결, 회의록, 아키텍처 결정, 코드 리뷰, 배포 기록)를 **project-wiki**에 구조화·영구 보존하는 워크플로우를 실행한다.

이 skill의 핵심 보장: 어떤 결정이 왜 내려졌는지, 어떤 사건이 어떻게 해결되었는지 — 그 맥락이 코드 변경 이후에도 추적 가능하도록 한다.

---

## When to Use

- 사용자가 PR 요약·이슈 본문·회의록·코드 설명을 주며 "프로젝트 위키에 넣어줘"
- "ADR 작성해줘", "아키텍처 결정 기록해줘"
- "인시던트 기록", "장애 사후 분석 정리"
- "모듈 문서화", "컴포넌트 설명 추가"
- "배포 기록", "릴리즈 노트 위키에"
- "용어 정의 추가", "glossary 업데이트"

---

## Page Types

6가지 페이지 타입이 있다. 각 타입은 고유한 저장 경로, 필수 섹션, 라이프사이클을 가진다.

### 1. `adr` — Architecture Decision Record

아키텍처·설계 결정과 그 근거를 영구 보존한다. 한 번 기록된 ADR은 삭제하지 않는다 — 번복되더라도 `superseded` 상태로 남긴다.

**저장 경로**: `project-wiki/<project>/adrs/<NNN>-<slug>.md`

- `NNN`: 3자리 0-패딩 일련번호 (기존 adrs/ 디렉토리를 스캔해 최대 번호 + 1)
- `<slug>`: 제목을 kebab-case로 변환 (예: `use-fastapi-over-flask`)

**예시**: `adrs/001-use-fastapi-over-flask.md`

**필수 섹션**:
```markdown
## Context
어떤 상황/문제에서 이 결정이 필요했는가. 당시 제약조건과 배경 포함.

## Decision
내린 결정을 명확하게 서술. "우리는 X를 선택했다" 형식.

## Alternatives Considered
검토했지만 선택하지 않은 대안들과 각 대안의 trade-off.

## Consequences
이 결정으로 인한 결과 — 긍정적·부정적·중립적 모두 포함.

## Status
현재 상태: `proposed` / `accepted` / `deprecated` / `superseded`
```

**Status 흐름**:
```
proposed → accepted → deprecated
                    ↘ superseded (새 ADR이 대체할 때)
```

`superseded` 상태의 ADR은 `related` 필드에 대체한 ADR의 id를 반드시 기재한다.

---

### 2. `module` — Module/Component Description

코드베이스의 특정 모듈·컴포넌트·서비스의 역할과 설계 의도를 기술한다.

**저장 경로**: `project-wiki/<project>/modules/<slug>.md`

**예시**: `modules/auth-middleware.md`

**필수 섹션**:
```markdown
## Purpose
이 모듈이 존재하는 이유. 어떤 문제를 해결하는가.

## Interface
공개 API / 진입점 / 주요 함수 시그니처. 내부 구현 상세는 생략.

## Dependencies
이 모듈이 의존하는 다른 모듈·외부 서비스·라이브러리.

## Design Rationale
현재 구조로 설계된 이유. 고려했던 다른 방식과 선택 근거.

## Related ADRs
이 모듈 설계에 영향을 준 ADR 목록. `[[adr-id]]` 형식으로 링크.
```

---

### 3. `glossary` — Project Glossary Term

프로젝트 고유 용어, 도메인 개념, 약어를 정의한다. 팀 내에서 다르게 쓰이던 용어를 단일 정의로 수렴시킨다.

**저장 경로**: `project-wiki/<project>/glossary/<slug>.md`

**예시**: `glossary/profiling-score.md`

**필수 섹션**:
```markdown
## Definition
이 용어의 공식 정의. 한 문장으로 시작, 필요 시 확장.

## Context
이 용어가 어느 맥락에서 쓰이는가. 코드베이스 어디에서 등장하는가.

## Related Modules
이 개념을 구현하거나 사용하는 모듈 목록.

## Caveats
혼동하기 쉬운 유사 용어, 잘못 쓰이는 패턴, 주의사항.
```

---

### 4. `runbook` — Operational Procedure

운영 절차를 단계별로 기술한다. 장애 상황이나 반복 작업에서 누구나 따를 수 있도록 구체적으로 작성한다.

**저장 경로**: `project-wiki/<project>/runbooks/<slug>.md`

**예시**: `runbooks/deploy-to-production.md`

**필수 섹션**:
```markdown
## Prerequisites
이 절차를 실행하기 전 갖춰야 할 조건·권한·환경.

## Step-by-step
번호가 매겨진 순서대로 실행할 명령·행동. 각 단계에 예상 출력 포함.

## Verification
절차가 성공적으로 완료되었는지 확인하는 방법.

## Rollback
문제 발생 시 이전 상태로 되돌리는 절차.

## Related Incidents
이 runbook과 연관된 인시던트 기록. 이 절차가 만들어진 계기가 된 사건.
```

---

### 5. `incident` — Incident Record

서비스 장애·데이터 문제·보안 사고 등의 사후 분석(post-mortem)을 기록한다.

**저장 경로**: `project-wiki/<project>/incidents/<YYYY-MM-DD>-<slug>.md`

- `<YYYY-MM-DD>`: 인시던트 발생일 (감지일 기준)

**예시**: `incidents/2026-04-15-db-pool-exhaustion.md`

**필수 섹션**:
```markdown
## Timeline
발생·감지·대응·복구 시각을 시간 순으로 정리. ISO 8601 형식 권장.

## Root Cause
근본 원인 분석. 직접 원인과 기여 요인을 구분.

## Impact
영향 범위 — 영향받은 사용자 수, 서비스 다운타임, 데이터 손실 여부.

## Resolution
문제를 해결하기 위해 취한 조치.

## Prevention
재발 방지를 위한 후속 조치 (각 항목에 담당자·마감일 포함 권장).

## Related ADRs · Modules
이 인시던트와 관련된 설계 결정, 영향받은 모듈.
```

---

### 6. `changelog` — Release/Milestone Change Summary

릴리즈·마일스톤·대규모 리팩토링 등 변경 사항을 요약한다.

**저장 경로**: `project-wiki/<project>/changelog/<slug>.md`

**예시**: `changelog/v2-api-migration.md`

**필수 섹션**:
```markdown
## Changes
이 릴리즈/마일스톤에 포함된 변경 사항 목록.

## Motivation
왜 이 변경이 필요했는가. 해결한 문제나 달성한 목표.

## Affected Modules
변경의 영향을 받은 모듈·컴포넌트 목록.

## Related ADRs · PRs
이 변경과 연관된 아키텍처 결정, PR 번호.
```

---

## Frontmatter Specification

모든 project-wiki 페이지는 아래 frontmatter를 포함해야 한다.

```yaml
---
id: <slug>
title: <title>
project: <project-name>
type: adr | module | glossary | runbook | incident | changelog
status: draft | active | deprecated | superseded
tags: [<namespace/slug>, ...]
refs:
  prs: [#123, #456]
  issues: [#789]
  commits: [abc1234]
related: [<wiki page id>, ...]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

**필드 규칙**:
- `id`: 파일명(확장자 제외)과 일치. ADR은 `NNN-<slug>` 형식.
- `status`:
  - `draft` — 작성 중, 아직 팀 합의 전
  - `active` — 현재 유효
  - `deprecated` — 더 이상 유효하지 않지만 대체 없음
  - `superseded` — 다른 페이지로 대체됨 (`related`에 대체 페이지 id 기재)
- `tags`: `<namespace>/<slug>` 형식 권장 (예: `backend/auth`, `infra/db`)
- `refs`: 소스 아티팩트 참조. 근거 없는 사실 기재 금지.
- `related`: 양방향 교차참조. A가 B를 related로 가리키면 B도 A를 related로 가져야 한다.

---

## Directory Structure

```
project-wiki/<project-name>/
├── index.md          ← 프로젝트 카탈로그 (카테고리별 페이지 목록)
├── log.md            ← append-only 이벤트 로그
├── adrs/             ← Architecture Decision Records
├── modules/          ← 모듈/컴포넌트 문서
├── glossary/         ← 프로젝트 용어집
├── runbooks/         ← 운영 절차
├── incidents/        ← 인시던트 기록
├── changelog/        ← 릴리즈·마일스톤 변경 기록
└── _inbox/           ← 타입 미분류 임시 보관
```

`_inbox/`에 들어간 페이지는 `type: note`로 frontmatter를 작성하고, 다음 `project-lint` 실행 시 재분류 대상이 된다.

---

## Ingest Procedure

### Step 1. 소스 타입 판별

아래 우선순위 순서로 페이지 타입을 결정한다.

| 우선순위 | 조건 | 타입 |
|----------|------|------|
| 1 | 사용자가 명시적으로 타입을 지정 | 지정된 타입 사용 |
| 2 | 소스가 PR / 릴리즈 노트 | `changelog` |
| 3 | 소스에 결정·트레이드오프 내용 포함 | `adr` |
| 4 | 소스가 시스템 장애·사고를 서술 | `incident` |
| 5 | 소스가 운영 절차·how-to를 서술 | `runbook` |
| 6 | 소스가 용어·개념을 정의 | `glossary` |
| 7 | 소스가 코드 컴포넌트를 설명 | `module` |
| 8 | 위 어디에도 해당 없음 | `_inbox/` (`type: note`) |

하나의 소스가 여러 타입에 해당할 수 있다. 예: PR에 아키텍처 결정이 포함된 경우 → `changelog` + `adr` 두 페이지 모두 생성.

### Step 2. 프로젝트 식별

`project-wiki/` 하위의 어느 프로젝트 디렉토리를 대상으로 할지 결정한다.

- 소스에서 프로젝트명을 명확히 식별할 수 있으면 → 해당 디렉토리 사용
- 모호하거나 여러 프로젝트에 걸치면 → **반드시 사용자에게 확인** (추측 금지)

### Step 3. 프로젝트 위키 존재 여부 확인

`project-wiki/<project>/index.md` 존재 여부를 확인한다.

- **존재하지 않으면** → 사용자에게 다음을 안내하고 중단:
  ```
  '<project>' 프로젝트 위키가 아직 초기화되지 않았습니다.
  먼저 `project-setup` 명령어를 실행해 주세요.
  ```
- **존재하면** → 다음 단계 진행

### Step 4. 소스 페이지 생성

결정된 타입과 경로 규칙에 따라 새 페이지를 생성한다.

- **ADR**: `adrs/` 디렉토리를 스캔해 기존 최대 NNN을 찾아 +1로 번호 부여. `adrs/` 가 비어있으면 `001`부터 시작.
- **incident**: 파일명 앞에 발생일 `YYYY-MM-DD-` 접두사 추가.
- **기존 페이지와 slug 충돌 시**: 사용자에게 기존 페이지 업데이트 여부 확인.

frontmatter의 모든 필수 필드를 채운다. 출처를 알 수 없는 필드는 빈 값(`""` 또는 `[]`)으로 남긴다 — 추측 금지.

### Step 5. 영향받는 기존 페이지 식별 및 업데이트

새 소스가 언급하거나 변경하는 기존 wiki 페이지를 찾는다.

- 인시던트 → 관련 `module`, `runbook` 페이지 업데이트
- ADR → 영향받는 `module` 페이지의 "Related ADRs" 섹션 업데이트
- changelog → 영향받는 `module` 페이지의 업데이트 내역 반영
- 새 소스가 기존 페이지 내용과 **모순**되면 → 모순을 명시적으로 기록. 조용히 덮어쓰지 않는다.

### Step 6. 양방향 교차참조 추가

모든 관계는 양방향으로 유지되어야 한다.

- 새 페이지의 `related` 필드에 연관 페이지 id 추가
- 연관된 기존 페이지의 `related` 필드에 새 페이지 id 추가
- 본문 내 `[[wikilink]]` 형식으로 인라인 링크 삽입 (해당되는 위치에)

**교차 도메인 주의**: 새 아티팩트가 `wiki-private` 도메인 페이지와 관련되면 — 관련성을 언급하되, `wiki-private` 페이지는 수정하지 않는다. 해당 관계를 새 페이지의 `related` 또는 본문 노트에 기록하는 것으로 그친다.

### Step 7. index.md 업데이트

`project-wiki/<project>/index.md`의 해당 카테고리 섹션에 새 페이지 항목을 추가한다.

```markdown
| [페이지 제목](상대경로) | 한 줄 요약 | YYYY-MM-DD |
```

### Step 8. log.md 추가

`project-wiki/<project>/log.md` 파일에 아래 형식으로 **append**한다. 기존 항목은 절대 수정하지 않는다.

```markdown
## [YYYY-MM-DD HH:MM] ingest | <title>

- type: <page-type>
- path: <relative-path-from-project-wiki>
- refs: <PR#/issue#/commit 등 소스 참조>
- related: <연관 페이지 id 목록>
```

log.md 접두사 형식 `## [YYYY-MM-DD HH:MM] ingest | ` — 이 형식을 절대 변경하지 않는다.

---

## Hard Rules

1. **프로젝트명 추측 금지**: 소스에서 프로젝트가 명확하지 않으면 반드시 사용자에게 확인한다.

2. **ADR 번호는 순차적**: `adrs/` 디렉토리를 실제로 스캔해 최대 NNN을 확인하고 +1 부여. 임의 번호 사용 금지.

3. **모든 사실에 출처 첨부**: 페이지 본문에 기재하는 사실·결정·수치는 반드시 소스(PR#, commit hash, issue#, 회의 날짜)를 참조해야 한다. 출처 없는 사실 기재 금지.

4. **log.md append-only**: log.md의 기존 항목을 수정하거나 삭제하지 않는다. 오직 추가만 허용.

5. **log 접두사 형식 불변**: `## [YYYY-MM-DD HH:MM] ingest | ` 형식을 변경하지 않는다.

6. **교차 도메인 불변**: `wiki-private` 등 다른 플러그인 도메인 페이지는 언급만 하고 직접 수정하지 않는다.

7. **양방향 참조 완결**: 단방향 링크 금지. A → B이면 B → A도 반드시 추가한다.

8. **모순 은폐 금지**: 새 소스가 기존 페이지와 충돌하면 충돌을 명시적으로 기록한다. 조용히 덮어쓰지 않는다.

---

## Verification Checklist

ingest 완료 전 아래 항목을 모두 확인한다.

- [ ] 페이지 타입이 타입 결정 우선순위 규칙에 따라 올바르게 판별됨
- [ ] frontmatter 모든 필수 필드 완성 (`id`, `title`, `project`, `type`, `status`, `created`, `updated`)
- [ ] `refs` 필드에 소스 참조(PR#, issue#, commit hash) 포함
- [ ] 양방향 교차참조 추가됨 (새 페이지 `related` + 기존 페이지 `related`)
- [ ] `index.md` 해당 카테고리에 새 항목 추가됨
- [ ] `log.md`에 올바른 형식으로 append됨
- [ ] 기존 페이지와 내용 충돌 시 모순이 명시적으로 기록됨
- [ ] ADR인 경우: 번호가 기존 최대값 + 1로 순차 부여됨
- [ ] incident인 경우: 파일명에 발생일 `YYYY-MM-DD-` 접두사 포함됨
