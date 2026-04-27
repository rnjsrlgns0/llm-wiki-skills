---
name: wiki-query
description: Use when the user asks a question that the wiki might answer, or says "위키에서 찾아줘", "vault에 뭐 있어?", "compare X and Y", "정리해줘 위키 기반으로", "내 노트에서 RAG 관련 찾아줘". Executes the Karpathy LLM-wiki query workflow — reads index_<domain>.md first, traverses wikilinks to collect entity/concept/source-summary pages, synthesizes a cited answer, and **promotes valuable answers back into the wiki as new pages** so exploration accumulates.
---

# Wiki Query Skill

Karpathy LLM-wiki 패턴(https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)의 **query 워크플로우**를 실행한다.

> "A comparison you asked for, an analysis, a connection you discovered — these are valuable and shouldn't disappear into chat history." — Karpathy

즉, **답변 자체가 위키 자산이 될 수 있을 때는 반드시 승격**한다. 이 skill은 단순 Q&A가 아니라 **탐색 결과를 누적시키는** 것이 핵심이다.

## When to use

- 사용자가 위키에 담긴 주제를 물어봄 ("RAG 뭐였지?", "LangChain vs LlamaIndex")
- "내 노트 기반으로", "vault에서" 같은 컨텍스트 한정 질의
- 여러 페이지를 가로지르는 비교·종합 요청
- 일반 질의여도 도메인 키워드가 강하면 먼저 wiki 검색

## Procedure

### Step 1. 도메인 판정

- 질의가 어느 도메인에 속하는지 1차 판정 (10개 도메인 중, 하단 Appendix A 참조)
- **크로스 도메인 허용**: 예) "CUBIG SynTitan에서 RAG 어떻게 구현?" → `work` + `ai-llm`
- 애매하면 2~3개 후보 도메인 index를 병렬 탐색

### Step 2. `index_<domain>.md` 먼저 읽기

- 진입점은 항상 index — 전체 페이지를 grep하지 않는다
- 카테고리 섹션에서 관련 페이지 링크 식별

### Step 3. 관련 페이지 수집

- index에서 찾은 페이지 진입 → wikilink 따라 entity·concept·source-summary 확장 수집
- 필요 시에만 `raw/` 원본 읽기 (요약으로 부족할 때)
- 읽은 페이지 목록 메모 (log 기록용)

### Step 4. 답변 합성

사용자 요구 또는 스스로 판단하여 형식 선택:
- **마크다운** (기본)
- **비교 테이블** (여러 대상 대조)
- **Marp 슬라이드** (`.md` with marp header)
- **matplotlib 차트** (수치 비교)
- **Canvas 파일** (관계도)

### Step 5. 인용 필수

- **모든 사실 주장에 출처 링크**: `[[wiki page]]` 또는 `raw/...`
- 인용 없는 주장 금지 — 생성(합성)이면 "합성" 명시
- 위키에 없는 사실을 외부 지식으로 보충했다면 → 그 자체가 ingest 후보 (답변 끝에 표시)

### Step 6. 모순·공백 발견 시 명시

- 답변 중 페이지 간 모순 발견 → 답변에도 언급 + `log` 하단에 `lint-candidate: conflict <페이지 쌍>`
- 데이터 공백(관련 소스 부족) 발견 → `lint-candidate: gap <주제>`

### Step 7. 승격 판단 (Karpathy 핵심)

답변이 다음 중 **하나라도** 해당하면 **wiki 페이지로 승격**:

- 재사용 가치 있는 비교·분석·연결
- 기존 어느 페이지에도 없는 신규 구조 (새 테이블, 새 매핑)
- 사용자가 "이거 저장해줘" 등 명시적 요청
- 단일 페이지 재인용에 가까운 답변 → 승격 불필요

**승격 시:**
- `type: comparison` (기본) 또는 해당하는 `concept`
- frontmatter에 `promoted_from: query` 추가
- 경로: `wiki/<domain>/<slug>.md` (comparison은 도메인 루트)
- 크로스 도메인 답변이면 가장 주된 도메인에 저장 + 타 도메인 index에서도 링크
- 저장 후 사용자에게 "위키에 `<경로>`로 저장했다" 보고

### Step 8. `log_<domain>.md` append — 포맷 엄수

```markdown
## [YYYY-MM-DD HH:MM] query | <질문 요약 한 줄>
read:
  - wiki/<domain>/<page-a>.md
  - wiki/<domain>/entities/<x>.md
  - raw/papers/<paper>.pdf
promoted: wiki/<domain>/<new-comparison>.md    # 승격 시에만
lint-candidate:                                  # 발견 시에만
  - conflict: <페이지 쌍>
  - gap: <주제>
```

- 접두사 `## [YYYY-MM-DD HH:MM] query | ` 변형 금지
- 크로스 도메인 질의면 주 도메인 log에만 기록 (중복 방지)

## Karpathy principles to honor

1. *"good answers can be filed back into the wiki as new pages"* — 승격 단계 절대 생략 금지
2. *"exploration accumulates"* — 같은 질문을 두 번 탐색시키지 말 것. 이전 답변이 이미 wiki에 있으면 그걸 인용
3. *"LLM is good at suggesting new questions to investigate"* — 답변 말미에 후속 질문·추가 ingest 후보 1~3개 제안

## Hard rules (금지)

- 인용 없는 사실 주장
- index를 건너뛰고 전역 grep으로 답변 생성
- 승격 가치 있는 답변을 채팅에만 남기고 버림
- log 접두사 포맷 변형

## Answer template (참고)

```markdown
<답변 본문 — 모든 사실에 [[링크]] 또는 raw/ 경로>

---
**출처**: [[...]], [[...]], raw/...
**연관 후속 질문**:
- ...
- ...
```

## Verification checklist

- [ ] `index_<domain>.md` 를 먼저 읽었다
- [ ] 모든 사실 주장에 인용 링크가 있다
- [ ] 승격 조건 점검 → 해당 시 `comparison` 생성 + `promoted_from: query`
- [ ] `log_<domain>.md` 에 query 이벤트 append
- [ ] 모순·공백 발견 시 lint-candidate 기록

---

## Appendix A — 도메인 맵 (10개)

이 vault는 **Karpathy LLM-wiki 패턴**으로 운영된다.

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

### Vault 3계층 구조

| 계층 | 경로 | 소유 | 역할 |
|------|------|------|------|
| Raw Sources | `raw/` | 사용자 (불변) | 원본 문서 (논문·기사·PDF·스크린샷). LLM은 읽기만 |
| Wiki | `wiki/<domain>/` | LLM | 도메인별 엔티티·개념·소스 요약·비교 페이지 |
| Schema | `wiki/<domain>/index_<domain>.md` | 공동 | 공통 규칙 + 도메인 특이사항 |

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

1. **루트 공용 파일 없음**: index·log는 도메인별로만 존재
2. **99.attached/ 경로 불변**: 기존 wikilink 호환 위해 이동 금지. 신규 첨부는 `raw/clippings/`
3. **프로젝트는 태그로**: 별도 폴더 없이 `project/<slug>` 태그
4. **크로스 도메인 — 정본 단일 원칙**: 동일 entity/concept는 가장 관련 깊은 도메인에 **정본 1개만** 생성. 타 도메인에서는 `[[wikilink]]`로 참조. 정본 도메인 판정 기준: ① 해당 대상을 가장 자주·깊게 다루는 도메인 ② 모호하면 `index_<domain>.md` 범위 섹션 확인 ③ 그래도 애매하면 `_inbox/`에 두고 lint에서 결정

---

## Appendix B — 페이지 타입 (5가지)

`wiki/<domain>/` 하위 모든 노트는 5가지 타입 중 하나. frontmatter `type:` 필드로 지정.

### 1. `entity` — 실체

- 대상: 인물, 제품, 도구, 회사, 프로젝트, 장소 등 고유 식별 가능한 대상
- 경로: `wiki/<domain>/entities/<slug>.md`
- 구성: 정의 / 주요 사실 / 관련 링크 / 시간순 이벤트
- 예: `entities/langchain.md`, `entities/cubig-syntitan.md`, `entities/karpathy.md`

### 2. `concept` — 개념

- 대상: 이론, 패턴, 기법, 용어 — 고유명사가 아닌 추상 개념
- 경로: `wiki/<domain>/concepts/<slug>.md`
- 구성: 정의 / 주요 원리 / 변종 / 대조 개념 / 참고
- 예: `concepts/rag-pattern.md`, `concepts/agent-loop.md`

### 3. `source-summary` — 단일 소스 요약

- 대상: 논문·기사·책·영상 1건에 대한 요약
- 경로: `wiki/<domain>/sources/<slug>.md`
- `sources:` frontmatter에 원본 경로/URL 필수
- 구성: 핵심 주장 / 방법 / 결과 / 비판점 / 내 코멘트
- 예: `sources/karpathy-llm-wiki-2026.md`

### 4. `comparison` — 비교·종합

- 대상: 여러 소스·엔티티·개념을 가로지르는 비교·합성 페이지
- 경로: `wiki/<domain>/` 루트 또는 `wiki/<domain>/entities|concepts/` (주제에 따라)
- 구성: 대상 목록 / 비교 테이블 / 상황별 추천 / 근거 링크
- 예: `ai-llm/langchain-vs-llamaindex.md`

### 5. `note` — 미분류

- 대상: 위 4가지에 명확히 안 맞는 임시 노트
- 경로: `wiki/<domain>/_inbox/<slug>.md`
- lint 시 타 타입으로 승격 또는 폐기

### 타입 판별 순서

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

`type` 값을 바꿀 때 경로도 함께 이동 (예: `_inbox/foo.md` → `concepts/foo.md`). `log_<domain>.md`에 `promote` 이벤트 기록.
