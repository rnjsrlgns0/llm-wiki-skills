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

## Source of truth

- `.claude/rules/40_ingest.md` — 상세 절차
- `.claude/rules/10_frontmatter.md` — frontmatter 필수 필드
- `.claude/rules/20_tags.md` — 태그 네임스페이스
- `.claude/rules/30_page_types.md` — 페이지 타입 5종 + 경로 규칙
- `.claude/rules/00_overview.md` — 9개 도메인

SKILL 본문은 위 규칙의 **실행 체크리스트**다. 충돌 시 규칙 문서 우선.

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
9개 도메인 중 1차 선택:
`ai-llm` / `engineering` / `data-ml` / `finance` / `work` / `career` / `readings` / `journal` / `life`

- 경계 모호 시 `wiki/<domain>/index_<domain>.md` 상단 "범위/경계" 섹션 확인
- 판정 애매하면 `wiki/<domain>/_inbox/` 에 임시 배치 후 lint에서 재분류

### 4. source-summary 페이지 작성 (L2 · Wiki)
- 경로: `wiki/<domain>/sources/<slug>.md`
- `type: source-summary`
- frontmatter 필수 (`10_frontmatter.md`): `id`(=파일명) / `title` / `domain` / `type` / `status: active` / `tags` / `sources` (원본 경로·URL 리스트) / `created` / `updated`
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
  - 경로 규칙: `30_page_types.md`

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

- ❌ `raw/` 원본 수정
- ❌ `99.attached/` 경로 변경 (기존 wikilink 호환)
- ❌ log 접두사 포맷 변형
- ❌ 확정되지 않은 도메인의 `sources/`에 저장 (애매하면 `_inbox/`)
- ❌ 한 ingest에서 10개 초과 페이지 **조용히** 수정 (반드시 보고)

## Verification checklist (ingest 완료 후 자기 점검)

- [ ] raw/ 에 원본 배치됨 (또는 URL 메타 생성)
- [ ] `wiki/<domain>/sources/<slug>.md` 생성, frontmatter 필수 필드 완비
- [ ] 언급된 entity/concept 페이지 모두 업데이트 또는 신규 생성
- [ ] 양방향 wikilink 연결
- [ ] `index_<domain>.md` 에 신규 페이지 등재
- [ ] `log_<domain>.md` 에 규정 포맷으로 append
- [ ] 크로스 도메인 영향 확인됨
