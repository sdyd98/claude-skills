# 하네스 설계 제안 (2026-04 내부 논의 기반)

- **작성일**: 2026-04-20
- **문서 성격**: **팀 내부 논의 산출물** — 정답·최종안이 아님
- **`harness-design-brief.md`와의 차이**:
  - 브리핑: 중립 레퍼런스 (공식 팩트만)
  - 이 문서: **특정 제안** (판단·선호 포함)
- **비교 시 유의**: 기존 사내 하네스가 이 제안과 다르다고 해서 반드시 열세는 아님. 팀 맥락·과거 경험이 반영된 의도적 설계일 수 있음.

---

## §1. 설계 원칙 4가지

| # | 원칙 | 근거 |
|--|--|--|
| **P1** | **축(요구사항) 먼저, 수단(확장점) 나중** | 브리핑 §2 7축 프레임 |
| **P2** | **기성 → 조합 → 자작** 순서 | `/ultrareview`·Jira MCP·Auto memory는 이미 존재 |
| **P3** | **결정적 자리엔 hook, AI 판단 자리엔 prompt/agent hook** | 공식이 AI를 hook에 한정적으로 지원 |
| **P4** | **플러그인화를 처음부터 염두** | Phase 3에서 사내 마켓플레이스 배포 |

---

## §2. 4계층 구조 (scope별)

```
Layer 0 — Managed Policy (IT 배포, 회피 불가)
  /etc/claude-code/CLAUDE.md + managed-settings.json
  • 보안 정책 (외부 전송 금지 경로)
  • 강제 데이터 정책, 인증 조직 잠금

Layer 1 — Project (git 공유, 팀 표준)
  ./CLAUDE.md  (≤200줄, 핵심 규약만)
  ./.claude/
   ├── rules/        (도메인·경로 스코프 규칙)
   ├── skills/       (Track A/B 워크플로)
   ├── agents/       (plan-clarifier, code-reviewer 등)
   ├── settings.json (hooks, permissions, MCP)
   └── CLAUDE.md     (선택, 프로젝트 상세)

Layer 2 — User (~/.claude/, 개인 취향)
  ~/.claude/CLAUDE.md, skills/, rules/

Layer 3 — Local (개인·프로젝트, gitignored)
  ./CLAUDE.local.md, ./.claude/settings.local.json
```

**핵심**: 모든 팀원이 보는 표준은 **Layer 1**. 개인차는 Layer 2/3. 보안 강제는 Layer 0.

---

## §3. Track A/B × 7축 × 공식 기능 매핑

| 7축 | Track A (오케스트레이션) | Track B (비즈니스 코드, 머지급) |
|--|--|--|
| ① 프롬프팅 | `/start-feature`·`/fix-bug`·`/start-request` Skills | `code-reviewer`·`test-writer` Subagents + C++/FlatBuffers rules |
| ② 컨텍스트 | Jira MCP, Jenkins MCP | `.claude/rules/` with `paths:` + 사내 코드베이스 MCP (Phase 2) |
| ③ 툴 호출 | `!`bash injection (빌드·git) | MSBuild Skill, Codex CLI via agent hook |
| ④ 상태·메모리 | Auto memory (팀 공유 디렉토리로 redirect) | CLAUDE.md + rules, Checkpointing |
| ⑤ 종료·제어 | Plan Mode, Auto mode | Stop hook (빌드·테스트), Skill의 `context: fork` |
| ⑥ 안전 가드 | permissions allow/deny, Sandboxing | PreToolUse hook (format·lint), prompt hook (위험 판단) |
| ⑦ 관측성 | statusline, costs dashboard | OpenTelemetry (Agent SDK), Jenkins 로그 |

---

## §4. 디렉토리 레이아웃 (Phase 1 MVP 기준 제안)

```
<game-server-repo>/
├── CLAUDE.md                           # ≤200줄, 팀 규약 핵심
│
├── .claude/
│   ├── rules/                          # 도메인·경로 스코프 규칙
│   │   ├── cpp-style.md                #   paths: ["**/*.{cpp,h,hpp}"]
│   │   ├── flatbuffers.md              #   paths: ["**/*.fbs"]
│   │   ├── packet-codegen.md           #   사내 codegen 사용법
│   │   ├── db-codegen.md
│   │   └── xls-codegen.md
│   │
│   ├── skills/                         # 워크플로 Skills
│   │   ├── write-tests/SKILL.md        # Track B (최우선, 테스트 0 대응)
│   │   ├── fix-bug/SKILL.md            # Track A
│   │   ├── merge-gate/SKILL.md         # Track B
│   │   └── codex-review/SKILL.md       # Track B (Codex 교차 검증)
│   │
│   ├── agents/                         # Subagents
│   │   ├── test-writer.md              # 대량 테스트 생성 특화
│   │   ├── plan-clarifier.md           # 명확화 전담
│   │   └── code-reviewer.md            # C++/사내 툴 특화 리뷰
│   │
│   └── settings.json                   # hooks, permissions, MCP
│
├── .claude/settings.local.json         # 개인 overrides (gitignored)
└── CLAUDE.local.md                     # 개인 메모 (gitignored)
```

**Phase 2 추가**: 자체 플러그인, 자작 MCP, Agent Teams 정의, `/start-feature` Skill.

---

## §5. Skills / Subagents 구성 스펙 (frontmatter 예시)

### `/write-tests` Skill (Phase 1 최우선)
```yaml
---
name: write-tests
description: 대상 파일·함수·클래스에 단위 테스트 대량 생성. 테스트 0 상태에서 커버리지 확보용
argument-hint: "<file-path-or-symbol>"
effort: xhigh
allowed-tools: Read Grep Bash(msbuild *) Bash(ctest *) Edit Write
---
# 절차
1. test-writer 서브에이전트 호출 (context: fork)
2. 대상 분석: 공개 함수·클래스·엣지 케이스 식별
3. GoogleTest 컨벤션으로 테스트 생성
4. 빌드·실행 → 통과 확인
5. 통과율 리포트
```

### `code-reviewer` Subagent (Track B 품질 바)
```yaml
---
name: code-reviewer
description: C++/FlatBuffers/사내 툴 관점의 특화 리뷰. 일반 리뷰는 /ultrareview 기성에 맡김
tools: Read, Grep, Glob, Bash
model: opus
---
너는 C++ 게임서버 시니어 엔지니어. 도메인 특화 관점:
- MSVC UB, 동시성 함정 (TSan 부재)
- FlatBuffers 스키마 호환성
- 사내 codegen(패킷/DB/xls) 규약 위반
- "거의 머지 가능" 기준 머지 가능성
```

### `/merge-gate` Skill (Track B 품질 바 실행)
```yaml
---
name: merge-gate
description: 빌드·테스트·정적분석·Codex 교차검증 전체 수행
disable-model-invocation: true
allowed-tools: Bash(msbuild *) Bash(ctest *) Bash(clang-tidy *)
---
1. !`msbuild /p:Configuration=Release`
2. !`ctest --output-on-failure`
3. !`clang-tidy ...`
4. codex-review Skill 호출
5. auto memory 체크
```

---

## §6. Hooks 전략 매트릭스

| Event | Matcher | Handler Type | 목적 |
|--|--|--|--|
| `PostToolUse` | `Edit\|Write` (*.cpp, *.h) | `command` | clang-format 자동 |
| `PreToolUse` | `Bash` + `if: "Bash(git push *)"` | `command` | 머지 게이트 호출 |
| `Stop` | — | `command` | 빌드·테스트 자동, 실패 시 재개 |
| `PreToolUse` | 위험 작업 | **`prompt`** | LLM 위험 판단 |
| `PostToolUse` | `Edit` (중요 파일) | **`agent`** | 서브에이전트 영향 분석 |
| `SessionStart` | `resume` | `command` | Jira·Jenkins 상태 주입 |
| `PreCompact` | — | `command` | 핵심 상태 auto memory 저장 |

**포인트**: 결정적 체크는 `command`, **AI 판단이 의도적 필요한 자리**에는 `prompt`/`agent` hook.

---

## §7. MCP 구성 (Phase별)

### Phase 1 (기성 + 최소 자작)
- **기성**: Jira MCP, GitHub MCP, filesystem MCP
- **Jenkins MCP**: 기성 없으면 custom tool로 시작
- **🆕 최소 코드베이스 인덱스 MCP** (원래 Phase 2였으나 580만 줄 대응으로 Phase 1 필수)
  - ripgrep wrapper + ctags/clangd 심볼 조회 수준
  - TypeScript SDK로 구축

### Phase 2 (자작 고도화)
- 사내 코드베이스 인덱스 MCP 성능 개선
- FlatBuffers 스키마 MCP
- Codex 통합 MCP

---

## §8. Phase 1 MVP 최소 기능 (테스트 0 제약 반영 수정판)

### 반드시 (4주)
- [ ] 디렉토리 레이아웃 (§4)
- [ ] CLAUDE.md (≤200줄)
- [ ] `.claude/rules/` 5개 (C++, FlatBuffers, 사내 툴 3종)
- [ ] **`/write-tests` Skill [P0 최우선]** — 테스트 인프라 AI화
- [ ] **`test-writer` Subagent [P0]**
- [ ] `/fix-bug` Skill
- [ ] `/merge-gate` Skill
- [ ] `code-reviewer` Subagent
- [ ] `plan-clarifier` Subagent
- [ ] Hooks: PostToolUse clang-format, Stop 빌드·테스트
- [ ] MCP 기성 (Jira 등)
- [ ] 🆕 **최소 코드베이스 인덱스 MCP [P0, 580만 줄 대응]**
- [ ] Settings (permissions, **Auto mode default**)
- [ ] `autoMemoryDirectory` 팀 공유 경로

### 안 할 것 (Phase 2로)
- 🚫 자작 MCP 고도화
- 🚫 Plugin 패키징
- 🚫 Agent Teams
- 🚫 관측성 대시보드
- 🚫 Routines·Channels
- 🚫 `/start-feature` (테스트 인프라 확보 후 Phase 2)

---

## §9. 검증 방법 (브리핑 §5 기반 + 팀 특화)

### 3층 지표
- **층 A 정량** (자동): 토큰·비용·통과율·재시도
- **층 B 품질** (구성): **머지급 품질 Index**
  ```
  Index = Merge-readiness × 0.4
        + First-try pass rate × 0.3
        + (Review round ≤1.5 이면 1.0) × 0.2
        + 정적분석 경고 증가 없음 × 0.1
  ```
- **층 C 생산성**: Lead time, bug regression, AI-assisted share

### 5 방법론
1. Eval dataset (팀 6명 × 티켓 5개씩 = 30개)
2. Shadow/Canary (3+3명 2주 교체)
3. Regression suite (Jenkins/Routines 야간)
4. Golden task + LLM Judge
5. Production telemetry (analytics/costs/OTEL)

### 단계별 도입
- **Phase 1**: `/en/costs`, statusline, 빌드 통과율 집계, PR 라벨링
- **Phase 2**: eval-set 구축, 주간 품질 Index 리포트, LLM Judge
- **Phase 3**: A/B 캐너리, OTEL 대시보드, 팀 설문

---

## §10. 팀 제약 반영 특이점

### 테스트 0 → `/write-tests` 최우선
- 공식 Best Practices의 단일 최고 원칙: *"Give Claude a way to verify its work"*
- 테스트 없이는 "거의 머지 가능" 품질 바 측정 불가
- Phase 1 첫 가치 산출을 **테스트 코드 대량 생성**에 투입

### 580만 줄 → 코드베이스 인덱스 MCP Phase 1 필수
- 컨텍스트 윈도우(~200K 토큰)는 코드베이스 1~2%만 커버
- 원래 Phase 2였으나 앞당김
- 초기엔 ripgrep wrapper 수준으로 충분

### MSBuild `.vcxproj` 직접 → VS IDE 의존
- CMake 도입 고려 대상 아님 (기존 자산 유지)
- 테스트 프로젝트도 `.vcxproj` 방식으로
- vcpkg로 GoogleTest 설치

### 팀 GoogleTest 경험 0명 → 학습 포함
- Day 1에 1시간 튜토리얼 학습 포함
- AI가 테스트 작성·가이드 동시에 수행
- 공식 docs 3개만 필독 (Primer, VS 사용법, vcpkg)

### 보안 통과 + 팀 6명 모두 AI 사용
- 가장 큰 장애물 2개 해소됨
- 성공 확률 상단 시나리오 가능

### .lib 모듈 → GameServer.exe 단일
- **큰 이점**: .lib 단위 테스트 가능, 모놀리식 아님
- 테스트 격리 자연스러움

---

## §11. 비교 시 주의사항

- 이 제안은 **논의 산출물**이지 **정답이 아님**
- 기존 사내 하네스가 여기 제안과 다르다고 해서:
  - 반드시 "누락"이나 "미흡"은 아님
  - 팀 맥락·과거 경험이 반영된 **의도적 설계**일 수 있음
- 이 제안도 **팀 제약 정보에 기반한 추정**이 포함됨
- 비교 결과는 **토론 자료**이지 자동 실행 대상 아님

### 이 제안이 약한 영역 (솔직히)
- 장기 운영·유지보수 전략 (팀 이탈, Claude Code 브레이킹 변경 대응)
- 비용 모니터링 구체안
- 팀원별 활용도 편차 대응
- 실제 성능 벤치마크 지표 (게임서버 특유)

---

## §12. 요약

- **목표**: C++ 게임서버 6인 팀의 개발 생산성 가속 (완전 자동 아님)
- **구조**: 4계층 × Track A/B × 7축 매핑
- **Phase 1 최우선**: `/write-tests` + 코드베이스 인덱스 MCP (테스트 0 + 580만 줄 대응)
- **품질 바**: "거의 머지 가능" → **머지급 품질 Index**로 정량화
- **성공 조건**: 팀 버인·보안·기대치 합의·3~6개월 지속 투자
