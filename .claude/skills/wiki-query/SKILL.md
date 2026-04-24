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

## Source of truth

- `.claude/rules/50_query.md` — 상세 절차
- `.claude/rules/00_overview.md` — 도메인 맵
- `.claude/rules/30_page_types.md` — `comparison` 타입 규칙

## Procedure

### 1. 도메인 판정
- 질의가 어느 도메인에 속하는지 1차 판정 (9개 중)
- **크로스 도메인 허용**: 예) "CUBIG SynTitan에서 RAG 어떻게 구현?" → `work` + `ai-llm`
- 애매하면 `00_overview.md` 도메인 맵 재확인 → 2~3개 후보 도메인 index 병렬 탐색

### 2. `index_<domain>.md` 먼저 읽기
- 진입점은 항상 index — 전체 페이지를 grep하지 않는다
- 카테고리 섹션에서 관련 페이지 링크 식별

### 3. 관련 페이지 수집
- index에서 찾은 페이지 진입 → wikilink 따라 entity·concept·source-summary 확장 수집
- 필요 시에만 `raw/` 원본 읽기 (요약으로 부족할 때)
- 읽은 페이지 목록 메모 (log 기록용)

### 4. 답변 합성
사용자 요구 또는 스스로 판단하여 형식 선택:
- **마크다운** (기본)
- **비교 테이블** (여러 대상 대조)
- **Marp 슬라이드** (`.md` with marp header)
- **matplotlib 차트** (수치 비교)
- **Canvas 파일** (관계도)

### 5. 인용 필수
- **모든 사실 주장에 출처 링크**: `[[wiki page]]` 또는 `raw/...`
- 인용 없는 주장 금지 — 생성(합성)이면 "합성" 명시
- 위키에 없는 사실을 외부 지식으로 보충했다면 → 그 자체가 ingest 후보 (답변 끝에 표시)

### 6. 모순·공백 발견 시 명시
- 답변 중 페이지 간 모순 발견 → 답변에도 언급 + `log` 하단에 `lint-candidate: conflict <페이지 쌍>`
- 데이터 공백(관련 소스 부족) 발견 → `lint-candidate: gap <주제>`

### 7. 승격 판단 (Karpathy 핵심)
답변이 다음 중 **하나라도** 해당하면 **wiki 페이지로 승격**:

- ✅ 재사용 가치 있는 비교·분석·연결
- ✅ 기존 어느 페이지에도 없는 신규 구조 (새 테이블, 새 매핑)
- ✅ 사용자가 "이거 저장해줘" 등 명시적 요청
- ❌ 단일 페이지 재인용에 가까운 답변 → 승격 불필요

**승격 시:**
- `type: comparison` (기본) 또는 해당하는 `concept`
- frontmatter에 `promoted_from: query` 추가
- 경로: `wiki/<domain>/<slug>.md` (comparison은 도메인 루트)
- 크로스 도메인 답변이면 가장 주된 도메인에 저장 + 타 도메인 index에서도 링크
- 저장 후 사용자에게 "위키에 `<경로>`로 저장했다" 보고

### 8. log_<domain>.md append — **포맷 엄수**

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

- ❌ 인용 없는 사실 주장
- ❌ index를 건너뛰고 전역 grep으로 답변 생성
- ❌ 승격 가치 있는 답변을 채팅에만 남기고 버림
- ❌ log 접두사 포맷 변형

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

- [ ] index_<domain>.md 를 먼저 읽었다
- [ ] 모든 사실 주장에 인용 링크가 있다
- [ ] 승격 조건 점검 → 해당 시 `comparison` 생성 + `promoted_from: query`
- [ ] log_<domain>.md 에 query 이벤트 append
- [ ] 모순·공백 발견 시 lint-candidate 기록
