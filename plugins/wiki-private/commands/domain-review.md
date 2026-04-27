---
description: 도메인 전체 점검을 수행합니다. "도메인 리뷰", "위키 점검", "domain review", "<domain> 건강검진" 등의 요청 시 사용합니다.
---

# Domain Review 워크플로우

**지정 도메인의 위키 건강 상태를 종합 점검합니다.**

---

## Phase A: 대상 확인

### A-1. 도메인 선택

사용자가 도메인을 지정하지 않은 경우:

1. 각 도메인의 `log_<domain>.md` 최근 lint 기록 확인
2. 가장 오래된 lint 또는 lint 기록 없는 도메인을 후보로 제시
3. 사용자에게 대상 도메인 확인

### A-2. 현황 스캔

```
Use Glob tool:
- pattern: "wiki/<domain>/**/*.md"
```

도메인 내 총 페이지 수, 타입별 분포를 요약합니다.

---

## Phase B: Lint 실행

```
Use Skill tool:
- skill: "wiki-private:wiki-lint"
- args: "<domain>"
```

8개 체크리스트 전체 실행:
1. 고아 페이지
2. 모순
3. Outdated
4. 누락 교차참조
5. 데이터 갭
6. 인덱스 정합성
7. _inbox 적체
8. Frontmatter 유효성

---

## Phase C: 조치 실행

### C-1. 자동 수정

lint 결과 중 자동 수정 가능 항목 즉시 적용:
- 누락 wikilink 추가
- frontmatter 필드 보완
- index 정합성 복구

### C-2. 사용자 결정 요청

수동 결정 필요 항목 목록 제시:
- 고아 페이지 처리 방향 (archive / delete / link)
- 모순 해결 방향
- _inbox 재분류

---

## 보고

Lint Report 형식으로 최종 보고:

```
# Lint Report — <domain> ([YYYY-MM-DD HH:MM])

## Summary
| Category         | Count | Auto-fixed | Pending |
|------------------|-------|------------|---------|
| ...              |       |            |         |

## Auto-fixed
- ...

## Needs your decision
- ...

## Suggestions (next ingest / query)
- ...
```

---

## 주의사항

- 한 번에 한 도메인만 점검
- 사용자 승인 없는 페이지 삭제 금지
- 자동 수정 사항도 변경 목록 보고 필수
