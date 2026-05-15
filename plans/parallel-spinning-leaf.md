# Skills 분할 기준 설계 플랜 (test / implement 분리)

## Context

이 프로젝트(NestJS + Prisma + BullMQ + Anthropic SDK)에서 각 레이어/컴포넌트별로
`test` / `implement` slash command skill을 정의.

기준: 레이어를 기본 축으로, 외부 의존성이 근본적으로 다른 경우만 별도 카테고리로 분리.
각 카테고리를 **test skill** / **implement skill** 2개로 나눔 → 총 8개.

---

## 분기 기준 트리

```
새 파일 작성 시
├── HTTP 진입점인가?              → nestjs-controller-{test,implment}
├── BullMQ @Processor인가?        → nestjs-worker-{test,implment}
├── Anthropic SDK를 직접 쓰는가?  → claude-service-{test,implment}
└── 나머지 Service                → nestjs-service-{test,implment}
```

---

## 최종 Skill 구성: 8개

| # | Skill 이름 | 역할 | 대상 파일 |
|---|-----------|------|----------|
| 1 | `claude-service-test` | spec 파일 작성 | `phase*.service.spec.ts`, `claude-agent.service.spec.ts` |
| 2 | `claude-service-implement` | 구현 파일 작성 | `phase*.service.ts`, `claude-agent.service.ts` |
| 3 | `nestjs-service-test` | spec 파일 작성 | `pipeline.service.spec.ts` 등 |
| 4 | `nestjs-service-implement` | 구현 파일 작성 | `pipeline.service.ts` 등 |
| 5 | `nestjs-worker-test` | spec 파일 작성 | `pipeline.worker.spec.ts` |
| 6 | `nestjs-worker-implement` | 구현 파일 작성 | `pipeline.worker.ts` |
| 7 | `nestjs-controller-test` | spec 파일 작성 | `pipeline.controller.spec.ts` |
| 8 | `nestjs-controller-implement` | 구현 파일 작성 | `pipeline.controller.ts` |

---

## 각 Skill 상세 내용

### 1. `claude-service-test`
```
핵심 지식:
- jest.mock('@anthropic-ai/sdk') + MockedClass<typeof Anthropic>
- mockClaudeAgent.runAgentLoop.mockImplementation(simulateAgentLoop) 패턴
- simulateAgentLoop: onToolCall 콜백을 순서대로 직접 호출해 agent loop 시뮬레이션
- 검증 포인트:
    - onToolCall 호출 순서 (toHaveBeenNthCalledWith)
    - 불완전 루프(일부 tool만 호출) → rejects.toThrow
    - API 실패 → rejects.toThrow
    - DB 저장 안 함 (not.toHaveBeenCalled) on error
- fs.readFileSync mock (prompts/*.md 로딩)
```

### 2. `claude-service-implement`
```
핵심 지식:
- ClaudeAgentService를 constructor DI로 주입
- runAgentLoop(options) 호출: systemPrompt, tools, onToolCall 콜백 구조
- tool result 수집: onToolCall 콜백 내에서 지역 변수에 누적
- 루프 완료 후 모든 필수 결과 존재 확인 → PrismaService로 저장
- 프롬프트는 prompts/*.md 파일로 분리, fs.readFileSync로 로딩
- execFileSync로 외부 script 호출 시 try/catch 처리
```

### 3. `nestjs-service-test`
```
핵심 지식:
- Prisma mock 객체: { modelName: { create: jest.fn(), findFirst: jest.fn(), update: jest.fn() } }
- BullMQ Queue mock: { add: jest.fn() }
- Test.createTestingModule providers 배열 구성
- jest.clearAllMocks() + beforeEach 재설정 패턴
- 검증: mockReturnValue/mockResolvedValue 설정 → 결과 assert
- null 반환(레코드 없음) → rejects.toThrow 케이스
```

### 4. `nestjs-service-implement`
```
핵심 지식:
- @Injectable() + constructor(private readonly ...) DI
- PrismaService: @Global 모듈이므로 별도 providers 없이 주입 가능
- BullMQ Queue: @InjectQueue(QUEUE_NAME) 데코레이터
- 비즈니스 로직 → Prisma 저장 → Queue job 추가 순서
- 상수(QUEUE_NAME 등)는 *.constants.ts 파일로 분리
```

### 5. `nestjs-worker-test`
```
핵심 지식:
- @Processor 데코레이터 때문에 실제 BullMQ 연결 없이 테스트하려면
  WorkerHost를 직접 instantiate하거나 TestingModule + mock Queue 사용
- Job mock 객체: { id: 'job-1', data: { ... }, name: 'job-name' } as Job
- process() 메서드 정상 케이스: job 처리 후 반환값 검증
- process() 실패 케이스: 내부 의존성 throw → process()도 rejects
- 내부에서 호출하는 Phase service들은 mock으로 제공
```

### 6. `nestjs-worker-implement`
```
핵심 지식:
- extends WorkerHost + @Processor(QUEUE_NAME) 패턴
- override async process(job: Job): Promise<void>
- job.name으로 분기 (switch/if) → 각 job type별 처리
- job.data는 타입 단언(as) 또는 zod 파싱으로 안전하게 접근
- 에러 throw 시 BullMQ가 자동 retry (설정에 따라)
```

### 7. `nestjs-controller-test`
```
핵심 지식:
- Test.createTestingModule + app = moduleRef.createNestApplication()
- supertest: request(app.getHttpServer()).post('/v1/...').send(body)
- 성공: expect(201) 또는 expect(200)
- 실패: NotFoundException → expect(404), 등
- Guard bypass: { provide: AuthGuard, useValue: { canActivate: () => true } }
- SSE 엔드포인트는 supertest로 테스트 어려움 → 별도 처리 또는 skip 언급
```

### 8. `nestjs-controller-implement`
```
핵심 지식:
- @Controller('v1/resource') + @Get()/@Post()/@Param()/@Body() 패턴
- DTO: class-validator (@IsString, @IsUUID 등) + class-transformer
- SSE: @Sse() + return new Observable<MessageEvent>(...)
- @Req() / @Res() 사용 시 NestJS 표준 응답 인터셉터 비활성화 주의
- 예외: throw new NotFoundException() 등 NestJS 내장 HttpException 사용
```

---

## Skill 파일 위치

```
.claude/skills/
├── claude-service-test/
│   └── SKILL.md
├── claude-service-implement/
│   └── SKILL.md
├── nestjs-service-test/
│   └── SKILL.md
├── nestjs-service-implement/
│   └── SKILL.md
├── nestjs-worker-test/
│   └── SKILL.md
├── nestjs-worker-implement/
│   └── SKILL.md
├── nestjs-controller-test/
│   └── SKILL.md
└── nestjs-controller-implement/
    └── SKILL.md
```

---

## 구현 순서

1. `claude-service-test` + `claude-service-implement` (기존 spec 파일 패턴 가장 풍부)
2. `nestjs-service-test` + `nestjs-service-implement`
3. `nestjs-worker-test` + `nestjs-worker-implement`
4. `nestjs-controller-test` + `nestjs-controller-implement`

각 skill 작성 시 `skill-creator` slash command 활용.
