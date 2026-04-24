# General rules

- 이해가지 않거나 변경이 필요한 사항은 반드시 사용자의 피드백을 기반으로 진행할 것
- 사용자가 작업 디렉토리를 지정하지 않은 경우 반드시 피드백 요청 할 것 
- 변경 사항은 반드시 .serena/memories/에 반영할 것 
- 아래 내용에 해당하는 코드베이스 작업의 경우 반드시 LSP Tool을 사용할 것 

    | 작업 | 설명 |
    |------|------|
    | goToDefinition | 심볼이 정의된 위치 찾기 |
    | findReferences | 심볼의 모든 참조 찾기 |
    | hover | 호버 정보 (문서, 타입 정보) 조회 |
    | documentSymbol | 문서 내 모든 심볼 조회 |
    | workspaceSymbol | 워크스페이스 전체에서 심볼 검색 |
    | goToImplementation | 인터페이스/추상 메서드의 구현체 찾기 |
    | prepareCallHierarchy | 호출 계층 항목 준비 |
    | incomingCalls | 해당 함수를 호출하는 모든 함수 찾기 |
    | outgoingCalls | 해당 함수가 호출하는 모든 함수 찾기 |