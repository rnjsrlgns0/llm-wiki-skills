# Commit Rules

## 커밋 메시지 포맷

### Conventional Commits 형식 사용
- `feat:` - 새 기능
- `fix:` - 버그 수정
- `refactor:` - 리팩토링
- `docs:` - 문서
- `test:` - 테스트
- `chore:` - 빌드/설정

## ⚠️ 중요: 커밋 메시지 제약사항

**커밋 메시지에 다음 내용을 포함하지 마세요:**
- ❌ "🤖 Generated with Claude Code" 또는 유사한 AI 생성 표시
- ❌ "Co-Authored-By: Claude [MODEL]" 또는 AI 공동 작성자 표시
- ✅ 일반적인 Conventional Commits 형식만 사용

## 올바른 커밋 메시지 예제

```bash
git commit -m "feat: add worktree detection to branch-manager

- Implement automatic worktree detection
- Add session-based worktree creation
- Support concurrent Claude Code sessions"
```

## 잘못된 커밋 메시지 예제 (사용 금지)

```bash
# ❌ 잘못된 예 - AI 생성 표시 포함
git commit -m "feat: add worktree detection

🤖 Generated with Claude Code
Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

## 데이터 컬럼의 의미나 값을 사용자에게 물어보는 커밋 메시지 작성 규칙

- 데이터 분석 또는 전처리 중 컬럼명/컬럼 의미가 불분명하거나 예제 데이터가 부족한 경우,
  Conventional Commits 포맷(주로 `chore:` 또는 `docs:`)을 사용해 데이터 명세 요청 커밋 메시지를 작성할 수 있다.
- 이때 커밋 본문 또는 Pull Request 코멘트 등에 **테이블 형태 예시를 포함**해, 해당 컬럼에 빈칸을 뚫어놓고 사용자가 쉽게 이해하고 채울 수 있도록 유도한다.
- 예시 테이블에는 어떤 값이 필요한지(예: 의미, 단위, 값 유형 등)를 명확히 기재한다.

### 예시

```
chore: request clarification for ambiguous columns in dataset

- Some column names (e.g., 'cnt', 'val1') are unclear in meaning or unit.
- Please fill in the blanks (**?**) for the following columns for documentation and analysis accuracy.

| 컬럼명   | 의미/정의          | 단위/형태 | 예시 값 | 비고    |
|----------|--------------------|-----------|---------|---------|
| cnt      | ?                  | ?         | 528     |         |
| val1     | ?                  | ?         | 19.4    |         |

→ 각 컬럼의 의미, 단위, 나타내는 값 등을 채워주세요.
```

- 모든 질의는 예시처럼 **테이블을 활용하여 답변을 구체적으로 요청**해야 하며, **불명확한 컬럼은 반드시 '?'로 명확히 표시**한다.