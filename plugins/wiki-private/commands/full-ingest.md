---
description: 대량 소스를 배치로 ingest합니다. "소스 배치 투입", "full ingest", "raw에 넣은 것들 전부 처리" 등의 요청 시 사용합니다.
---

# Full Ingest 워크플로우

**raw/ 하위에 추가된 미처리 소스를 일괄 ingest합니다.**

---

## Phase A: 소스 스캔

### A-1. 미처리 소스 식별

`raw/` 하위 전체를 스캔하여 아직 `wiki/*/sources/` 에 대응 source-summary가 없는 파일/URL을 목록화합니다.

```
Use Bash tool:
- command: "find raw/ -type f -name '*.md' -o -name '*.pdf' | sort"
```

```
Use Glob tool:
- pattern: "wiki/*/sources/*.md"
```

두 목록을 대조하여 미처리 소스 리스트를 생성합니다.

### A-2. 도메인 사전 분류

미처리 소스를 도메인별로 그룹핑합니다:
- 파일 경로, 제목, 내용 스캔 기반
- 판정 애매한 소스는 `_inbox` 그룹으로 분류

### A-3. 배치 분할

- **배치당 최대 10편** (ingest 규칙)
- 도메인별로 그룹 → 같은 도메인 소스끼리 배치 구성
- 배치 계획을 사용자에게 제시하고 승인 받기

---

## Phase B: 순차 Ingest

각 배치에 대해 순차적으로 `wiki-ingest` skill을 호출합니다.

```
Use Skill tool (각 소스마다):
- skill: "wiki-private:wiki-ingest"
- args: "<소스 경로 또는 URL>"
```

**배치 간 규칙:**
- 한 배치 완료 후 다음 배치 진행
- 배치 완료 시 중간 보고 (처리된 소스 수, 생성/수정된 페이지 수)
- 에러 발생 시 해당 소스 건너뛰고 보고

---

## Phase C: 정합성 점검

모든 배치 완료 후:

```
Use Skill tool:
- skill: "wiki-private:wiki-lint"
- args: "<가장 많이 영향받은 도메인>"
```

---

## 보고

최종 보고:
- 총 처리 소스 수
- 도메인별 신규/수정 페이지 수
- 건너뛴 소스 (에러)
- lint 결과 요약

---

## 주의사항

- Phase A → B → C 순서 고정
- 배치 분할 계획은 반드시 사용자 승인 후 진행
- 한 세션에 50편 초과 시 세션 분할 권장
