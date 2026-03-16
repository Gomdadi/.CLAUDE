---
name: setup-frontend
description: docs/tech-stack.md의 프론트엔드 섹션을 읽어 프론트엔드 개발 환경을 세팅한다. 프론트엔드 프로젝트 초기 세팅, 의존성 설치, 설정 파일 생성이 필요할 때 사용한다.
---

`./docs/tech-stack.md`의 프론트엔드 섹션을 읽어 **프론트엔드** 개발 환경을 세팅한다.

## 수행할 작업

1. **기술 스택 파악**: 프론트엔드 섹션에서 프레임워크, 언어, 스타일링, 상태 관리 등을 파악한다.

2. **프로젝트 구조 생성**: 프론트엔드 스택에 맞는 설정 파일을 생성한다.
   - 패키지 매니저 설정 파일 (package.json)
   - 린터/포매터 설정 (eslint, prettier 등)
   - TypeScript를 포함하면 tsconfig.json 생성
   - Git 사용을 전제로 .gitignore 생성
   - `.claudeignore` 생성: 빌드 결과물(dist/, build/, .next/, out/), 의존성(node_modules/), 민감한 파일(*.env), 대용량 파일(*.log), IDE 설정(.idea/) 제외
   - Docker를 포함하면 Dockerfile, docker-compose.yml, .dockerignore 생성
   - 프레임워크에 맞는 진입점 파일 생성 (React면 `src/main.tsx`, `src/App.tsx` 등)

3. **의존성 설치**: 프론트엔드 스택에 필요한 패키지를 설치한다.

4. **완료 보고**: 생성된 파일 목록과 다음 단계(실행 방법 등)를 간결하게 안내한다.

## 주의사항
- 모든 파일은 `./frontend/` 디렉토리 하위에 생성한다.
- 이미 존재하는 파일은 덮어쓰기 전에 확인한다.
- 모든 설명과 응답은 한국어로 작성한다.
