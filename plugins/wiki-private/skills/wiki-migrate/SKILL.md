---
name: wiki-migrate
description: Use when the user says "레거시 이관해줘", "migrate <domain>", "PARA 이관", "_legacy 정리", or wants to convert old PARA-structure notes into the LLM-wiki domain structure. Executes the Karpathy LLM-wiki migration workflow — reads _legacy/ notes, determines type/domain, normalizes frontmatter, creates wiki pages, and records in log_<domain>.md.
---

# Wiki Migrate Skill

Karpathy LLM-wiki 패턴(https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)의 **레거시 이관 워크플로우**를 실행한다.

> "The goal of migration is not speed — it is accuracy. One misclassified note is harder to fix than one unclassified note." — Operating principle

기존 PARA 구조(`_legacy/`)의 노트들을 도메인 단위로 점진 이관한다. 현재 총 약 1,865개 노트가 `_legacy/` 하위에 보존되어 있다.

## When to use

- 사용자가 "_legacy 정리해줘", "레거시 이관해줘", "PARA 이관" 등 요청
- `migrate <domain>` 명령으로 특정 도메인 이관 시작
- `_legacy/` 내 노트를 wiki 구조로 변환해야 할 때

---

## Vault Overview (인라인)

### 3계층 구조

| 계층 | 경로 | 소유 | 역할 |
|------|------|------|------|
| Raw Sources | `raw/` | 사용자 (불변) | 원본 문서 (논문·기사·PDF·스크린샷). LLM은 읽기만 |
| Wiki | `wiki/<domain>/` | LLM | 도메인별 엔티티·개념·소스 요약·비교 페이지 |
| Schema | `wiki/<domain>/index_<domain>.md` | 공동 | 도메인 특이사항 + 카탈로그 |

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

도메인 간 경계가 모호하면 해당 도메인의 `index_<domain>.md` 상단 "범위/경계" 섹션을 우선 참고한다. 판단이 끝까지 애매하면 `wiki/<domain>/_inbox/`에 두고 lint에서 재분류.

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

1. **공통 규칙 < 도메인 규칙**: 도메인 예외는 `index_<domain>.md`가 우선
2. **루트 공용 파일 없음**: index·log는 도메인별로만 존재
3. **99.attached/ 경로 불변**: 기존 wikilink 호환 위해 이동 금지
4. **프로젝트는 태그로**: 별도 폴더 없이 `project/<slug>` 태그 사용
5. **크로스 도메인 — 정본 단일 원칙**: 동일 entity/concept는 가장 관련 깊은 도메인에 정본 1개만 생성. 타 도메인에서는 `[[wikilink]]`로 참조

---

## 이관 순서 (우선순위)

1. `ai-llm` (~450 노트, 파일럿) — 주로 `_legacy/2.Resource/0.LLM`
2. `readings` (~330) — `_legacy/2.Resource/2.읽은것들`
3. `engineering` (~300) — `_legacy/2.Resource/{apps, python, git, docker, ...}`
4. `finance` (~120) — `_legacy/2.Resource/Economics` + `_legacy/0.Project/{auto-stock, auto-btc, ...}`
5. `work` (~280) — `_legacy/0.Project/CUBIG*` + `_legacy/3.Archive/{wavus, Electrip, PoC_*, Dify, CUBIG-LLMCapsule}`
6. `career` (~70)
7. `journal` (~130)
8. `life` (~40)
9. `data-ml` (~40)

---

## 페이지 타입 (인라인)

`wiki/<domain>/` 하위 모든 노트는 5가지 타입 중 하나. frontmatter `type:` 필드로 지정.

### 5가지 타입

#### 1. `entity` — 실체
- 대상: 인물, 제품, 도구, 회사, 프로젝트, 장소 등 고유 식별 가능한 대상
- 경로: `wiki/<domain>/entities/<slug>.md`
- 구성: 정의 / 주요 사실 / 관련 링크 / 시간순 이벤트
- 예: `entities/langchain.md`, `entities/cubig-syntitan.md`

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

#### 4. `comparison` — 비교·종합
- 대상: 여러 소스·엔티티·개념을 가로지르는 비교·합성 페이지
- 경로: `wiki/<domain>/` 루트 또는 `wiki/<domain>/entities|concepts/` (주제에 따라)
- 구성: 대상 목록 / 비교 테이블 / 상황별 추천 / 근거 링크

#### 5. `note` — 미분류
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

---

## Frontmatter 표준 (인라인)

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

### Frontmatter 규칙

1. `id`는 파일명(확장자 제외)과 반드시 일치
2. `updated`는 수정 시마다 갱신 (LLM이 자동 갱신)
3. `status: outdated`는 더 새로운 소스로 교체 필요함을 표시
4. `sources`가 비어있는 concept 페이지는 합성된 지식임을 의미
5. 한국어 제목도 허용하나, `id`(파일명)는 영문 kebab-case로 통일

### 레거시 호환 매핑 (이관 시 적용)

| 기존 필드 | 변환 규칙 |
|-----------|-----------|
| `author: hunhun` | 유지 (선택 필드) |
| `type: "[[📌 학습노트]]"` 등 | 새로운 `type` 값(concept/source-summary 등)으로 재지정 |
| `topic: Agent` | `tags: [tech/agent]` 로 변환 |
| `source: URL` | `sources: [URL]` (단수 → 리스트화) |
| `created:` | 유지 |
| `domain`, `status`, `updated` | 없으면 신규 부여 |

---

## 태그 네임스페이스 (인라인)

모든 태그는 **네임스페이스/슬러그** 형식.

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

### 태그 작성 규칙

1. 태그 값은 kebab-case 영문 (한글 금지)
2. frontmatter `tags:` 리스트에 모두 기재 (본문 `#tag` 사용 지양)
3. 페이지당 태그 3~7개 권장. 10개 초과 시 분할 검토
4. 프로젝트 태그는 페이지당 최대 2개 (주 프로젝트 + 관련 프로젝트)

---

## Procedure — 노트별 이관 절차

각 `_legacy/` 노트에 대해 아래 8단계를 순서대로 실행한다.

### Step 1. 타입 판정

판별 순서를 엄수한다:

1. 원본 1건에 대응하는 요약? → `source-summary`
2. 고유 대상(인물·도구·제품·프로젝트)? → `entity`
3. 2개 이상 대상을 가로지름? → `comparison`
4. 추상 개념? → `concept`
5. 위 어디에도 명확히 맞지 않으면 → `note` (경로: `_inbox/`)

### Step 2. 경로 결정

타입별 목적지 경로:

| 타입 | 경로 |
|------|------|
| `entity` | `wiki/<domain>/entities/<slug>.md` |
| `concept` | `wiki/<domain>/concepts/<slug>.md` |
| `source-summary` | `wiki/<domain>/sources/<slug>.md` |
| `comparison` | `wiki/<domain>/<slug>.md` |
| `note` | `wiki/<domain>/_inbox/<slug>.md` |

- `<slug>` 는 영문 kebab-case. 기존 파일명을 참고하되 필요 시 정제
- 도메인 경계 모호 시 해당 도메인의 `index_<domain>.md` 범위/경계 섹션 확인 후 결정

### Step 3. 원본 분리 (source-summary인 경우만)

- 본문 내 외부 URL → `raw/articles/<slug>.md` 생성 (메타데이터만: URL, 수집일, 제목)
- 첨부 PDF → `raw/papers/` 또는 `raw/docs/`로 이동
- 이미 `99.attached/`에 있는 이미지 → **이동 금지**, 경로 유지

### Step 4. Frontmatter 정규화

레거시 호환 매핑을 적용하여 필수 필드를 완성한다:

```yaml
---
id: <slug>                     # 파일명과 동일
title: <원본 제목 또는 정제된 제목>
domain: <판정된 도메인>
type: <판정된 타입>
status: draft                  # 이관 직후 기본값; 검토 후 active로 승격
tags: [<변환된 태그들>]
sources: [<URL 또는 raw 경로>]
created: <원본 created 유지 또는 오늘 날짜>
updated: <오늘 날짜>
author: hunhun                 # 원본에 있으면 유지
---
```

레거시 필드 변환:
- `type: "[[📌 학습노트]]"` 등 → 새 type 값으로 재지정 (Step 1 판정 결과 사용)
- `topic: <값>` → `tags: [tech/<값의 kebab-case>]`
- `source: <URL>` → `sources: [<URL>]`
- `domain`, `status`, `updated` 없으면 신규 부여

### Step 5. Wikilink 재연결

- 다른 레거시 노트로 가는 링크: 이미 이관된 경우 새 경로로 갱신. 미이관 대상은 일시적으로 `_legacy/...` 경로 유지
- `99.attached/...` 이미지·첨부 링크: **변경 금지**
- 전역 find/replace로 wikilink 일괄 치환 금지 (확인 없이)

### Step 6. 교차 참조 생성

- 이관 대상이 다른 도메인 페이지와 관련되면 wikilink 추가
- 크로스 도메인 영향 발생 시 해당 도메인의 `log_<그 도메인>.md`에도 이벤트 기록

### Step 7. log_<domain>.md 기록 — **포맷 엄수**

```markdown
## [YYYY-MM-DD HH:MM] migrate | <legacy path> → <wiki path>
type: <판정된 타입>
tags: [<태그 목록>]
notes: <특이사항·결정 1~3줄 (없으면 생략)>
```

- 접두사 `## [YYYY-MM-DD HH:MM] migrate | ` **절대 변형 금지**
- 시간은 현재 로컬 시각

### Step 8. index_<domain>.md 갱신

- 해당 도메인의 카테고리 섹션(엔티티/개념/소스 요약 등)에 링크 + 한 줄 요약 추가
- 신규 파일 생성 시마다 갱신 (배치 종료 후 한꺼번에 갱신해도 무방)

**원본은 `_legacy/`에 그대로 둔다** — Phase 4 전 삭제 금지.

---

## 배치 규칙

- 세션당 30~50 노트 이관
- 이관 후 해당 도메인 sanity check (고아·링크 깨짐) 수행
- 도메인 전체 이관 완료 시 `log_<domain>.md`에 완료 마커 기록:

```markdown
## [YYYY-MM-DD] migrate_complete | <domain>
```

- **한 세션에 2개 도메인 동시 이관 금지**

---

## 이관 완료 후 정리 (raw/legacy-notes)

`raw/legacy-notes/<domain>/`은 `_legacy/`에서 복사한 이관 스테이징 폴더다. 원본이 아니므로 아래 조건 충족 시 삭제 가능하다.

### 삭제 조건 (모두 충족 시)

1. `log_<domain>.md`에 `migrate_complete` 마커가 있거나 로그에서 90%+ 이관 완료 확인됨
2. `_legacy/`에 원본이 온전히 보존되어 있음

### 삭제 절차

1. `rm -rf raw/legacy-notes/<domain>/`
2. `log_<domain>.md`에 기록:
   ```markdown
   ## [YYYY-MM-DD HH:MM] cleanup | raw/legacy-notes/<domain> 삭제
   ```
3. `raw/legacy-notes/`가 완전히 비면 디렉토리 자체도 삭제

---

## Hard rules (금지)

- `99.attached/` 경로 변경 (기존 wikilink 호환 파괴)
- `_legacy/` 삭제 (Phase 4 전까지 백업 역할)
- 전역 find/replace로 wikilink 일괄 치환 (확인 없이)
- 한 세션에 2개 도메인 동시 이관
- 이관 미완료 도메인의 `raw/legacy-notes/<domain>/` 삭제
- log 접두사 포맷 변형
- 타입 판별 순서 무시 (순서대로 판별해야 함)
- `status: active` 즉시 부여 (이관 직후 기본값은 `draft`)

---

## 완료 기준 (도메인당)

- [ ] 해당 `_legacy/` 경로 노트 모두 이관 또는 `_inbox/` 배치
- [ ] `index_<domain>.md` 카탈로그 최신화
- [ ] 고아 페이지 < 전체의 5%
- [ ] `log_<domain>.md`에 `migrate_complete` 이벤트 기록

---

## Verification checklist (이관 세션 완료 후 자기 점검)

- [ ] 대상 도메인이 이관 순서상 적절한가 확인됨
- [ ] 각 노트에 타입 판별 순서(1→5) 적용됨
- [ ] 레거시 frontmatter 호환 매핑 모두 적용됨
- [ ] `topic:` → `tags:`, `source:` → `sources:` 변환 완료
- [ ] `99.attached/` 경로 변경 없음
- [ ] `_legacy/` 원본 유지됨
- [ ] 양방향 wikilink 재연결됨 (이관된 노트 간)
- [ ] `log_<domain>.md`에 이관된 노트 수만큼 migrate 이벤트 append됨
- [ ] `index_<domain>.md`에 신규 페이지 등재됨
- [ ] 크로스 도메인 영향 확인됨
- [ ] 도메인 전체 이관 완료 시 `migrate_complete` 마커 기록됨
