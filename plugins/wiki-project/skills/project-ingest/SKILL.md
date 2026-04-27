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

8가지 페이지 타입이 있다. 각 타입은 고유한 저장 경로, 필수 섹션, 라이프사이클을 가진다.

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

### 7. `entity` — Project-scoped Entity

프로젝트에서 실제로 등장하는 도구, 프레임워크, 팀, 사람, 회사 등 고유명사 개체를 기술한다. 소스 전반에서 반복 참조되는 고유 개체에 단일 진실 페이지를 제공한다.

**저장 경로**: `project-wiki/<project>/entities/<slug>.md`

**예시**: `entities/fastapi.md`, `entities/platform-team.md`

**필수 섹션**:
```markdown
## Definition
이 개체가 무엇인지 한 문장으로 정의. (예: "FastAPI는 Python 기반의 비동기 웹 프레임워크다.")

## Key Facts
이 개체에 대한 핵심 사실 목록 (버전, 역할, 책임, 주요 특징 등).

## Related Modules
이 개체를 사용하거나 직접 연관된 모듈 목록. `[[module-slug]]` 형식.

## Related ADRs
이 개체의 도입·변경·제거와 관련된 ADR 목록. `[[adr-id]]` 형식.

## Timeline
이 개체의 프로젝트 내 주요 이력 (도입일, 버전 변경, 역할 변화 등).
```

---

### 8. `concept` — Project-scoped Concept

프로젝트에서 적용하거나 논의되는 추상적 패턴, 기법, 아키텍처 스타일을 기술한다. 코드베이스 전반에 영향을 미치는 설계 사고를 단일 페이지로 집약한다.

**저장 경로**: `project-wiki/<project>/concepts/<slug>.md`

**예시**: `concepts/circuit-breaker-pattern.md`, `concepts/event-sourcing.md`

**필수 섹션**:
```markdown
## Definition
이 개념이 무엇인지 한 문장으로 정의.

## Core Principles
이 개념을 구성하는 핵심 원칙·전제 조건 목록.

## Variants
이 개념의 변형 또는 세부 구현 방식 (있는 경우).

## Contrast Concepts
혼동하기 쉬운 유사 개념과의 차이점.

## References
이 개념이 적용된 ADR, 모듈, 외부 문헌 링크.
```

---

## Frontmatter Specification

모든 project-wiki 페이지는 아래 frontmatter를 포함해야 한다.

```yaml
---
id: <slug>
title: <title>
project: <project-name>
type: adr | module | glossary | runbook | incident | changelog | entity | concept
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
├── entities/         ← 프로젝트 고유 개체 (도구·팀·사람·회사)
├── concepts/         ← 추상 개념·패턴·기법
├── _raw/             ← 원본 소스 보존 (불변)
│   ├── prs/          ← PR 원문 (PR-<number>.md)
│   ├── issues/       ← 이슈 원문 (ISSUE-<number>.md)
│   ├── meetings/     ← 회의록 원문 (<YYYY-MM-DD>-<slug>.md)
│   └── docs/         ← 기타 문서 원문 (<slug>.md)
└── _inbox/           ← 타입 미분류 임시 보관
```

`_inbox/`에 들어간 페이지는 `type: note`로 frontmatter를 작성하고, 다음 `project-lint` 실행 시 재분류 대상이 된다.

`_raw/`에 저장된 파일은 **절대 수정하지 않는다** — 원본이 변경되면 새 파일로 추가한다.

---

## Ingest Procedure

### Step 1. 소스 읽기 및 타입 판별

아래 우선순위 순서로 페이지 타입을 결정한다.

| 우선순위 | 조건 | 타입 |
|----------|------|------|
| 1 | 사용자가 명시적으로 타입을 지정 | 지정된 타입 사용 |
| 2 | 소스가 PR / 릴리즈 노트 | `changelog` |
| 3 | 소스에 결정·트레이드오프 내용 포함 | `adr` |
| 4 | 소스가 시스템 장애·사고를 서술 | `incident` |
| 5 | 소스가 운영 절차·how-to를 서술 | `runbook` |
| 6 | 소스가 특정 도구·프레임워크·팀·사람을 언급 | `entity` |
| 7 | 소스가 추상적 패턴·기법을 설명 | `concept` |
| 8 | 소스가 용어·개념을 정의 | `glossary` |
| 9 | 소스가 코드 컴포넌트를 설명 | `module` |
| 10 | 위 어디에도 해당 없음 | `_inbox/` (`type: note`) |

하나의 소스가 여러 타입에 해당할 수 있다. 예: PR에 아키텍처 결정이 포함된 경우 → `changelog` + `adr` 두 페이지 모두 생성.

### Step 1-b. 애매한 해석 확인

소스 해석이 모호한 경우 (PR 의도 불명확, 결정 근거 불충분, 인시던트 원인 불분명 등), 다음 단계로 진행하기 전에 **사용자에게 1-2개의 명확화 질문**을 한다.

- 질문 예시:
  - "이 PR의 핵심 결정 의도가 X인가요, 아니면 Y인가요?"
  - "인시던트 근본 원인이 A로 보이는데, 확인이 필요한 부분이 있나요?"
- 소스가 자명한 경우 이 단계를 건너뛰고 바로 진행한다.
- 추측으로 계속 진행하지 않는다 — 불확실한 해석은 반드시 확인 후 처리한다.

### Step 2. 원본 보존 (_raw/)

소스 아티팩트를 `_raw/` 디렉토리에 먼저 기록한다. 이 단계는 구조화 작업보다 선행한다.

| 소스 종류 | 저장 경로 |
|-----------|-----------|
| PR 텍스트 | `_raw/prs/PR-<number>.md` |
| 이슈 텍스트 | `_raw/issues/ISSUE-<number>.md` |
| 회의록 | `_raw/meetings/<YYYY-MM-DD>-<slug>.md` |
| 기타 문서 | `_raw/docs/<slug>.md` |

**_raw/ 파일 형식**:
```markdown
---
title: <원본 제목>
date: YYYY-MM-DD
source_url: <원본 URL 또는 참조>
---

<!-- 원본 전문 또는 메타데이터만 포함 (전문이 없을 경우 title/date/source_url로 충분) -->
```

**_raw/ 불변 규칙**: 한 번 기록된 `_raw/` 파일은 절대 수정하지 않는다. 원본이 업데이트된 경우 새 파일로 추가한다.

### Step 3. 프로젝트 식별

`project-wiki/` 하위의 어느 프로젝트 디렉토리를 대상으로 할지 결정한다.

- 소스에서 프로젝트명을 명확히 식별할 수 있으면 → 해당 디렉토리 사용
- 모호하거나 여러 프로젝트에 걸치면 → **반드시 사용자에게 확인** (추측 금지)

### Step 4. 프로젝트 위키 존재 여부 확인

`project-wiki/<project>/index.md` 존재 여부를 확인한다.

- **존재하지 않으면** → 사용자에게 다음을 안내하고 중단:
  ```
  '<project>' 프로젝트 위키가 아직 초기화되지 않았습니다.
  먼저 `project-setup` 명령어를 실행해 주세요.
  ```
- **존재하면** → 다음 단계 진행

### Step 5. 페이지 생성 (주 페이지 + entity + concept + 교차참조)

**이 Step은 4개 하위 단계로 구성되며, 5-a만으로는 미완성이다. 5-a~5-d 전체를 완료해야 1건의 페이지 생성이 완료된다.**

#### 5-a. 주 페이지 생성

결정된 타입과 경로 규칙에 따라 새 페이지를 생성한다.

- **ADR**: `adrs/` 디렉토리를 스캔해 기존 최대 NNN을 찾아 +1로 번호 부여. `adrs/` 가 비어있으면 `001`부터 시작.
- **incident**: 파일명 앞에 발생일 `YYYY-MM-DD-` 접두사 추가.
- **기존 페이지와 slug 충돌 시**: 사용자에게 기존 페이지 업데이트 여부 확인.

frontmatter의 모든 필수 필드를 채운다. 출처를 알 수 없는 필드는 빈 값(`""` 또는 `[]`)으로 남긴다 — 추측 금지.

#### 5-b. entity 추출 및 페이지 생성/업데이트

주 페이지 본문에서 **모든 고유명사**(도구, 프레임워크, 라이브러리, 팀, 인물, 회사)를 스캔한다.

**entity 추출 대상**:
- 도구 및 프레임워크 (예: FastAPI, Redis, Kafka)
- 팀 및 조직 단위 (예: Platform Team, Data Engineering)
- 사람 (예: 시스템 오너, 의사결정자)
- 외부 회사·서비스 (예: AWS, Stripe)

각 고유명사에 대해:
- `entities/` 에 기존 페이지 있으면 → 관련 섹션(Related Modules, Related ADRs, Timeline) 업데이트 + `updated` 갱신
- 없으면 → 새 entity 페이지 생성

이 단계를 생략하면 주 페이지 생성도 미완성으로 간주한다.

#### 5-c. concept 추출 및 페이지 생성/업데이트

주 페이지 본문에서 **모든 추상 개념**(패턴, 기법, 아키텍처 스타일, 원칙)을 스캔한다.

**concept 추출 대상**:
- 설계 패턴 (예: Circuit Breaker, CQRS)
- 기법·원칙 (예: Event Sourcing, Idempotency)
- 아키텍처 스타일 (예: Hexagonal Architecture)

각 추상 개념에 대해:
- `concepts/` 에 기존 페이지 있으면 → 관련 섹션(References) 업데이트 + `updated` 갱신
- 없으면 → 새 concept 페이지 생성

이 단계를 생략하면 주 페이지 생성도 미완성으로 간주한다.

#### 5-d. 양방향 교차참조

5-a~c에서 생성/업데이트한 모든 페이지 간 양방향 링크를 추가한다.

- 주 페이지 → entity/concept 페이지: `related` 필드 + 본문 `[[wikilink]]`
- entity/concept → 주 페이지: `related` 필드 + 본문 역링크
- entity ↔ concept 간: 관련 있으면 상호 링크

**교차 도메인 주의**: 새 아티팩트가 `wiki-private` 도메인 페이지와 관련되면 — 관련성을 언급하되, `wiki-private` 페이지는 수정하지 않는다. 해당 관계를 새 페이지의 `related` 또는 본문 노트에 기록하는 것으로 그친다.

**5-a만 완료하고 5-b~d를 건너뛰는 것은 절차 위반이다.**

> Karpathy 원칙: **"하나의 소스는 10-15개 페이지를 수정할 수 있다"** — 5-a만으로 끝나면 이 원칙을 위반한 것이다.

### Step 6. 영향받는 기존 페이지 식별 및 업데이트

새 소스가 언급하거나 변경하는 기존 wiki 페이지를 찾는다.

- 인시던트 → 관련 `module`, `runbook` 페이지 업데이트
- ADR → 영향받는 `module` 페이지의 "Related ADRs" 섹션 업데이트
- changelog → 영향받는 `module` 페이지의 업데이트 내역 반영
- 새 소스가 기존 페이지 내용과 **모순**되면 → 모순을 명시적으로 기록. 조용히 덮어쓰지 않는다.

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

9. **_raw/ 원본 수정 금지 (불변)**: `_raw/` 디렉토리에 한 번 기록된 파일은 절대 수정하지 않는다. 원본 업데이트 시 새 파일로 추가한다.

10. **Step 5 전체 완료 의무**: Step 5는 5-a(주 페이지) + 5-b(entity) + 5-c(concept) + 5-d(교차참조)의 4개 하위 단계로 구성된다. Step 5-a만 완료하고 5-b~d를 생략하는 것은 ingest 미완료이다. "시간이 없어서" 또는 "주 페이지만 만들면 충분해서" 5-b~d를 건너뛰지 않는다.

---

## Verification Checklist

ingest 완료 전 아래 항목을 모두 확인한다.

- [ ] 페이지 타입이 타입 결정 우선순위 규칙에 따라 올바르게 판별됨
- [ ] 애매한 해석이 있었던 경우 사용자 확인 후 진행됨 (Step 1-b)
- [ ] `_raw/` 에 원본 소스 보존됨 (Step 2)
- [ ] frontmatter 모든 필수 필드 완성 (`id`, `title`, `project`, `type`, `status`, `created`, `updated`)
- [ ] `refs` 필드에 소스 참조(PR#, issue#, commit hash) 포함
- [ ] Step 5 전체 완료: 5-a 주 페이지 생성 + 5-b entity 추출·페이지 생성/업데이트 + 5-c concept 추출·페이지 생성/업데이트 + 5-d 양방향 교차참조 — 4개 하위 단계 모두 완료되어야 Step 5 완료로 인정
- [ ] `index.md` 해당 카테고리에 새 항목 추가됨
- [ ] `log.md`에 올바른 형식으로 append됨
- [ ] 기존 페이지와 내용 충돌 시 모순이 명시적으로 기록됨
- [ ] ADR인 경우: 번호가 기존 최대값 + 1로 순차 부여됨
- [ ] incident인 경우: 파일명에 발생일 `YYYY-MM-DD-` 접두사 포함됨
