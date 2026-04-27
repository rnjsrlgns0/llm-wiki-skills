---
name: project-sync
description: Use when the user says "위키 동기화", "project sync", "코드 변경 반영", "모듈 문서 최신화", or after significant code changes. Detects code changes via git diff, maps them to project wiki module/ADR pages, and marks affected pages as outdated or suggests updates.
---

# project-sync

> 핵심 원칙: "결정의 맥락이 코드보다 오래 산다"

코드는 변한다. 그러나 왜 그렇게 결정했는지, 어떤 트레이드오프를 감수했는지는
코드에 남지 않는다. project-sync는 코드 변경이 발생했을 때 관련 위키 페이지를
자동으로 감지하여 문서가 코드 현실을 반영하도록 유지한다.

**트리거 방식**: 수동 전용. git hook, 자동 실행 없음.

---

## 전제 조건

- 대상 프로젝트의 `project-wiki/<project>/` 디렉토리가 존재해야 한다.
- 위키가 없으면 실행을 중단하고 사용자에게 안내한다:
  > "project-wiki/<project>/ 가 존재하지 않습니다. project-setup 스킬을 먼저 실행하세요."
- 실행 전 반드시 git diff 범위를 사용자와 확인한다.

---

## 위키 페이지 타입

| 타입        | 설명                                                              | 저장 위치       | 코드 변경 영향 가능성 |
|-------------|------------------------------------------------------------------|-----------------|----------------------|
| `module`    | 코드 컴포넌트 문서                                               | `modules/`      | 높음 (주요 대상)     |
| `adr`       | 아키텍처 결정 기록                                               | `adrs/`         | 중간 (모순 감지)     |
| `runbook`   | 운영 절차서                                                      | `runbooks/`     | 중간 (스크립트 변경) |
| `glossary`  | 프로젝트 용어집                                                  | `glossary/`     | 낮음                 |
| `incident`  | 인시던트 기록                                                    | `incidents/`    | 낮음                 |
| `changelog` | 변경 기록 요약                                                   | `changelog/`    | 낮음                 |
| `entity`    | 프로젝트 스코프 엔티티: 도구, 프레임워크, 팀, 사람               | `entities/`     | 중간 (도구/설정 변경) |
| `concept`   | 프로젝트 스코프 개념: 패턴, 기법, 아키텍처 스타일               | `concepts/`     | 중간 (구현 변경)     |

---

## 위키 페이지 Frontmatter 명세

모든 wiki 페이지는 아래 frontmatter를 가진다.

```yaml
---
id: <slug>
title: <title>
project: <project-name>
type: adr | module | glossary | runbook | incident | changelog | entity | concept
status: draft | active | deprecated | superseded
tags: [...]
refs:
  prs: [#123]
  issues: [#789]
  commits: [abc1234]
related: [<wiki page id>, ...]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

**status 값 정의**:
- `draft` — 작성 중, 미확정
- `active` — 현재 유효
- `deprecated` — 더 이상 유효하지 않음 (참조 코드/모듈 삭제됨)
- `superseded` — 다른 결정/문서로 대체됨

---

## 동기화 절차

### Step 1. 변경 파일 수집

```bash
git diff --name-only <base>..HEAD
```

- `<base>`가 지정되지 않은 경우 기본값: `HEAD~10` (최근 10 커밋)
- 범위 확정 전 반드시 사용자에게 확인:
  > "git diff HEAD~10..HEAD 범위로 진행할까요? 다른 base 커밋이나 브랜치가 있으면 알려주세요."
- 변경 파일을 카테고리별로 분류:
  - `new` — 신규 추가
  - `modified` — 수정됨
  - `deleted` — 삭제됨
  - `renamed` — 이름 변경 (이전 경로 → 새 경로)
- **제외 대상** (기본값, 사용자 요청 시 포함 가능):
  - 테스트 파일: `**/test_*.py`, `**/*.test.*`, `**/*.spec.*`, `tests/`, `__tests__/`
  - 설정 파일: `*.config.*`, `*.env`, `.env*`, `pyproject.toml`, `package.json`, `Makefile`
  - 생성 파일: `**/migrations/`, `**/generated/`, `dist/`, `build/`, `*.lock`

---

### Step 2. 변경 파일을 위키 페이지에 매핑

각 변경 파일에 대해 `project-wiki/<project>/` 전체를 탐색한다.

**매핑 탐지 기준** (아래 조건 중 하나 이상 충족 시 매핑):

1. **본문 파일 경로 언급**: 위키 페이지 본문에 변경된 파일의 경로(또는 파일명)가 명시적으로 포함된 경우
2. **refs.commits 교차**: 위키 페이지 frontmatter `refs.commits`에 포함된 커밋이 해당 변경과 연관된 경우
3. **모듈 경로 포함**: `modules/` 페이지의 본문에 변경 파일이 속한 디렉토리 경로가 기술된 경우
4. **엔티티 경로 포함**: `entities/` 페이지의 본문에 변경된 도구/프레임워크의 소스 코드 또는 설정 파일 경로가 기술된 경우 — 해당 entity 페이지가 영향을 받을 수 있음
5. **개념 경로 포함**: `concepts/` 페이지의 본문에 변경된 파일이 구현하는 패턴이나 기법이 기술된 경우 — 해당 concept 페이지가 영향을 받을 수 있음

**매핑 결과 구조**:
```
{
  "src/auth/jwt.py": ["modules/auth-module", "adrs/adr-003-jwt-strategy"],
  "src/api/routes.py": ["modules/api-gateway"],
  "src/infra/redis.py": ["entities/redis", "concepts/caching-strategy"],
  "src/utils/cache.py": []   ← 매핑 없음 (미문서화)
}
```

매핑 없는 파일은 Step 5에서 별도 보고한다.

---

### Step 3. 영향 평가

각 매핑된 위키 페이지에 대해 아래 기준으로 영향을 평가한다.

#### module 페이지

| 변경 유형           | 평가 결과                                                          |
|---------------------|--------------------------------------------------------------------|
| 참조 파일 삭제됨    | `status: deprecated` 마킹 후보 → 사용자 승인 필요                 |
| 참조 파일 수정됨    | `status: outdated` + `sync-note` 추가 → 자동 적용                 |
| 참조 파일 이름 변경 | 본문 내 파일 경로 참조 업데이트 → 자동 적용                       |

`sync-note` 추가 형식 (frontmatter 바로 아래):
```markdown
> **sync-note**: source file `<path>` changed on <YYYY-MM-DD>. Review and update this page.
```

#### adr 페이지

| 변경 유형                                    | 평가 결과                                                               |
|----------------------------------------------|-------------------------------------------------------------------------|
| ADR이 결정한 내용과 코드 변경이 상충될 가능성 | `lint-candidate: adr-contradiction` 플래그 → 사용자 검토 필요          |
| ADR이 참조한 모듈이 삭제됨                   | `status: superseded` 마킹 후보 → 사용자 승인 필요                      |

ADR 모순 감지는 휴리스틱 기반이다. 코드 변경 파일 경로와 ADR 본문의 기술 스택, 패턴, 모듈명을 비교하여 상충 가능성을 판단한다. 확신할 수 없으면 플래그만 설정하고 사용자가 직접 판단하도록 한다.

#### runbook 페이지

| 변경 유형                          | 평가 결과                                                |
|------------------------------------|----------------------------------------------------------|
| 참조된 스크립트/CLI 도구가 변경됨  | `status: outdated` + `sync-note` 추가 → 자동 적용       |

#### entity 페이지

| 변경 유형                                        | 평가 결과                                                          |
|--------------------------------------------------|--------------------------------------------------------------------|
| entity가 참조하는 도구/프레임워크가 업그레이드되거나 변경됨 | `status: outdated` + `sync-note` 추가 → 자동 적용        |
| entity가 참조하는 설정 파일이나 초기화 코드가 변경됨        | `sync-note` 추가 후 업데이트 제안 → 자동 적용             |

#### concept 페이지

| 변경 유형                                        | 평가 결과                                                          |
|--------------------------------------------------|--------------------------------------------------------------------|
| concept이 기술하는 패턴의 구현 코드가 크게 변경됨 | `status: outdated` + `sync-note` 추가 → 자동 적용                |

#### glossary / incident / changelog 페이지

- 코드 변경에 의한 직접 수정 없음
- 매핑 탐지 시 사용자에게 알림만 제공 ("확인 필요" 목록에 포함)

---

### Step 4. 변경 적용

**자동 적용 항목** (사용자 확인 없이 진행):
- 파일 이름 변경에 따른 위키 페이지 내 파일 경로 참조 업데이트
- `updated` frontmatter 날짜 갱신 (오늘 날짜로)
- `status: outdated` 마킹 + `sync-note` 추가

**사용자 승인 필요 항목** (목록 제시 후 승인 받아 진행):
- `status: deprecated` 마킹 (참조 파일 삭제로 인한 모듈 페이지)
- `status: superseded` 마킹 (참조 모듈 삭제로 인한 ADR 페이지)
- `lint-candidate: adr-contradiction` 플래그 설정

**적용 전 요약 제시 형식**:
```
자동 적용 예정:
  - modules/auth-module.md → status: outdated, sync-note 추가
  - modules/api-gateway.md → 파일 경로 참조 업데이트 (src/api/v1/routes.py → src/api/v2/routes.py)
  - entities/redis.md → status: outdated, sync-note 추가
  - concepts/caching-strategy.md → status: outdated, sync-note 추가

사용자 승인 필요:
  - modules/legacy-parser.md → status: deprecated (참조 파일 src/parser/old.py 삭제됨)
  - adrs/adr-005-rest-only.md → lint-candidate: adr-contradiction (GraphQL 도입 가능성 감지)

진행할까요? (자동 적용만 먼저 진행 / 전체 승인 / 취소)
```

---

### Step 5. 미문서화 파일 보고

매핑되지 않은 신규/수정 파일 목록을 출력한다.

```
미문서화 파일 (위키 module 페이지 없음):
  - src/notifications/email_sender.py  [new]
  - src/utils/retry.py                 [modified]
  - src/jobs/cleanup_worker.py         [new]

이 파일들에 대한 module 페이지 생성을 권장합니다.
project-ingest 스킬로 module 페이지를 생성할 수 있습니다.

미문서화 도구/프레임워크 (위키 entity 페이지 없음):
  - src/infra/celery_app.py에서 Celery 도입이 감지되었으나 entity 페이지가 없습니다.

미문서화 패턴/개념 (위키 concept 페이지 없음):
  - src/core/circuit_breaker.py에서 Circuit Breaker 패턴 구현이 감지되었으나 concept 페이지가 없습니다.
```

자동 생성하지 않는다. 보고만 한다.

---

### Step 6. 이벤트 로그 기록

동기화 완료 후 `project-wiki/<project>/log.md`에 아래 형식으로 append한다.

```markdown
## [YYYY-MM-DD HH:MM] sync | <project>
base: <commit-range>
changed_files: N
affected_pages: N
  - outdated: N
  - deprecated: N
  - renamed_refs: N
  - adr_conflicts: N
  - entity_outdated: N
  - concept_outdated: N
unmapped_files: N
actions:
  - auto-fixed: <요약 (예: 3개 페이지 outdated 마킹, 1개 파일 경로 업데이트)>
  - pending_user: <요약 (예: 1개 deprecated 승인 대기, 1개 ADR 모순 플래그 대기)>
```

**로그 형식 규칙**:
- `## [YYYY-MM-DD HH:MM] sync | ` 접두사는 절대 수정하지 않는다.
- 기존 로그 항목을 수정하지 않는다. 항상 파일 끝에 append한다.
- `log.md`가 없으면 새로 생성한다.

---

## 하드 규칙

1. **수동 트리거만** — git hook, 파일 감시, 자동 실행 절대 금지
2. **소스 코드 수정 금지** — wiki 페이지만 수정한다. 실제 `.py`, `.ts`, `.go` 등의 소스 파일은 절대 건드리지 않는다.
3. **위키 페이지 자동 삭제 금지** — 상태를 `deprecated`로 마킹할 뿐, 파일을 삭제하지 않는다.
4. **파일 이름 변경 참조 업데이트는 자동 적용 가능** — 경로 참조 수정은 안전한 변경이다.
5. **deprecated/superseded 상태 변경은 사용자 승인 필수** — 되돌리기 어려운 의미론적 변경이다.
6. **위키 없으면 중단** — `project-wiki/<project>/`가 없으면 즉시 중단하고 project-setup 안내.
7. **git diff 범위는 반드시 사용자와 확인** — 기본값(HEAD~10)으로 가정하지 않는다.

---

## 검증 체크리스트

동기화 완료 후 아래 항목을 확인한다.

- [ ] Git diff 범위를 사용자와 확인했다
- [ ] 모든 변경 파일을 매핑했다 (또는 미매핑으로 보고했다)
- [ ] 영향받는 위키 페이지를 식별했다
- [ ] 자동 적용 항목을 처리하고 결과를 보고했다
- [ ] 사용자 승인 필요 항목을 별도로 제시했다
- [ ] 미문서화 파일 목록을 보고했다
- [ ] `log.md`에 이벤트를 append했다

---

## 실행 흐름 요약

```
트리거 감지
    │
    ▼
project-wiki/<project>/ 존재 확인
    │ 없음 → "project-setup 먼저 실행하세요" 중단
    │
    ▼
[Step 1] git diff 범위 사용자 확인 → 변경 파일 수집 및 분류
    │
    ▼
[Step 2] 변경 파일 → wiki 페이지 매핑 탐색 (modules / adrs / entities / concepts 포함)
    │
    ▼
[Step 3] 각 위키 페이지 영향 평가 (module / adr / runbook / entity / concept 별 기준)
    │
    ▼
[Step 4] 자동 적용 실행 + 사용자 승인 항목 제시
    │
    ▼
[Step 5] 미문서화 파일 보고 (module 없음 / entity 없음 / concept 없음, 제안만)
    │
    ▼
[Step 6] log.md append
    │
    ▼
완료 요약 출력
```

---

## 위키 디렉토리 구조 참조

```
project-wiki/<project>/
├── modules/
├── adrs/
├── runbooks/
├── glossary/
├── incidents/
├── changelog/
├── entities/
├── concepts/
├── _raw/
└── log.md
```
