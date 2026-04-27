---
description: 새 프로젝트 위키를 초기화합니다. "프로젝트 위키 생성", "project setup", "위키 초기화" 등의 요청 시 사용합니다.
---

# Project Setup 워크플로우

**새 프로젝트의 wiki 디렉토리 구조와 초기 파일을 생성합니다.**

---

## Phase A: 프로젝트 정보 확인

### A-1. 프로젝트 이름

사용자에게 확인:
- 프로젝트 slug (kebab-case 영문): 예) `my-awesome-app`
- 프로젝트 한글 제목: 예) `나의 앱`
- 간단한 설명 (1~2줄)

### A-2. 기존 위키 확인

```
Use Glob tool:
- pattern: "project-wiki/*/index.md"
```

동일 이름 프로젝트 위키가 이미 존재하는지 확인. 존재 시 사용자에게 알림.

---

## Phase B: 디렉토리 생성

```
Use Bash tool:
- command: "mkdir -p project-wiki/<project>/{adrs,modules,glossary,runbooks,incidents,changelog,entities,concepts,_inbox,_raw/{prs,issues,meetings,docs}}"
```

---

## Phase C: 초기 파일 생성

### C-1. index.md

```markdown
---
id: index
title: <프로젝트 한글 제목> — Project Wiki
project: <project-slug>
type: index
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

# <프로젝트 한글 제목>

> <설명>

## ADRs (Architecture Decision Records)
_아직 등록된 ADR이 없습니다._

## Modules
_아직 등록된 모듈이 없습니다._

## Glossary
_아직 등록된 용어가 없습니다._

## Runbooks
_아직 등록된 런북이 없습니다._

## Incidents
_아직 등록된 인시던트가 없습니다._

## Changelog
_아직 등록된 변경 기록이 없습니다._

## Entities
_아직 등록된 엔티티가 없습니다._

## Concepts
_아직 등록된 개념이 없습니다._
```

### C-2. log.md

```markdown
# <project-slug> — Event Log

## [YYYY-MM-DD HH:MM] setup | project wiki initialized
notes: 프로젝트 위키 초기화 완료
```

---

## Phase D: 보고

```
프로젝트 위키가 생성되었습니다:

  project-wiki/<project>/
  ├── index.md
  ├── log.md
  ├── adrs/
  ├── modules/
  ├── glossary/
  ├── runbooks/
  ├── incidents/
  ├── changelog/
  ├── entities/
  ├── concepts/
  ├── _inbox/
  └── _raw/
      ├── prs/
      ├── issues/
      ├── meetings/
      └── docs/

다음 단계:
- ADR 작성: /project-ingest 로 아키텍처 결정 기록
- 모듈 문서화: /project-ingest 로 주요 모듈 설명 추가
- 용어 정의: /project-ingest 로 프로젝트 용어집 구축
- 엔티티 등록: /project-ingest 로 도구/프레임워크/팀 기록
- 개념 정의: /project-ingest 로 패턴/기법/아키텍처 스타일 기록
```

---

## 주의사항

- 프로젝트 slug는 kebab-case 영문만 허용
- 기존 프로젝트 위키가 있으면 덮어쓰지 않음
- `.gitignore`에 `project-wiki/`가 있으면 사용자에게 알림
