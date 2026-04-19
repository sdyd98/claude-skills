# Claude Code 하네스 비교 분석 브리핑

- **작성일**: 2026-04-20
- **문서 성격**: **중립 레퍼런스** — 공식 Claude Code 확장점(2026-04 기준) + 7축 설계 프레임 + Phase 전략 원칙
- **사용 대상**: 사내 하네스와 비교 분석하는 Claude Code 에이전트
- **중요**: 이 문서는 **정답이나 권장안이 아니다**. 사내 하네스의 설계 결정이 팀 맥락상 더 적합할 수 있음. "좋다/나쁘다" 판단이 아니라 "커버/미커버" 식별에만 사용할 것.

---

## §1. 비교 분석의 맥락

### 1.1 팀 상황
- C++ 게임서버 개발팀 (6명, 전담 2일 투입)
- 스택: Windows / MSVC / Visual Studio / MSBuild `.vcxproj` 직접
- 빌드 구조: 여러 `.lib` 모듈 → `GameServer.exe` 단일 산출물
- 의존: FlatBuffers, Jira, Jenkins
- 사내 코드생성 툴 3종: 패킷 codegen, DB codegen, xls codegen
- AI: Claude Code (메인) + Codex CLI (서브, 교차 검증 후보)

### 1.2 팀장의 지시·목표
- "AI로 게임서버 개발 속도 가속"
- 제품용 AI가 아닌 **내부 개발 생산성 도구**
- 설계 의도: 하이브리드 **Track A + Track B**
  - **Track A**: 개발 워크플로 오케스트레이션 (명확화·계획·구현·검증·커밋)
  - **Track B**: 비즈니스 코드 생성·리뷰·테스트, 목표 품질 바 "거의 머지 가능"

### 1.3 팀 제약 (비교 시 고려)
- **테스트 인프라 현재 0** — AI 검증 도구 부재 상태
- 코드베이스 **580만 줄** — 컨텍스트 윈도우 대비 큰 규모
- 보안 정책 승인됨 (사내 코드 외부 전송 가능)
- 팀원 6명 전원 AI 도구 활용 의지 있음
- GoogleTest/Catch2 경험자 0명

---

## §2. 7축 설계 프레임 (레퍼런스, 절대 기준 아님)

LLM 하네스의 요구사항을 쪼개는 한 가지 체크리스트. Anthropic "Building effective agents" (3~4축), ReAct (2축), LangChain 컴포넌트(7) 등 다른 프레이밍도 존재. 이 프레임을 사용하는 이유는 Claude Code 확장점과의 매핑이 자연스럽기 때문.

| # | 축 | 하는 일 | 구멍 났을 때의 실패 모드 |
|---|----|--------|----------------------|
| ① | 프롬프팅 | 시스템·태스크 프롬프트, 스타일·제약 | 같은 요청에 매번 다른 출력 |
| ② | 컨텍스트 관리 | 모델이 보는 것 — 파일·메모리·검색 결과 | 일반론만 하거나 맥락 밖 판단 |
| ③ | 툴 호출 루프 | 툴 사용 시점·방식·결과 피드백 | LLM이 "말만 하고" 실행 못 함 |
| ④ | 상태·메모리 | 세션 내·대화·장기 지속 | 어제 고친 걸 오늘 또 해결, 팀 규약 매번 재주입 |
| ⑤ | 종료·제어 흐름 | 언제 멈출지, 언제 에스컬레이트할지 | 무한 루프, 비용 폭주, 과수정 |
| ⑥ | 안전 가드 | 뭘 못 하게 할지, 승인 흐름 | 위험 작업을 AI가 혼자 결정 |
| ⑦ | 관측성 | 로그·트레이싱·eval | 개선·회귀 감지 불가 |

**중요**: 하나의 확장점은 **여러 축에 동시에 걸침** (예: Hooks는 ⑤·⑥·⑦ 모두 관여). "이 수단 = 이 축"의 1:1 매핑은 잘못된 모델.

---

## §3. Claude Code 공식 확장점 카탈로그 (2026-04 기준)

출처: `https://code.claude.com/docs/en/` 공식 문서 실측.

### 3.1 CLAUDE.md (Memory)
- 4 scope: **Managed** / **Project** (`./CLAUDE.md` or `./.claude/CLAUDE.md`) / **User** (`~/.claude/CLAUDE.md`) / **Local** (`./CLAUDE.local.md`, gitignored)
- 우선순위: 더 구체적 scope 우선. 모든 파일은 **병합**되어 로드됨
- `@path/to/file` **import 문법** — 상대/절대 경로, 최대 5 hops 재귀
- 권장 크기 **≤200줄** — 초과 시 `.claude/rules/` 또는 `@import`로 분할
- 초기화: `/init` (기본) 또는 `CLAUDE_CODE_NEW_INIT=1 /init` (인터랙티브 다단계)
- `AGENTS.md` 직접 읽지 않음 — CLAUDE.md에서 `@AGENTS.md` 해야 함
- 블록 HTML 주석 `<!-- -->` 은 context에서 제거됨

### 3.2 `.claude/rules/` (경로 스코프 규칙)
- 도메인별 규칙 모듈화: `.claude/rules/testing.md`, `.claude/rules/security.md` 등
- **`paths:` frontmatter**로 파일 경로 스코프 지정 가능
  ```yaml
  ---
  paths:
    - "src/api/**/*.ts"
    - "lib/**/*.ts"
  ---
  ```
- `paths` 없으면 모든 세션에 로드 (CLAUDE.md와 동등)
- `paths` 있으면 해당 파일 접근 시 자동 로드
- 심볼릭 링크 지원 (공유 규칙 라이브러리 구성 가능)
- User 레벨: `~/.claude/rules/`

### 3.3 Skills (Custom Commands 통합)
- **중요**: `.claude/commands/` 는 `.claude/skills/`로 통합됨 (하위 호환 유지)
- 기본 위치: `.claude/skills/<name>/SKILL.md`
- scope: Enterprise / Personal (`~/.claude/skills/`) / Project (`.claude/skills/`) / Plugin
- frontmatter 필드:
  - `name`, `description`, `when_to_use`
  - `argument-hint`
  - `disable-model-invocation` (사용자만 호출)
  - `user-invocable: false` (Claude만 호출)
  - `allowed-tools`
  - `model`, `effort` (low/medium/high/xhigh/max)
  - `context: fork` + `agent: Explore` (서브에이전트에서 실행)
  - `hooks`, `paths`, `shell`
- **문자열 치환**: `$ARGUMENTS`, `$ARGUMENTS[N]`, `$N`, `${CLAUDE_SESSION_ID}`, `${CLAUDE_SKILL_DIR}`
- **동적 컨텍스트 주입**: `` !`<command>` `` (인라인) 또는 ` ```! ` 블록 (다중 라인) — Claude 전에 실행되어 결과가 프롬프트로 주입됨
- Bundled skills (기본 제공): `/simplify`, `/batch`, `/debug`, `/loop`, `/claude-api` 등
- Live change detection — 세션 중 파일 변경 즉시 반영
- Nested directories 자동 발견 (모노레포 지원)

### 3.4 Subagents
- 위치: `.claude/agents/<name>.md`
- frontmatter:
  - `name`, `description`
  - `tools` (허용 도구 목록)
  - `model`
- 별도 컨텍스트 윈도우·독립 권한
- Claude는 description 기반으로 **자동 위임** 결정
- `enable-persistent-memory`로 subagent 전용 auto memory 유지 가능
- Skills의 `context: fork`로 subagent에서 skill 실행 가능

### 3.5 Hooks
- **세션 레벨**: `SessionStart`, `SessionEnd`
- **턴 레벨**: `UserPromptSubmit`, `Stop`, `StopFailure`
- **툴 레벨**: `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `PermissionRequest`, `PermissionDenied`
- **추가 이벤트 (20+)**: `Notification`, `SubagentStart`, `SubagentStop`, `TaskCreated`, `TaskCompleted`, `TeammateIdle`, `ConfigChange`, `CwdChanged`, `FileChanged`, `InstructionsLoaded`, `PreCompact`, `PostCompact`, `WorktreeCreate`, `WorktreeRemove`, `Elicitation`, `ElicitationResult`
- **4가지 핸들러 타입**:
  - `command` — shell script
  - `http` — HTTP endpoint
  - `prompt` — LLM 프롬프트를 hook으로 (AI 판단을 결정적으로 호출)
  - `agent` — subagent를 hook으로
- Exit code: `0`(성공), `2`(차단, stderr→Claude), 기타(비차단)
- Matcher: 정확·정규식·와일드카드 지원
- Decision values: `allow` / `deny` / `ask` / `defer`
- 보안: `allowedEnvVars`, `disableAllHooks`, `allowManagedHooksOnly`
- 위치: 4 scope settings에 설정

### 3.6 MCP (Model Context Protocol)
- 3종 primitive: **Tool** / **Resource** / **Prompt**
- Transport: **stdio** / **HTTP** / **SSE**
- 기성 MCP registry 제공 (Jira, Slack, GitHub, Google Drive 등)
- `claude mcp add` CLI 또는 settings.json 등록
- 권한: settings의 `allowedTools`로 제어
- 자작 시 TypeScript/Python SDK 권장
- Channels (관련 기능): 외부 웹훅·알림을 세션에 push하는 MCP 기반

### 3.7 Settings
- 4 scope: **Managed** (`/Library/Application Support/ClaudeCode/`, `/etc/claude-code/`, `C:\Program Files\ClaudeCode\`) / **User** (`~/.claude/`) / **Project** (`.claude/`) / **Local** (`.claude/settings.local.json`)
- 주요 필드:
  - `hooks`
  - `permissions` (allow/deny/ask 리스트)
  - `env` (환경 변수)
  - `claudeMdExcludes` (glob 패턴)
  - `autoMemoryEnabled`, `autoMemoryDirectory`
  - `disableSkillShellExecution`
  - `sandbox` 관련
  - `forceLoginMethod`, `forceLoginOrgUUID`
- `/config` 명령으로 대화형 편집
- 팀 공유: Project scope를 git commit

### 3.8 Auto Memory
- 경로: `~/.claude/projects/<project>/memory/`
- 구조: `MEMORY.md`(인덱스) + 토픽별 `.md` 파일
- `MEMORY.md` 첫 200줄 / 25KB 만 세션 시작 시 로드
- Topic 파일은 on-demand 로드
- 토글: `/memory` 명령, `autoMemoryEnabled` 설정, `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`
- 위치 변경: `autoMemoryDirectory` (user/local 설정에서만)
- 필요 버전: v2.1.59+
- Subagent도 자체 auto memory 가능

### 3.9 Plugins
- 구조: `.claude-plugin/` 디렉토리 + manifest
- Skills / Subagents / Hooks / MCP 번들로 패키징
- `/plugin` 명령으로 마켓플레이스 브라우징
- 사내 자체 마켓플레이스 호스팅 가능 (`plugin-marketplaces.md` 참조)

### 3.10 상위 인프라·워크플로 기능
- **Plan Mode** — 탐색·계획 분리 (`common-workflows.md`)
- **Auto mode** — classifier 기반 자동 승인 (`--permission-mode auto`)
- **Sandboxing** — OS 레벨 격리 (`sandboxing.md`)
- **Rewind / Checkpointing** — 대화·코드 되돌리기
- **Routines** — Anthropic 인프라에서 cron/이벤트 트리거 실행
- **Channels** — 외부 이벤트 push
- **Ultraplan / Ultrareview** — 클라우드 기반 멀티 에이전트 계획·리뷰
- **Agent Teams** — 여러 세션 간 조정·메시징
- **Agent SDK** — Python/TypeScript로 커스텀 에이전트 구축
- **Non-interactive mode** — `claude -p` (CI·파이프라인 통합)
- **GitHub Code Review** — 공식 자동 PR 리뷰
- **GitHub Actions / GitLab CI/CD** — 공식 통합
- **Slack 통합** — `@Claude` 멘션 → PR

### 3.11 관측성·비용
- **Statusline** — context·cost 실시간 표시
- **Costs dashboard** (`/en/costs`) — 팀·명령별 비용
- **Analytics dashboard** (`/en/analytics`) — 채택·속도 메트릭
- **OpenTelemetry** (`/en/monitoring-usage`) — traces·metrics·events
- **Transcript logs** — `transcript_path`
- **Session replay** — checkpointing

---

## §4. Phase 전략 원칙 (중립)

### 4.1 일반적 단계 구분
| 단계 | 성격 | 예시 활동 |
|--|--|--|
| 탐색 | 팀·환경 이해, 기준선 | 인터뷰, 기존 자산 인벤토리, 기준선 측정 |
| 기반 (MVP) | 최소 가치 산출 | 핵심 워크플로 1~3개, 기본 자동화 |
| 확장 | 범위 확대 | 추가 워크플로, 자작 MCP, 고도화 |
| 배포 | 조직 확산 | 플러그인화, 문서화, 온보딩 |

### 4.2 설계 원칙 (공식 Best Practices 기반)
- **Give Claude a way to verify its work** — 가장 큰 단일 레버
- **Explore first, then plan, then code** — 성급한 구현 지양
- **Specific context in prompts** — 모호성 = 오류
- **기성 → 조합 → 자작** 순서 — 자작은 마지막 수단
- **결정적 자동화 자리엔 hook, AI 판단 자리엔 prompt/agent hook** — 4가지 핸들러 타입 의도적 선택
- **CLAUDE.md는 ≤200줄, 프루닝 적극** — 긴 파일 → 지시 무시
- **Subagent는 "별도 컨텍스트 이유 뚜렷할 때만"** — 과잉 시 비용·복잡도 폭증

---

## §5. 검증·측정 프레임 (중립)

### 5.1 3층 지표
- **층 A — 정량** (자동 수집): 토큰/비용/wall time/tool call 수/빌드·테스트 통과율/재시도 수
- **층 B — 품질** (구성 필요): 머지 준비율, 리뷰 왕복 수, diff 크기, rework rate, 정적분석 경고 추이
- **층 C — 생산성**: Lead time, bug regression, AI-assisted share, 만족도

### 5.2 5가지 방법론
1. **Eval dataset** — 과거 실제 태스크 20~50개 골든셋
2. **Shadow / Canary** — A/B 팀 분할 비교
3. **Regression suite (CI)** — 프롬프트·Skill 변경 시 자동 회귀
4. **Golden task + LLM Judge** — 확장 가능한 품질 채점
5. **Production telemetry** — analytics·costs·OpenTelemetry

### 5.3 주의 함정
- Flakiness (N회 반복 평균), Overfit (held-out), Hawthorne (무관측 집계), Goodhart (복수 지표), 통계 유의성 (월 단위)

---

## §6. 비교 분석 시 주의사항

### 6.1 편향 방지
- 이 브리핑은 **레퍼런스**이지 정답 아님
- 사내 하네스의 설계 결정은 **팀 맥락·과거 경험**이 반영됐을 수 있음 — 단정 금지
- "좋다/나쁘다"가 아니라 **"커버/미커버"**만 판단
- 버전 차이: 사내 Claude Code가 구버전이면 §3 기능 중 일부 미지원 가능

### 6.2 비교의 한계
- 팀의 **암묵적 설계 의도**는 파일로 보이지 않음 — AI 출력은 토론 자료로만 쓰고 최종 판단은 팀 대면 리뷰로
- 사내 하네스가 "미커버" 상태여도 의도적 단순화일 수 있음
- 공식 기능이 많다고 모두 도입해야 하는 건 아님 — 팀 규모·제약에 맞는 선택이 핵심

### 6.3 객관성 유지 원칙
- 실제로 읽은 파일만 근거로 (Read tool 호출 결과만)
- 읽을 수 없는 부분은 **"확인 불가"**로 명시
- 추측 부분은 **"추정"**으로 구분 표기
- 팀에 확인이 필요한 내용은 **"확인 필요 질문"** 섹션으로 수집

---

## §7. 참고 URL (팩트 체크용)

- Overview: `https://code.claude.com/docs/en/overview`
- Features overview: `https://code.claude.com/docs/en/features-overview`
- Memory: `https://code.claude.com/docs/en/memory`
- Skills: `https://code.claude.com/docs/en/skills`
- Subagents: `https://code.claude.com/docs/en/sub-agents`
- Hooks: `https://code.claude.com/docs/en/hooks`
- MCP: `https://code.claude.com/docs/en/mcp`
- Settings: `https://code.claude.com/docs/en/settings`
- Plugins: `https://code.claude.com/docs/en/plugins`
- Best Practices: `https://code.claude.com/docs/en/best-practices`
- Agent SDK: `https://code.claude.com/docs/en/agent-sdk/overview`
- Whats new (주간 릴리스): `https://code.claude.com/docs/en/whats-new/`
