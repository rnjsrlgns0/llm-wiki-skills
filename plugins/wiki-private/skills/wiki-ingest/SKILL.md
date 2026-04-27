---
name: wiki-ingest
description: Always use when the user adds a new source (PDF, article, URL, clipping, note) to the wiki, or says things like "이거 위키에 넣어줘", "ingest this", "소스 추가", "raw에 넣고 위키 업데이트", "이 논문 정리해줘", "add this to the vault". Executes the full Karpathy LLM-wiki ingest workflow — places raw source, writes a source-summary page, updates affected entity/concept pages (often 10~15 pages), and appends to index_<domain>.md + log_<domain>.md.
---

# Wiki Ingest Skill

Karpathy LLM-wiki 패턴(https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)의 **ingest 워크플로우**를 실행한다.

> "The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping." — Karpathy

이 skill은 bookkeeping(교차참조·index·log)이 누락되지 않도록 보장하는 것이 핵심이다.

## When to use

- 사용자가 파일/URL/스크린샷을 주며 "위키에 넣어줘", "정리해줘", "ingest"
- 이미 `raw/` 에 넣힌 원본을 가리키며 "이거 요약해서 위키 반영"
- 새 소스 기반으로 관련 페이지 업데이트가 필요할 때

## Procedure

### 1. 원본 확정 (L1 · Raw Sources)
- 파일 → `raw/{papers,articles,clippings,docs}/` 중 적절 위치 배치 (papers=논문, articles=기사/블로그, clippings=뉴스 조각, docs=기타 PDF/문서)
- URL → `raw/articles/<slug>.md` 생성, **메타데이터만**(URL, 수집일, 제목). 본문 복사는 선택
- **원본 수정 금지** — raw/ 는 불변

### 2. 소스 읽기 & 핵심 논점 확인
- 전문 읽기
- Karpathy 원칙: *"LLM reads the source, discusses key takeaways with you"* — 애매한 해석·관점이 필요하면 **한 줄 질문 1~2개**로 사용자에게 확인
- 자명한 경우는 생략하고 진행

### 3. 도메인 판정
10개 도메인 중 1차 선택:
`ai-llm` / `engineering` / `data-ml` / `finance` / `work` / `career` / `research` / `readings` / `journal` / `life`

- 경계 모호 시 `wiki/<domain>/index_<domain>.md` 상단 "범위/경계" 섹션 확인
- 판정 애매하면 `wiki/<domain>/_inbox/` 에 임시 배치 후 lint에서 재분류

### 4. source-summary 페이지 작성 (L2 · Wiki)
- 경로: `wiki/<domain>/sources/<slug>.md`
- `type: source-summary`
- frontmatter 필수 필드: `id`(=파일명) / `title` / `domain` / `type` / `status: active` / `tags` / `sources` (원본 경로·URL 리스트) / `created` / `updated`
- 본문 구성:
  - **핵심 주장** (3~5 bullet)
  - **방법** (해당 시)
  - **결과·사실**
  - **비판점 / 한계**
  - **파생 아이디어 · 내 코멘트**

### 5. 영향받는 entity·concept 업데이트 (L2)
**Karpathy 핵심 원칙: 하나의 소스가 10~15개 페이지를 수정할 수 있다.**

소스에서 언급된 모든 고유명사(인물·제품·도구·회사·프로젝트)와 추상 개념(패턴·기법·이론) 각각에 대해:

- **기존 페이지 존재** → 본문에 신규 사실 추가, `updated` 필드 갱신
  - 새 소스가 기존 주장과 **모순**되면 → 명시적으로 기록 ("새 소스 X는 이전 Y 주장을 뒤집음" 또는 "강화함"). 숨기지 않는다.
- **기존 페이지 부재** → 새 페이지 생성
  - entity: `wiki/<domain>/entities/<slug>.md`
  - concept: `wiki/<domain>/concepts/<slug>.md`
  - 타입 판별 순서: ① 원본 1건 대응 → `source-summary` ② 고유명사·식별 대상 → `entity` ③ 2개 이상을 가로지름 → `comparison` ④ 추상 개념 → `concept` ⑤ 해당 없음 → `note` (`_inbox/`)

### 6. 양방향 교차참조
- source-summary → 언급된 모든 entity/concept 페이지로 `[[wikilink]]`
- entity/concept → 본 source-summary로 역 wikilink (근거로 인용)

### 7. index_<domain>.md 갱신 (L3)
신규 페이지 각각에 대해:
- 해당 도메인의 카테고리 섹션(엔티티/개념/소스 요약/비교 등)에 링크 + 한 줄 요약 추가
- 기존 페이지 대규모 수정 시 index 요약도 갱신 검토

### 8. log_<domain>.md append (L3) — **포맷 엄수**
Karpathy 원칙: 접두사 일관성 → unix 파싱 가능해야 한다.

```markdown
## [YYYY-MM-DD HH:MM] ingest | <title>
source: raw/<path or URL>
updated:
  - wiki/<domain>/sources/<new-summary>.md (new)
  - wiki/<domain>/entities/<x>.md
  - wiki/<domain>/concepts/<y>.md (new)
notes: <모순·결정·특이사항 1~3줄>
```

- 접두사 `## [YYYY-MM-DD HH:MM] ingest | ` **절대 변형 금지**
- 시간은 현재 로컬 시각

### 9. 크로스 도메인 영향 확인
- 본 소스가 타 도메인 페이지와도 연관되면(예: `ai-llm` 소스지만 `work/entities/cubig-syntitan` 에도 영향):
  - 해당 도메인 페이지 업데이트
  - **해당 도메인의 `log_<그 도메인>.md`에도 ingest 이벤트 append**

### 10. 보고
- 변경된 페이지 수 요약
- 10개 초과 수정 시 → 목록 제시 + 사용자 확인 권장
- 신규 질문·데이터 갭 발견 시 → `log` 하단에 `lint-candidate:` 로 메모

## Karpathy principles to honor

1. *"one source can modify 10~15 pages"* — entity/concept 연쇄 업데이트 생략하지 말 것
2. *"discusses key takeaways with you"* — 애매할 땐 먼저 물어볼 것
3. *"integrate incrementally — record contradictions, strengthen or challenge prior claims"* — 모순 숨기지 말 것
4. *"one source at a time is preferred"* — 대량 배치 ingest 시에도 각 원본은 개별로 raw/에 보존

## Hard rules (금지)

- raw/ 원본 수정
- `99.attached/` 경로 변경 (기존 wikilink 호환)
- log 접두사 포맷 변형
- 확정되지 않은 도메인의 `sources/`에 저장 (애매하면 `_inbox/`)
- 한 ingest에서 10개 초과 페이지 **조용히** 수정 (반드시 보고)
- **wiki 작업(ingest/lint/query)은 반드시 해당 skill을 호출하여 실행**. Write/Edit로 파일만 생성하는 것은 skill 사용이 아님
- **skill 로드 후 Step 1~10 전체를 따를 것**. source-summary 생성(Step 4)만으로 끝내지 말고, entity/concept 추출(Step 5)·교차참조(Step 6)·index(Step 7)·log(Step 8)·크로스 도메인(Step 9)까지 완료해야 1회 ingest 종료. 속도를 위해 절차 생략 금지

## Verification checklist (ingest 완료 후 자기 점검)

- [ ] raw/ 에 원본 배치됨 (또는 URL 메타 생성)
- [ ] `wiki/<domain>/sources/<slug>.md` 생성, frontmatter 필수 필드 완비
- [ ] 언급된 entity/concept 페이지 모두 업데이트 또는 신규 생성
- [ ] 양방향 wikilink 연결
- [ ] `index_<domain>.md` 에 신규 페이지 등재
- [ ] `log_<domain>.md` 에 규정 포맷으로 append
- [ ] 크로스 도메인 영향 확인됨

---

## Reference: Vault Overview

이 vault는 **Karpathy LLM-wiki 패턴**으로 운영된다. 참고: https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f

### 3계층 구조

| 계층 | 경로 | 소유 | 역할 |
|------|------|------|------|
| Raw Sources | `raw/` | 사용자 (불변) | 원본 문서 (논문·기사·PDF·스크린샷). LLM은 읽기만 |
| Wiki | `wiki/<domain>/` | LLM | 도메인별 엔티티·개념·소스 요약·비교 페이지 |
| Schema | `wiki/<domain>/index_<domain>.md` | 공동 | 도메인 카탈로그 + 도메인 특이 규칙 |

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

1. **공통 규칙 < 도메인 규칙**: 일반 원칙만 적용. 도메인 예외는 `index_<domain>.md`가 우선
2. **루트 공용 파일 없음**: index·log는 도메인별로만 존재
3. **99.attached/ 경로 불변**: 기존 wikilink 호환 위해 이동 금지. 신규 첨부는 `raw/clippings/`
4. **프로젝트는 태그로**: 별도 폴더 없이 `project/<slug>` 태그 사용
5. **크로스 도메인 — 정본 단일 원칙**: 동일 entity/concept는 가장 관련 깊은 도메인에 **정본 1개만** 생성. 타 도메인에서는 `[[wikilink]]`로 참조. 정본 도메인 판정 기준: ① 해당 대상을 가장 자주·깊게 다루는 도메인 ② 모호하면 `index_<domain>.md` 범위 섹션 확인 ③ 그래도 애매하면 `_inbox/`에 두고 lint에서 결정. 크로스 도메인으로 영향받는 페이지 수정 시 해당 도메인의 `log_<domain>.md`에도 이벤트 기록

---

## Reference: Frontmatter Standard

모든 `wiki/` 하위 노트는 아래 YAML frontmatter를 **필수**로 포함한다.

### 필수 필드

```yaml
---
id: <kebab-case-slug>          # 파일명과 동일 권장. 전역 유일
title: <사람이 읽는 제목>
domain: ai-llm | engineering | data-ml | finance | work | career | research | readings | journal | life
type: entity | concept | source-summary | comparison | note
status: draft | active | outdated | archived
tags: [project/<slug>, tech/<slug>, ...]
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

기존 노트의 frontmatter 필드 매핑:
- `author: hunhun` → 유지 (선택 필드)
- `type: "[[📌 학습노트]]"` 등 → 새로운 `type` 값(concept/source-summary 등)으로 재지정
- `topic: Agent` → `tags: [tech/agent]` 로 변환
- `source: URL` → `sources: [URL]` (리스트화)
- `created:` 유지

---

## Reference: Tag Namespaces

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

---

## Reference: Page Types

`wiki/<domain>/` 하위 모든 노트는 5가지 타입 중 하나. frontmatter `type:` 필드로 지정.

### 5가지 타입

#### 1. `entity` — 실체
- 대상: 인물, 제품, 도구, 회사, 프로젝트, 장소 등 고유 식별 가능한 대상
- 경로: `wiki/<domain>/entities/<slug>.md`
- 구성: 정의 / 주요 사실 / 관련 링크 / 시간순 이벤트
- 예: `entities/langchain.md`, `entities/cubig-syntitan.md`, `entities/karpathy.md`

#### 2. `concept` — 개념
- 대상: 이론, 패턴, 기법, 용어 — 고유명사가 아닌 추상 개념
- 경로: `wiki/<domain>/concepts/<slug>.md`
- 구성: 정의 / 주요 원리 / 변종 / 대조 개념 / 참고
- 예: `concepts/rag-pattern.md`, `concepts/agent-loop.md`

#### 3. `source-summary` — 단일 소스 요약
- 대상: 논문·기사·책·영상 1건에 대한 요약
- 경로: `wiki/<domain>/sources/<slug>.md`
- `sources:` frontmatter에 원본 경로/URL 필수
- 구성: 핵심 주장 / 방법 / 결과 / 비판점 / 내 코멘트
- 예: `sources/karpathy-llm-wiki-2026.md`

#### 4. `comparison` — 비교·종합
- 대상: 여러 소스·엔티티·개념을 가로지르는 비교·합성 페이지
- 경로: `wiki/<domain>/` 루트 또는 `wiki/<domain>/entities|concepts/` (주제에 따라)
- 구성: 대상 목록 / 비교 테이블 / 상황별 추천 / 근거 링크
- 예: `ai-llm/langchain-vs-llamaindex.md`

#### 5. `note` — 미분류
- 대상: 위 4가지에 명확히 안 맞는 임시 노트
- 경로: `wiki/<domain>/_inbox/<slug>.md`
- lint 시 타 타입으로 승격 또는 폐기

### 판별 순서

1. 원본 1건에 대응하면 → `source-summary`
2. 고유명사·식별 대상이면 → `entity`
3. 2개 이상을 가로지르면 → `comparison`
4. 추상 개념이면 → `concept`
5. 위에 맞지 않으면 → `note` (`_inbox/`)

### 페이지 간 관계

- `source-summary` → `entity`/`concept`의 근거로 인용됨 (wikilink)
- `comparison` → 여러 `entity`/`concept`을 참조
- `entity`·`concept` 간 상호 참조 자유

### 타입 변경

`type` 값을 바꿀 때 경로도 함께 이동(예: `_inbox/foo.md` → `concepts/foo.md`). `log_<domain>.md`에 `promote` 이벤트 기록.

---

## Reference: Ingest Procedure (Detail)

### 절차 상세

1. **원본 확정**
   - 파일이면 `raw/papers/` · `raw/articles/` · `raw/clippings/` · `raw/docs/` 중 적절한 곳에 배치
   - URL이면 `raw/articles/<slug>.md`에 메타데이터(URL, 수집일, 제목)만 저장. 본문 복사는 선택
2. **소스 읽기 및 도메인 판정**
   - 전문을 읽고 10개 도메인 중 1차 도메인 결정 (경계 모호 시 `index_<domain>.md` 범위 섹션 참조)
3. **source-summary 페이지 작성**
   - 경로: `wiki/<domain>/sources/<slug>.md`
   - 타입: `source-summary`
   - frontmatter `sources:`에 원본 경로·URL 기재
   - 구성: 핵심 주장 / 방법 / 결과 / 비판점 / 파생 아이디어
4. **영향받는 entity · concept 업데이트**
   - 소스에서 언급된 고유명사/개념 각각에 대해:
     - 기존 entity/concept 페이지가 있으면 → 해당 페이지의 본문·관련 링크·`updated` 필드 갱신, 본 소스에서 얻은 신규 사실 추가
     - 없으면 → 새 entity/concept 페이지 생성 (경로 규칙 준수)
   - 한 소스로 3~15개 기존 페이지가 영향받을 수 있음 (정상)
5. **교차 참조 추가**
   - 본 source-summary에서 관련 entity·concept로 wikilink
   - 관련 entity·concept 페이지에서도 본 source-summary로 역 wikilink
6. **index_<domain>.md 갱신**
   - 신규 페이지 링크와 한 줄 요약 추가
   - 기존 카테고리 섹션에 배치 (엔티티/개념/소스 요약 등)
7. **log_<domain>.md 기록**
   - 형식: `## [YYYY-MM-DD HH:MM] ingest | <title>` 접두사 고정
   - 본문: `source:` 원본 경로, `updated:` 영향받은 페이지 목록
8. **크로스 도메인 영향 확인**
   - 본 소스가 타 도메인(예: `ai-llm`이지만 `work/entities/cubig-syntitan`에도 영향) 페이지와 연관되면 해당 도메인 페이지도 업데이트 + 해당 `log_<그 도메인>.md`에도 `ingest` 이벤트 append

### 배치 vs 개별

- **개별 ingest**: 중요 소스 1건 (논문, 핵심 기사) — 전체 절차 철저 수행
- **배치 ingest**: 가벼운 소스 다수 (뉴스 클리핑 모음) — 단일 source-summary에 합쳐도 가능, 단 각각의 원본은 `raw/articles/`에 개별 저장
- **배치 크기 제한**: 1회 배치 ingest는 **10편 이하**. 플랜에서 배치를 분할했으면 임의로 합치지 말 것
