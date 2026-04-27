---
name: project-query
description: Use when the user asks questions about project context, architecture decisions, module design, or says "이 모듈 왜 이렇게?", "인증 어떻게 동작?", "project query", "프로젝트 위키에서 찾아줘", "ADR 뭐 있어?", "최근 변경 이력". Reads project wiki index, traverses cross-references, synthesizes a cited answer, and promotes valuable answers back into the wiki.
---

# project-query Skill

> 핵심 원칙: "결정의 맥락이 코드보다 오래 산다"

프로젝트 위키에서 질문에 대한 컨텍스트를 탐색하고, 인용 근거가 있는 답변을 합성하며, 재사용 가능한 답변은 위키로 승격한다.

---

## 1. Page Types (8 types)

| Type | Description | Directory |
|------|-------------|-----------|
| `adr` | Architecture Decision Record — 기술 결정의 배경·옵션·결과 | `adrs/` |
| `module` | Module/component description — 역할·인터페이스·의존성·소유자 | `modules/` |
| `glossary` | Project glossary term — 프로젝트 고유 용어 정의 | `glossary/` |
| `runbook` | Operational procedure — 배포·장애 대응·유지보수 절차 | `runbooks/` |
| `incident` | Incident record — 장애 경위·원인·후속 조치 | `incidents/` |
| `changelog` | Release change summary — 버전별 변경 사항 요약 | `changelog/` |
| `entity` | Project-scoped entities: tools, frameworks, teams, people | `entities/` |
| `concept` | Project-scoped concepts: patterns, techniques, architectural styles | `concepts/` |

---

## 2. Frontmatter Specification

모든 위키 페이지는 아래 frontmatter를 가진다. 질의 중 페이지를 파싱할 때 이 스펙을 기준으로 필드를 읽는다.

```yaml
---
id: <slug>                          # 고유 식별자 (파일명 기반 kebab-case)
title: <제목>                        # 사람이 읽을 수 있는 제목
project: <project-name>             # 소속 프로젝트명 (project-wiki/ 하위 디렉토리명과 일치)
type: adr | module | glossary | runbook | incident | changelog | entity | concept
status: draft | active | deprecated | superseded
tags: [<namespace/slug>, ...]       # 네임스페이스 포함 태그 (예: auth/jwt, infra/db)
refs:
  prs: [#123]                       # 관련 Pull Request 번호
  issues: [#789]                    # 관련 Issue 번호
  commits: [abc1234]                # 관련 커밋 해시
related: [<wiki page id>, ...]      # 교차 참조할 위키 페이지 id 목록
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

### Frontmatter 필드 규칙

| 필드 | 필수 | 규칙 |
|------|------|------|
| `id` | 필수 | 파일명(확장자 제외)과 동일. 변경 불가 |
| `title` | 필수 | 한글/영문 모두 허용 |
| `project` | 필수 | `project-wiki/<project>/` 디렉토리명과 정확히 일치 |
| `type` | 필수 | 위 8종 중 하나 |
| `status` | 필수 | `draft` → `active` → `deprecated`/`superseded` |
| `tags` | 권장 | `namespace/slug` 형식. 최소 1개 권장 |
| `refs` | 선택 | 없으면 빈 객체 `{}` |
| `related` | 선택 | 없으면 빈 배열 `[]` |
| `created` | 필수 | 최초 생성일. 변경 불가 |
| `updated` | 필수 | 매 수정 시 갱신 |

---

## 3. Directory Structure

```
project-wiki/<project-name>/
├── index.md          # 프로젝트 위키 진입점. 모든 질의는 여기서 시작
├── log.md            # 모든 skill 이벤트 로그 (ingest/query/lint/sync)
├── adrs/             # Architecture Decision Records
├── modules/          # 모듈/컴포넌트 페이지
├── glossary/         # 용어집
├── runbooks/         # 운영 절차서
├── incidents/        # 인시던트 기록
├── changelog/        # 릴리즈 변경 기록
├── entities/          ← Project-scoped entities
├── concepts/          ← Project-scoped concepts
├── _raw/              ← Immutable source archive
└── _inbox/           # 미처리 원문 보관 (ingest 대기)
```

### index.md 구조

index.md는 위키의 진입점이며 다음 섹션을 포함한다:

```markdown
# <project-name> Wiki

## Overview
<프로젝트 한 줄 설명>

## Key Modules
- [[module-id]] — <한 줄 설명>

## Active ADRs
- [[adr-id]] — <결정 제목>

## Recent Changes
- [[changelog-id]] — <버전/날짜>

## Runbooks
- [[runbook-id]] — <절차 제목>
```

---

## 4. Query Procedure

### Step 1 — Project Identification

질문에서 대상 프로젝트를 파악한다.

```
판단 순서:
1. 사용자가 프로젝트명을 명시했으면 → project-wiki/<명시된 프로젝트>/ 사용
2. 현재 디렉토리 또는 대화 컨텍스트에서 프로젝트를 유추할 수 있으면 → 유추한 프로젝트 사용
3. project-wiki/ 하위에 디렉토리가 1개면 → 해당 프로젝트 사용
4. 여러 프로젝트가 존재하고 특정 불가면 → 사용자에게 질문
```

project-wiki/ 디렉토리 자체가 존재하지 않으면 즉시 중단하고 아래 메시지를 출력한다:

```
project-wiki/ 디렉토리를 찾을 수 없습니다.
project-setup 커맨드를 먼저 실행하여 프로젝트 위키를 초기화하세요.
```

### Step 2 — Read index.md First

**항상 index.md를 먼저 읽는다. 전체 파일을 grep하지 않는다.**

```
읽기 순서:
1. project-wiki/<project>/index.md 전체 읽기
2. 질문 키워드와 관련된 섹션 식별
3. 관련 페이지 id 목록 수집 (wikilink [[...]] 파싱)
```

index.md에 없는 키워드라도 index.md를 통해 발견한 페이지들의 `related` 필드를 따라 탐색한다.

### Step 3 — Collect Related Pages

index.md에서 수집한 페이지 id들을 출발점으로 교차 참조를 순회한다.

```
탐색 규칙:
1. 각 페이지의 frontmatter `related` 필드 → 연결된 페이지 id 수집
2. 본문의 [[wikilink]] → 연결된 페이지 id 수집
3. ADR → module → entity → concept 체인 우선 순회
   - ADR에서 참조된 entity/concept 페이지를 따라간다
   - module 페이지에서 참조된 entity(tools used) 페이지를 따라간다
   - entity 페이지에서 참조된 concept 페이지를 따라간다
   - 탐색 체인: ADR → module → entity → concept (최대 4단계 깊이)
4. 탐색 깊이: 최대 3홉 (index → 1차 페이지 → 2차 페이지 → 3차 페이지)
5. 이미 읽은 페이지는 재방문하지 않음 (순환 참조 방지)
```

질문 유형별 우선 탐색 방향:

| 질문 유형 | 우선 탐색 경로 |
|-----------|--------------|
| "왜 이렇게 설계했어?" | `adrs/` → `related` module → `related` entity/concept |
| "이 모듈 어떻게 동작해?" | `modules/` → `related` adr, runbook → `related` entity |
| "최근 변경 이력" | `changelog/` → `related` adr |
| "이 용어 무슨 뜻?" | `glossary/` |
| "장애 대응 어떻게 해?" | `runbooks/` → `related` incident |
| "ADR 뭐 있어?" | `index.md` Active ADRs 섹션 → `adrs/` |

### Step 4 — Synthesize Answer

수집한 페이지들을 바탕으로 답변을 합성한다.

**인용 규칙**:
- 모든 사실 주장에는 반드시 출처를 명시한다
- 위키 페이지 인용: `[[page-id]]`
- 외부 참조 인용: `PR #123`, `commit abc1234`, `Issue #456`
- 출처 없는 사실 주장은 금지

**답변 포맷 선택**:

| 상황 | 포맷 |
|------|------|
| 기본 질문 | 마크다운 서술형 |
| 옵션 비교 질문 ("A vs B") | 비교 테이블 |
| 변경 이력 질문 | 타임라인 목록 |
| 의존성/연관 질문 | 텍스트 의존성 다이어그램 |
| 코드 관련 질문 | 위키 + 실제 소스 파일 교차 검증 후 코드 스니펫 |

**코드 관련 질문 처리**:
위키 페이지가 특정 코드 구현을 설명하면, 실제 소스 파일을 읽어 위키 내용의 최신성을 검증한다. 위키 내용과 코드가 다르면 Step 5에서 모순으로 기록한다.

### Step 5 — Discover Contradictions and Gaps

답변 합성 과정에서 발견한 문제를 기록한다.

**모순 (Contradiction)**:
- ADR이 결정 X를 기술하지만 실제 코드는 Y를 구현하고 있음
- 두 위키 페이지가 서로 상충하는 정보를 기술하고 있음
- `status: active`이지만 `superseded`된 ADR을 참조하고 있음
- Entity 페이지의 도구/프레임워크 설명이 현재 코드 상태와 불일치함
- Concept 페이지의 패턴/기법 설명이 실제 구현 방식과 다름
- `_raw/` 원본 소스가 존재하는 경우, `_raw/` 원문과 위키 페이지 요약 간 내용 불일치

**공백 (Gap)**:
- index.md가 참조하는 모듈 페이지가 존재하지 않음
- 핵심 모듈에 module 페이지가 없음
- 코드에서 사용하는 용어가 glossary에 정의되어 있지 않음
- `related`가 가리키는 페이지가 삭제되어 존재하지 않음

발견된 모순/공백은 log.md에 `lint-candidate`로 기록한다 (Step 7 참고).

### Step 6 — Promotion Judgment

**Karpathy 원칙**: "탐색은 축적된다" — 가치 있는 답변을 채팅 히스토리에 버리지 않는다.

**승격 기준 (하나라도 해당하면 승격)**:

| 기준 | 설명 |
|------|------|
| 재사용 가능한 비교 분석 | 두 개 이상의 모듈/ADR을 비교한 새로운 분석 |
| 새로운 교차 분석 | 여러 페이지를 종합한 새로운 인사이트 |
| 사용자 명시 요청 | 사용자가 "저장해줘", "위키에 추가해" 요청 |

**승격 불필요 기준 (모두 해당하면 승격 안 함)**:

| 기준 | 설명 |
|------|------|
| 단일 페이지 재인용 | 하나의 페이지 내용을 그대로 요약한 경우 |
| 단순 조회 | "이 ADR 제목이 뭐야?" 수준의 trivial 질의 |

**승격 실행**:
- 적합한 type 디렉토리에 새 페이지 생성
- frontmatter에 `promoted_from: query` 추가
- index.md 해당 섹션에 링크 추가
- `updated` 날짜 갱신

```yaml
# 승격된 페이지의 frontmatter 예시
---
id: auth-comparison-jwt-vs-session
title: 인증 방식 비교 — JWT vs Session
project: <project-name>
type: adr
status: active
tags: [auth/comparison]
refs: {}
related: [adr-001-jwt-adoption, module-auth-service]
created: YYYY-MM-DD
updated: YYYY-MM-DD
promoted_from: query
---
```

### Step 7 — Log Event

질의 완료 후 반드시 `project-wiki/<project>/log.md`에 이벤트를 기록한다.

**로그 포맷**:

```markdown
## [YYYY-MM-DD HH:MM] query | <질문 요약 (50자 이내)>

- read: [[page-id-1]], [[page-id-2]], [[page-id-3]]
- promoted: <승격된 페이지 경로> (없으면 생략)
- lint-candidate: <모순/공백 설명> (없으면 생략)
```

**로그 규칙**:
- `## [YYYY-MM-DD HH:MM]` 접두사 형식은 절대 변경하지 않는다
- 새 로그는 기존 로그 위(파일 상단)에 추가한다
- `read:` 항목은 실제로 읽은 페이지만 기록한다 (탐색 중 스킵한 페이지 제외)
- `lint-candidate:` 항목은 Step 5에서 발견한 모순/공백을 간결하게 기술한다

---

## 5. Answer Template

모든 답변은 아래 템플릿을 기반으로 작성한다.

```markdown
<답변 본문 — 모든 사실에 [[링크]] 또는 외부 ref 인용>

---
**출처**: [[page-id-1]], [[page-id-2]], PR #123
**연관 후속 질문**:
- <이 답변에서 자연스럽게 이어지는 질문 1>
- <이 답변에서 자연스럽게 이어지는 질문 2>
```

**답변 본문 작성 지침**:
- 인용 없는 문장은 작성하지 않는다
- 여러 페이지의 정보를 종합할 때는 각 주장 끝에 `[[출처]]`를 인라인으로 표시한다
- 코드 스니펫을 포함할 경우, 실제 소스 파일을 확인한 후 작성한다
- 답변 길이는 질문의 복잡도에 비례하게 조절한다 (단순 조회: 3-5줄 / 설계 분석: 필요한 만큼)

---

## 6. Hard Rules

아래 규칙은 어떠한 상황에서도 예외 없이 적용된다.

1. **index.md 우선 읽기**: 질의 시작 시 반드시 index.md를 먼저 읽는다. 전체 디렉토리 grep으로 시작하지 않는다.

2. **무인용 사실 주장 금지**: 위키 또는 코드에서 확인되지 않은 사실을 답변에 포함하지 않는다. 불확실한 정보는 "위키에 기록되지 않았습니다"로 명시한다.

3. **가치 있는 답변 보존**: 재사용 가능한 분석 결과를 채팅 히스토리에 버리지 않는다. 승격 기준을 충족하면 반드시 승격한다.

4. **로그 접두사 불변**: `## [YYYY-MM-DD HH:MM]` 형식은 lint와 자동 파싱이 의존하므로 변경하지 않는다.

5. **project-wiki 미존재 시 즉시 중단**: project-wiki/ 또는 지정 프로젝트 디렉토리가 없으면 답변 생성을 시도하지 않고 project-setup 실행을 안내한다.

6. **코드 검증 의무**: 위키 페이지가 코드 구현을 설명할 때, 답변에 해당 내용을 포함하기 전 실제 소스 파일로 최신성을 확인한다.

---

## 7. Verification Checklist

답변 제출 전 아래 항목을 순서대로 확인한다.

- [ ] index.md를 가장 먼저 읽었는가
- [ ] 모든 사실 주장에 `[[wiki-id]]` 또는 외부 ref 인용이 있는가
- [ ] 코드 관련 내용은 실제 소스 파일로 검증했는가
- [ ] 승격 기준을 검토했으며, 해당하면 승격을 실행했는가
- [ ] log.md에 query 이벤트를 기록했는가
- [ ] 발견된 모순/공백을 `lint-candidate`로 log.md에 기록했는가

---

## 8. Examples

### Example 1 — 아키텍처 결정 질의

**사용자**: "인증 왜 JWT로 했어?"

**절차**:
1. `project-wiki/<project>/index.md` 읽기 → Active ADRs 섹션에서 `[[adr-001-jwt-adoption]]` 발견
2. `adrs/adr-001-jwt-adoption.md` 읽기 → `related: [module-auth-service, adr-003-refresh-token]`
3. `modules/module-auth-service.md` 읽기
4. `adrs/adr-003-refresh-token.md` 읽기
5. 답변 합성

**답변 예시**:
```markdown
JWT 채택 결정은 [[adr-001-jwt-adoption]]에 기록되어 있습니다.

핵심 이유는 세 가지였습니다:
1. 수평 확장 시 세션 스토어 없이 stateless 인증 가능 [[adr-001-jwt-adoption]]
2. 마이크로서비스 간 토큰 전파가 쉬움 [[adr-001-jwt-adoption]]
3. 당시 팀 내 Redis 운영 경험 부족 [[adr-001-jwt-adoption]]

실제 구현은 [[module-auth-service]]에 있으며, refresh token 전략은 [[adr-003-refresh-token]]에서 별도 결정되었습니다.

---
**출처**: [[adr-001-jwt-adoption]], [[module-auth-service]], [[adr-003-refresh-token]]
**연관 후속 질문**:
- refresh token 만료 처리는 어떻게 동작하나요?
- JWT 페이로드에 어떤 클레임이 들어가나요?
```

### Example 2 — 모순 발견

**상황**: `adr-002-postgres-adoption.md`는 PostgreSQL 채택을 기술하지만, `modules/data-store.md`는 MySQL을 사용한다고 기술하고 있음.

**log.md 기록**:
```markdown
## [2026-04-27 14:30] query | DB 선택 이유

- read: [[index]], [[adr-002-postgres-adoption]], [[module-data-store]]
- lint-candidate: adr-002-postgres-adoption은 PostgreSQL 채택을 기술하나 module-data-store는 MySQL 사용을 기술함 — 모순 또는 adr 미갱신 의심
```

### Example 3 — 승격 실행

**상황**: 사용자가 "auth 관련 ADR 다 정리해줘"를 요청하여 3개의 ADR을 비교 분석한 새로운 테이블을 작성함.

**승격 페이지**: `project-wiki/<project>/adrs/auth-adr-summary.md`

```yaml
---
id: auth-adr-summary
title: 인증 관련 ADR 종합 요약
project: <project-name>
type: adr
status: active
tags: [auth/summary]
refs: {}
related: [adr-001-jwt-adoption, adr-003-refresh-token, adr-007-oauth2]
created: 2026-04-27
updated: 2026-04-27
promoted_from: query
---
```
