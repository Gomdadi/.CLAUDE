---
name: unit-test
description: Unit test 코드 작성 전문 skill. Java(JUnit 5 + Mockito), TypeScript/JavaScript(Jest), Python(pytest) 프로젝트에서 unit test를 작성하거나, 기존 코드에 테스트를 추가하거나, TDD 방식으로 테스트를 먼저 작성할 때 사용. 사용자가 "테스트 작성", "unit test", "테스트 코드 추가", "TDD", "mock", "Jest", "JUnit", "pytest" 등을 언급할 때 트리거.
---

# Unit Test 작성 가이드

## 언어별 Reference 선택

대상 언어/프레임워크를 파악한 뒤 해당 reference를 읽는다:

- **Java** (Spring Boot, Maven/Gradle) → `references/java.md`
- **TypeScript / JavaScript** (Node.js, React, NestJS 등) → `references/typescript.md`
- **Python** (FastAPI, Django, Flask 등) → `references/python.md`

언어가 명확하지 않으면 파일 확장자, import 문, 프로젝트 구조를 보고 판단한다.

## 공통 워크플로우

1. **대상 코드 파악** — 테스트할 함수/클래스/메서드를 읽는다
2. **Reference 로드** — 위 표에서 해당 언어 reference를 읽는다
3. **테스트 케이스 설계**
   - Happy path (정상 동작)
   - Edge case (경계값, 빈 입력, null/undefined)
   - Error case (예외 발생, 에러 핸들링)
4. **테스트 코드 작성** — reference의 패턴을 따른다
5. **검증** — 테스트가 실제로 실행 가능한지 확인 (파일 경로, import, 의존성)

## TDD 모드

사용자가 TDD를 요청하면:
1. 구현 코드 없이 인터페이스/시그니처만 파악한다
2. 실패하는 테스트를 먼저 작성한다 (Red)
3. 테스트를 통과시킬 최소한의 구현 방향을 안내한다 (Green)

## 주의사항

- 테스트는 서로 독립적이어야 한다 (각 테스트가 다른 테스트 상태에 의존하지 않음)
- 외부 의존성(DB, HTTP, 파일 I/O)은 mock/stub으로 격리한다
- 테스트 이름은 "무엇을 테스트하는지"가 명확히 드러나야 한다
- 하나의 테스트는 하나의 동작만 검증한다
