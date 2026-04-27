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
Appendix B (Frontmatter 표준) 기준:
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

---

## Appendix A — Vault 전체 구조

이 vault는 **Karpathy LLM-wiki 패턴**으로 운영된다. 참고: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f

### 3계층 구조

| 계층 | 경로 | 소유 | 역할 |
|------|------|------|------|
| Raw Sources | `raw/` | 사용자 (불변) | 원본 문서 (논문·기사·PDF·스크린샷). LLM은 읽기만 |
| Wiki | `wiki/<domain>/` | LLM | 도메인별 엔티티·개념·소스 요약·비교 페이지 |
| Schema | `wiki/<domain>/index_<domain>.md` | 공동 | 공통 규칙 + 도메인 특이사항 |

### 역할 분담

- **사용자**: 원시 소스 수집, 탐색, 질의, 최종 승인
- **LLM (Claude Code)**: 요약·교차참조·분류·lint — 유지 부담 제거

### 도메인 10개

| 디렉토리 | 범위 |
|---------|------|
| `wiki/ai-llm/` | LLM·에이전트·RAG·프롬프트·AI 트렌드 |
| `wiki/engineering/` | SW 개발·도구·인프라 |
| `wiki/data-ml/` | 데이터사이언스·ML·DL·수학 |
| `wiki/finance/` | 투자·금융·자산·자동매매 |
| `wiki/work/` | 업무 프로젝트 엔티티·의사결정 로그 |
| `wiki/career/` | 이력서·포트폴리오·블로그·채용 |
| `wiki/research/` | 학술 논문·기술 리포트·벤치마크·연구 블로그 |
| `wiki/readings/` | 책·영화·뉴스 클리핑·개인 독서/관람 기록 |
| `wiki/journal/` | 데일리·위클리·일정 |
| `wiki/life/` | 건강·결혼·주거·여행·개인금융 |

도메인 간 경계가 모호하면 각 도메인의 `index_<domain>.md` 상단 "범위/경계" 섹션을 우선 참고한다. 판단이 끝까지 애매하면 `wiki/<domain>/_inbox/`에 두고 lint에서 재분류.

### 디렉토리 트리

```
PrivateWiki/
├── raw/                      ← 원본 소스 (불변)
│   ├── papers/ articles/ clippings/ docs/
├── wiki/<domain>/
│   ├── index_<domain>.md     ← 도메인 카탈로그 + 도메인 특이 규칙
│   ├── log_<domain>.md       ← 도메인 전용 append-only 로그
│   ├── sources/              ← 소스 요약 (raw 원본 1건+ 대응)
│   ├── entities/ concepts/ _inbox/
├── _legacy/                  ← 기존 PARA 원본 (이관 안전망)
└── 99.attached/              ← 기존 첨부 (wikilink 호환 유지)
```

### 핵심 원칙

1. **공통 규칙 < 도메인 규칙**: 일반 원칙만 존재. 도메인 예외는 `index_<domain>.md`가 우선
2. **루트 공용 파일 없음**: index·log는 도메인별로만 존재
3. **99.attached/ 경로 불변**: 기존 wikilink 호환 위해 이동 금지. 신규 첨부는 `raw/clippings/`
4. **프로젝트는 태그로**: 별도 폴더 없이 `project/<slug>` 태그 (Appendix C 참조)
5. **크로스 도메인 — 정본 단일 원칙**: 동일 entity/concept는 가장 관련 깊은 도메인에 **정본 1개만** 생성. 타 도메인에서는 `[[wikilink]]`로 참조. 정본 도메인 판정 기준: ① 해당 대상을 가장 자주·깊게 다루는 도메인 ② 모호하면 `index_<domain>.md` 범위 섹션 확인 ③ 그래도 애매하면 `_inbox/`에 두고 lint에서 결정. 크로스 도메인으로 영향받는 페이지 수정 시 해당 도메인의 `log_<domain>.md`에도 이벤트 기록

---

## Appendix B — Frontmatter 표준

모든 `wiki/` 하위 노트는 아래 YAML frontmatter를 **필수**로 포함한다.

### 필수 필드

```yaml
---
id: <kebab-case-slug>          # 파일명과 동일 권장. 전역 유일
title: <사람이 읽는 제목>
domain: ai-llm | engineering | data-ml | finance | work | career | research | readings | journal | life
type: entity | concept | source-summary | comparison | note
status: draft | active | outdated | archived
tags: [project/<slug>, tech/<slug>, ...]   # Appendix C 참조
sources: [raw/papers/xxx.pdf, https://...]  # 원본 경로 또는 URL 목록
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

### 선택 필드

- `aliases: [다른 이름, Other Name]` — Obsidian 별칭
- `author: hunhun` — 기존 노트 호환
- `topic: <short>` — 기존 노트 호환 (권장하지 않음, tags로 대체)
- `promoted_from: query | source-summary` — 질의 승격 기록

### 예시

**Entity (도구):**
```yaml
---
id: langchain
title: LangChain
domain: ai-llm
type: entity
status: active
tags: [tech/langchain, tech/agent-pattern]
sources: [https://python.langchain.com/docs/]
created: 2026-04-13
updated: 2026-04-13
aliases: [랭체인]
---
```

**Source summary (논문 요약):**
```yaml
---
id: karpathy-llm-wiki-2026
title: Karpathy — LLM Wiki Pattern
domain: ai-llm
type: source-summary
status: active
tags: [tech/knowledge-management, person/karpathy]
sources: [https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f]
created: 2026-04-13
updated: 2026-04-13
---
```

**Concept:**
```yaml
---
id: rag-pattern
title: Retrieval-Augmented Generation
domain: ai-llm
type: concept
status: active
tags: [tech/rag, tech/retrieval]
sources: []
created: 2026-04-13
updated: 2026-04-13
---
```

### 규칙

1. `id`는 파일명(확장자 제외)과 반드시 일치
2. `updated`는 수정 시마다 갱신 (LLM이 자동 갱신)
3. `status: outdated`는 더 새로운 소스로 교체 필요함을 표시 — lint가 수집
4. `sources`가 비어있는 concept 페이지는 합성된 지식임을 의미
5. 한국어 제목도 허용하나, `id`(파일명)는 영문 kebab-case로 통일

### 레거시 호환

기존 노트의 frontmatter 필드는 이관 시 아래와 같이 매핑:
- `author: hunhun` → 유지 (선택 필드)
- `type: "[[📌 학습노트]]"` 등 → 새로운 `type` 값(concept/source-summary 등)으로 재지정
- `topic: Agent` → `tags: [tech/agent]` 로 변환
- `source: URL` → `sources: [URL]` (리스트화)
- `created:` 유지

---

## Appendix C — 태그 네임스페이스

모든 태그는 **네임스페이스/슬러그** 형식. 중첩은 `/`로 구분 (Obsidian nested tags).

### 네임스페이스

| 프리픽스 | 용도 | 예시 |
|---------|------|------|
| `project/` | 프로젝트 추적 (폴더 대체) | `project/cubig-syntitan`, `project/auto-stock` |
| `tech/` | 기술·프레임워크·도구·기법 | `tech/langchain`, `tech/rag`, `tech/docker` |
| `person/` | 인물 | `person/karpathy`, `person/hunhun` |
| `company/` | 회사·조직 | `company/cubig`, `company/anthropic` |
| `status/` | 특수 상태 플래그 | `status/todo`, `status/review`, `status/deprecated` |

### 프로젝트 태그 목록 (현재)

**활성 (`status: active`):**
- `project/cubig-syntitan` — CUBIG SynTitan 플랫폼
- `project/cubig-business` — CUBIG 비즈니스 문서
- `project/auto-stock` — 주식 자동매매
- `project/auto-btc` — 암호화폐 자동매매
- `project/lightcode`
- `project/wordpress-adsense`

**아카이브 (`status: archived`):**
- `project/wavus` — Wavus 업무
- `project/electrip`
- `project/poc-nonghyup` — 농협은행 PoC
- `project/poc-danal` — 다날 PoC
- `project/dify`
- `project/cubig-llmcapsule`

신규 프로젝트 추가 시 이 목록과 `wiki/work/index_work.md`에 동시 등록.

### 작성 규칙

1. 태그 값은 kebab-case 영문 (한글 금지)
2. frontmatter `tags:` 리스트에 모두 기재 (본문 `#tag` 사용 지양, 일관성 위해)
3. 페이지당 태그 3~7개 권장. 10개 초과 시 분할 검토
4. 프로젝트 태그는 페이지당 최대 2개 (주 프로젝트 + 관련 프로젝트)
