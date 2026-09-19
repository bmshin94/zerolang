# zerolang 전수조사 분석 정리 (한국어)

> 이 문서는 zerolang 저장소를 전수조사해서 "이게 뭐고, 언제 쓰고, 나에게 어떤 도움이 되는지"를
> 정리한 학습/전략 노트입니다. 컴파일러 동작에는 영향을 주지 않는 문서 파일입니다.

- 작성일: 2026-09-19
- 분석 대상 버전: `0.3.4` (`package.json`)
- 대화 정리: Claude Code 세션에서 수행한 전수조사 결과

## 관련 GitHub / 링크

| 항목 | 주소 |
| --- | --- |
| 이 저장소 (포크) | https://github.com/bmshin94/zerolang |
| 원본 (upstream) | https://github.com/vercel-labs/zerolang |
| 릴리스 | https://github.com/vercel-labs/zerolang/releases/latest |
| 라이선스 (Apache-2.0) | https://github.com/vercel-labs/zerolang/blob/main/LICENSE |
| 공식 문서 사이트 | https://zerolang.ai |
| 설치 스크립트 | https://zerolang.ai/install.sh |
| 에이전트 스킬 설치 | `npx skills add vercel-labs/zerolang` |
| Vercel Labs 실험 목록 | https://vercel.com/labs#active-experiments |

---

## 1. 정체: 한 줄 요약

**zerolang = "AI 에이전트가 코딩하기 위해 설계된 프로그래밍 언어 + 네이티브 컴파일러"**

- Vercel Labs의 공개 실험 프로젝트 (`LABS-EXPERIMENT` 배지)
- 슬로건: **"The programming language for agents."**
- 저장소가 스스로 붙인 경고: 실험 단계이며 브레이킹 체인지 · 보안 취약점을 전제로 한다.
  격리된 워크스페이스에서만 실행할 것. (README, `AGENTS.md`)

## 2. 핵심 아이디어: "텍스트가 아니라 그래프가 소스코드다"

| 구분 | 기존 언어 (JS/Python/Go 등) | zerolang |
| --- | --- | --- |
| 진짜 소스 | `.js` / `.py` 텍스트 파일 | `zero.graph` (바이너리 그래프 스토어) |
| 에이전트 편집 방식 | 문자열 치환 / 라인 범위 수정 | `zero patch` (의미 단위 연산) |
| 잘못된 편집 | 저장은 됨 → 나중에 빌드 실패 | **저장 자체가 거부됨** |
| 사람이 읽는 것 | 소스 그 자체 | `.0` 파일 = 그래프의 **투영(projection)** |

- **기존 에이전트 루프:** 텍스트 작성 → check → format → build → 실패 분석 → 다시 작성 (반복)
- **Zero 루프:** 그래프 질의 → 검증된 패치 제출 → 컴파일러 수락/거부 → 통과 시 검증 실행

핵심은 **"잘못된 상태가 디스크에 아예 기록되지 않는다"**는 점.
오래된 그래프 해시, 예상과 다른 필드 값, 잘못된 형태, 타입 에러는 모두 **저장 전에** 실패한다.

그래프가 에이전트에게 제공하는 명시적 손잡이(handle):
심볼, 노드 ID, 그래프 해시, 타입, 이펙트, 소유권 사실, capability, import, 호출 엣지, 타겟 사실.

## 3. 폴더 전수조사 결과

| 경로 | 규모 | 역할 |
| --- | --- | --- |
| `native/zero-c/` | **C 소스 121개 파일 / 약 117,859줄** | 컴파일러 본체. 순수 C. `checker.c`, `direct_emit.c`, `emit_elf64.c`, `emit_macho64.c`, `emit_coff.c`, `aarch64_emit.c`, `emit_llvm_ir.c` → **LLVM 없이 ELF/Mach-O/COFF 실행파일 직접 생성** (LLVM 백엔드는 experimental) |
| `std/` | 약 35MB, 모듈 40여 개 | 표준 라이브러리. `.graph`가 진짜 컴파일 소스, `.0`는 사람용 투영. 예: `http.graph` 7.3MB / `http.0` 139KB |
| `skill-data/` | md 8개, 약 135KB | 에이전트용 설명서. `agent.md`(작업 루프), `graph.md`(패치 op 전체), `language.md`(문법), `stdlib.md`(92KB 시그니처 카탈로그), `diagnostics.md`, `packages.md`, `builds.md`, `testing.md` |
| `skills/zero/SKILL.md` | 3.8KB | Agent Skill **디스커버리 스텁**. 일부러 얇게 유지하고 실제 내용은 컴파일러가 서빙 |
| `examples/` | 약 20MB | 실행 가능한 예제. `crm-api`(HTTP CRM 라우터, src 7파일), `ping-pong-api`, `batch3-cli`, `c-interop`, `agent-repair-demo`, `zero-hash`, `direct-*` 백엔드 예제 다수 |
| `conformance/` | `run.mjs` 단일 파일 316KB | 언어 · CLI 동작 고정 테스트. `agent-surface/`, command-contract 스냅샷 포함 |
| `evals/` | TS 4파일 | **AI 모델 평가 하네스.** Vercel Sandbox에서 Claude Code를 띄워 과제 수행 성공률 측정. 기본 모델셋 `anthropic/claude-opus-4.7`, `anthropic/claude-sonnet-4.6` |
| `benchmarks/` | `rosetta/`, `zero/` | 빌드시간 · 런타임 · 바이너리 크기 · peak RSS 회귀 측정 |
| `docs/` | Next.js 앱 | zerolang.ai 문서 사이트. 모듈 문서 36개 + 개념 문서(`graph-architecture`, `semantic-vs-text`, `projections`, `compile-path`) |
| `extensions/vscode/` | - | `.0` 파일 문법 하이라이팅 확장 |
| `scripts/` | `.mts` 40여 개 | 검증 파이프라인. `snapshot-command-contracts.mts` 567KB, `compiler-metrics.mts` 163KB, `program-graph-parity.mts` 102KB |
| `bin/zero` | 283 bytes | 빌드된 네이티브 바이너리(`.zero/bin/zero`)로 exec 위임하는 래퍼 |
| `CLAUDE.md` | 1.4KB | 이 포크에 추가된 카리나 페르소나 파일 (PR #1로 머지) |
| `.github/workflows/` | `ci.yml`, `release.yml` | CI 잡 8개 이상 (conformance, native preflight, 런타임 2샤드, 메타데이터, 직접 백엔드 artifacts, sanitizer smoke, command-contract 스냅샷). 스케줄 deep run은 6샤드 |

## 4. 언어 문법 맛보기

```zero
pub fn main(world: World) -> Void raises {
    check world.out.write("hello from zerolang\n")
}
```

- `World` — 외부 세계(stdout / 파일 / 네트워크)에 접근할 권한을 담은 **명시적 티켓**.
  ambient global이 아니라 인자로 받아야 한다 (capability 기반).
- `raises` — 이 함수가 실패할 수 있음을 선언. 숨겨진 예외가 없다.
- `check` — 실패 가능한 호출을 하고 실패를 상위로 전파. `rescue`는 폴백.
- 기타: `let`(기본) / `var`(변경될 때만), `Maybe<T>`, `Span<T>` / `MutSpan<T>`,
  `ref<T>` / `mutref<T>` / `owned<T>`, `type` / `enum` / `choice` + exhaustive `match`,
  제네릭(`fn id<T: Type>(...)`), `test "name" { expect ... }`.
- 한 함수의 고정 로컬 합계는 128 KiB 제한 (MEM003).
- 배열 / 스팬 인덱싱은 런타임 경계 검사 → 범위 초과 시 시그널 종료.

중요한 점: 에이전트는 위 텍스트를 "타이핑"하지 않는다.
`addMain`, `addCheckWrite` 같은 **조립 명령**을 보내고 컴파일러가 구조를 만든 뒤 사람용으로 렌더링한다.

## 5. 설치 및 사용법

### 5.1 사용자 (컴파일러만 쓸 때)

```sh
curl -fsSL https://zerolang.ai/install.sh | bash
export PATH="$HOME/.zero/bin:$PATH"
zero --version

npx skills add vercel-labs/zerolang   # 에이전트 부트스트랩 스킬
```

일상 루프:

```sh
zero query                  # 그래프 둘러보기
zero patch --op help        # 사용 가능한 편집 연산 목록
zero patch --op 'addMain'
zero check
zero test
zero run -- <args>
```

사람 리뷰 / 역반영:

```sh
zero export              # .0 투영 생성
zero verify-projection   # 투영 드리프트 검사 (쓰지 않음)
zero import              # .0을 직접 고쳤을 때 그래프로 역반영
```

### 5.2 이 저장소 자체를 개발할 때

```sh
pnpm install
make -C native/zero-c    # → .zero/bin/zero
bin/zero --version
```

검증 명령:

```sh
pnpm run agent:checks     # 전체 (conformance 포함, 병렬, 격리 /tmp 워크스페이스)
pnpm run conformance
pnpm run native:test
pnpm run command-contracts
pnpm run docs:build
```

요구사항: **Node 24 이상**(`.node-version` = 24), pnpm 11.1.3, C 툴체인, **Zig**(CI에서 핀 고정 설치, 링커/크로스컴파일용).

## 6. 플러그인? 스킬? MCP?

| 분류 | 해당 여부 | 설명 |
| --- | --- | --- |
| 프로그래밍 언어 + 컴파일러 | **예 (본체)** | C 약 117,859줄 네이티브 컴파일러 + 표준 라이브러리 |
| Agent Skill | **예 (부속)** | `skills/zero/SKILL.md`, `npx skills add vercel-labs/zerolang` |
| MCP 서버 | **아니오** | 저장소에 MCP 서버 구현 없음. 에이전트는 셸에서 `zero` 명령을 직접 실행 |
| Claude Code 플러그인 | 아니오 | 플러그인 매니페스트 없음 |
| VSCode 확장 | 곁다리로 존재 | `extensions/vscode/` — `.0` 문법 하이라이팅만 |

### 버전 매칭 스킬 (핵심 설계 포인트)

`SKILL.md`는 **일부러 얇은 디스커버리 스텁**이고("This file is only a discovery stub"),
실제 워크플로 문서는 **컴파일러 바이너리가 자기 버전에 맞춰 직접 서빙**한다.

```sh
zero skills                              # 토픽 목록
zero skills get agent                    # ~4KB   읽기-편집-검증 루프
zero skills get language                 # ~6KB   문법
zero skills get graph                    # ~9KB   패치 연산 전체
zero skills get stdlib                   # ~39KB  전체 시그니처 카탈로그
zero skills get stdlib --topic std.time  # ~1KB   한 모듈 섹션만
```

이유: 스킬 문서가 컴파일러 버전과 어긋나면 에이전트가 없는 기능을 쓰려 한다.
그리고 `--topic`으로 필요한 섹션만 받아 **토큰을 절약**한다.

## 7. API 토큰 필요 여부

**컴파일러 사용에는 토큰이 전혀 필요 없다. 무료 · 오프라인 동작.**
`zero init` / `patch` / `check` / `test` / `run` / `build`는 모두 로컬 네이티브 바이너리다.

토큰이 필요한 곳은 두 군데뿐:

1. `evals/` 라이브 실행 — `AI_GATEWAY_API_KEY`, 그리고 `VERCEL_OIDC_TOKEN`
   (또는 `VERCEL_TOKEN` + `VERCEL_TEAM_ID` + `VERCEL_PROJECT_ID`).
   API 키 없이 고정 fixture로만 돌리는 것도 가능: `pnpm evals -- --case hello-world --fixture`
2. 에이전트 쪽 (Claude Code 등) 자체 요금 — zerolang 비용이 아니다.

## 8. GitHub에서 유명한 이유 (분석)

1. **Vercel Labs 브랜드** — Next.js / v0 / AI SDK를 만든 팀의 실험은 자동으로 화제가 된다.
2. **타이밍** — 다들 "기존 언어에 AI를 끼워넣는" 방향인데, Zero는 반대로 "AI를 위해 언어를 처음부터 설계"했다. 질문 자체를 뒤집었다.
3. **강한 한 문장** — "The programming language for agents." / "The semantic graph is the program database." → "텍스트 소스코드는 레거시인가?"라는 논쟁을 유발.
4. **실제 구현 밀도** — C 약 117,859줄 컴파일러, LLVM 없이 ELF/Mach-O/COFF 직접 생성, std 모듈 40여 개, conformance runner 316KB, command-contract 스냅샷 567KB, CI 잡 8개 이상, sanitizer smoke, 6샤드 deep matrix, 동작하는 HTTP CRM API 예제.
5. **디테일과 태도** — 확장자가 `.0`, 보안 경고를 당당히 명시, `AGENTS.md`에 "레거시 호환 보존 금지"를 원칙으로 선언.

## 9. 로컬 에이전트 구축에 도움되는 5가지 설계 패턴 (핵심 수확)

"Zero로 코딩하라"가 아니라 **"Zero의 설계를 이식하라"**가 실질적 가치다.

### 9.1 도구가 자기 문서를 서빙 (version-matched skills)

```sh
mytool skills                       # 토픽 목록
mytool skills get api --topic auth  # 필요한 섹션만
```

→ 문서-코드 불일치로 에이전트가 헛짓하는 문제를 원천 차단.

### 9.2 낙관적 락 (`expect` 패턴) — 가장 가치 있는 아이디어

```sh
zero patch --op 'set node="#a647" field="value" expect="1" value="8"'
expect graphHash "graph:a7f7e6899a73f3b4"
```

"현재 값이 `1`인 게 맞으면 `8`로 바꿔라. 아니면 거부."
→ 에이전트가 오래된 정보로 엉뚱한 위치를 고치는 사고를 구조적으로 차단.
React 상태관리, DB 업데이트, 어떤 도메인에도 적용 가능.

### 9.3 계층적 읽기 API (토큰 다이어트)

```
--outline          시그니처 + 한 줄 doc만
--fn <name>        함수 하나만
--around <text>    그 텍스트를 감싸는 블록만
--topic <prefix>   문서 한 섹션만
--handles          편집용 핸들 포함
```

→ 에이전트 비용의 대부분은 "불필요한 전체 파일 읽기". 읽기 API 계층화가 비용 절감의 핵심.

### 9.4 구조적 편집 연산

```sh
addParamTo fn="scan" name="bias" type="i32" default="0"   # → "updated N call sites"
zero patch --rewrite 'bnCmp($A, $B) == 0' --to 'bnEq($A, $B)' --apply
```

`$A`, `$B`는 임의의 표현식 서브트리를 바인딩하고, 같은 메타변수가 두 번 쓰이면 동일 서브트리여야 한다.
→ 12곳을 각각 고치면 12번 실수할 수 있다. **1번의 의미적 명령**으로 만들면 실수 확률이 1/12.
→ JS/TS는 `ts-morph`, PHP는 `nikic/php-parser`로 동일 구현 가능.

### 9.5 dry-run을 기본값으로

```sh
zero patch --rewrite '...' --to '...'            # 기본 = dry run, 바뀔 목록만 출력
zero patch --rewrite '...' --to '...' --apply    # 실제 적용은 명시적으로
zero patch --check-only / --dry-run --json
```

→ 위험한 일괄 작업은 "미리보기가 기본, 실행이 옵션". 에이전트 안전장치의 정석.

### 9.6 프롬프트 설계 참고 문구 (`skill-data/agent.md`)

- "After `validated: check-equivalent`, the graph is saved and checked.
  **Do not run `zero check`, `zero view`, or `zero export` just to confirm.**"
  → 에이전트가 불필요한 확인 명령을 남발하는 걸 막는다. **"안 해도 되는 일"을 명시하는 것**이 핵심.
- "**Do not invent syntax or CLI fields**; load `language` when unsure." → 환각 방지.
- "**Check `stdlib` before hand-writing parsing or validation**" → 바퀴 재발명 방지.
  (`std.time` RFC 3339 / `std.inet` / `std.regex` / `std.unicode` 검증기 내장)

## 10. React / PHP로 만들 수 있나?

### 레벨 1 — Zero 컴파일러 자체를 React/PHP로: 비현실적

C 약 117,859줄, ELF/Mach-O/COFF 직접 생성, aarch64 기계어 emit. 시스템 프로그래밍 영역이고 투자 대비 의미가 없다.

### 레벨 2 — Zero의 아이디어를 JS/TS/PHP에 적용: 완전 가능 (정답)

"그래프가 소스다"는 결국 **"AST를 DB처럼 다뤄라"**이고, 도구는 이미 존재한다.

TypeScript / React:

```
ts-morph / @babel/parser / SWC → AST
  ↓ Node.js CLI + MCP 서버
mycode query --component Button --props
mycode patch --add-prop Button name="size" type="'sm'|'lg'" default="'sm'"   # 모든 사용처 자동 갱신
mycode patch --rename-hook useAuth --to useSession --expect-hash abc123
mycode rewrite 'useState<$T>($A)' --to 'useSignal<$T>($A)' --apply
```

| 작업 | 텍스트 방식 | 그래프 방식 |
| --- | --- | --- |
| prop 하나 추가 | 사용처 47곳 수동 수정 | `addPropTo` 1번 → "updated 47 사용처" |
| 컴포넌트 rename | import 전부 grep | `rename` 1번 (import 자동) |
| 이 컴포넌트 누가 쓰나 | grep 추측 | `--refs Button` (정확) |
| 클래스명 일괄 변경 | 정규식 지옥 | 구조적 rewrite |

PHP:

```
nikic/php-parser → AST (PHP 생태계 표준)
  ↓ Symfony Console CLI
code:query --class UserService --methods
code:patch --add-method UserService --name findByEmail ...
```

PHP에는 이미 Rector(nikic/php-parser 기반)라는 구조적 리팩토링 도구가 있다.
거기에 Zero 스타일의 **`expect` 낙관적 락 + `--handles` + dry-run 기본값 + MCP 노출**을 얹으면
그것이 "PHP용 에이전트 코딩 레이어"가 된다.

### 레벨 3 — React로 UI 레이어: 가장 현실적이고 수익성 높은 방향

Zero는 CLI만 있고 GUI가 없다. 그런데 `zero query --json`, `zero inspect --json`,
`zero patch --dry-run --json` 등 **JSON 출력이 전부 준비되어 있다.**

React로 만들 수 있는 것:

- 노드/엣지 그래프 뷰 (React Flow)
- 에이전트가 제안한 패치의 before/after 비교 뷰
- 승인 / 거부 버튼 → `zero patch --apply`
- `.0` 투영 실시간 미리보기

이 UI 레이어는 Zero 전용이 아니라 **모든 AST 기반 에이전트 도구에 재사용 가능**하다.

## 11. 수익화 아이디어

전제: **zerolang 자체로 직접 돈 벌기는 어렵다.** Apache-2.0(누구나 무료 상업 이용 가능),
Labs 실험체(중단 가능, 브레이킹 체인지가 기본값), 생태계 사실상 없음.

→ 전략 A: **Zero에서 배운 설계로 다른 시장에서 수익화** (현실적, 추천)
→ 전략 B: Zero 생태계 초기 선점 (고위험 고수익)

### A-1. "Graph Layer for React/TS" — 에이전트용 구조적 편집 CLI + MCP

- **문제:** 현재 AI 코딩 도구의 최대 비용/실패 요인 두 가지 — (1) 전체 파일 읽기로 토큰 낭비, (2) 문자열 치환 실패로 재시도 반복.
- **해법:** ts-morph 위에 Zero의 편집 철학을 이식.
- **구성:** ① CLI ② **MCP 서버**(Claude Code / Cursor / Windsurf에 바로 연결 — 배포 경로가 이미 존재) ③ VSCode 확장(패치 미리보기 + 승인 UI)
- **킬러 기능:** 계층적 읽기(토큰 60~80% 절감), `expect` 낙관적 락, `addPropTo` 사용처 자동 갱신, 구조적 rewrite, dry-run 기본값
- **가격:** 오픈코어 — CLI 무료(MIT) / 팀 $20~30/개발자/월(커스텀 룰, 팀 룰셋, 감사 로그, CI 통합) / 엔터프라이즈 온프렘·SSO
- **난이도:** 중 (ts-morph가 무거운 작업을 담당, MVP 4~8주)
- **리스크:** Cursor·Anthropic이 직접 구현 가능 → 틈새(모노레포 대규모 리팩토링, 디자인시스템 마이그레이션)를 깊게 파야 승산

### A-2. "Agent-Ready Codebase" 감사 / 컨설팅 — 가장 빠른 현금흐름

- **문제:** 회사들이 Cursor / Claude Code를 도입했는데 성과가 안 난다. 원인은 도구가 아니라 **코드베이스가 에이전트 친화적이지 않아서**다. `skill-data/agent.md`가 그 처방전이다.
- **제공 항목:**
  1. 에이전트 관측성 감사 — 함수 하나만 읽을 수 있나? `--outline`급 도구가 있나?
  2. 버전 매칭 문서 시스템 구축 — `npm run skills get <topic>` 방식
  3. `CLAUDE.md` / `AGENTS.md` 설계 — Zero의 "Do not ~" 패턴 적용
  4. 도메인 전용 구조적 편집 스크립트(codemod) 제작 + MCP 노출
  5. 토큰 비용 진단 리포트 — "읽기 API 계층화로 월 API 비용 X% 절감"
- **가격:** 초기 감사 300~800만원 / 구축 1,500~5,000만원 / 리테이너 월 200~500만원
- **난이도:** 낮음 (제품 없이 지식으로 시작 가능)
- **시작법:** 이 분석을 블로그/발표로 공개 → 그 자체가 영업 자산. 한국에서 이 주제를 깊게 다루는 사람이 적어 선점 가능.

### A-3. 에이전트 패치 리뷰 UI (React SaaS)

- **문제:** 진짜 병목은 생성이 아니라 **리뷰**다. AI가 500줄 diff를 던지면 사람은 훑고 그냥 승인한다.
- **기능:** "이 PR이 실제로 바꾼 의미"(함수 3개 시그니처 변경, 호출부 12곳 갱신, 새 capability 1개 요구) / React Flow 영향 그래프 / 위험 신호 하이라이트(새 네트워크·파일 권한, 에러 처리 삭제, 테스트 없는 동작 변경) / 부분 승인
- **기술:** React + React Flow + Node 백엔드(ts-morph, php-parser) + GitHub App
- **가격:** 리포당 $99~499/월, 엔터프라이즈 온프렘
- **난이도:** 중상 (GitHub 통합 + 다언어 AST)
- **적합성:** 프론트엔드가 제품의 핵심 → React 역량과 궁합이 가장 좋다.

### A-4. 교육 콘텐츠 / 유료 강의 — 가장 쉬운 시작

- **주제:** "AI 에이전트를 위한 도구 설계 — Vercel zerolang 해부"
- **차별점:** "AI로 코딩하기"는 포화, **"AI가 쓸 도구를 어떻게 만드나"**는 희소. Zero는 실제 동작하고 설계 의도가 문서에 남아 있어 교보재로 적합.
- **포맷:** 무료 블로그 5편(인바운드) → 유료 강의 15~25만원 → 유료 뉴스레터 월 1만원 → 컨퍼런스 발표(A-2 영업 연결)
- **난이도:** 매우 낮음 / **가치:** 단독 수익은 작지만 A-1·A-2·A-3의 마케팅 엔진이 된다.

### B-1. Zero 생태계 초기 선점 (로또)

패키지 레지스트리 / 호스팅 / 프레임워크를 선점. Vercel이 밀면 초기 리더 포지션.
리스크: 프로젝트 중단 시 전부 무효. → 사업이 아니라 **취미 + 기여자 크레딧 축적**으로 접근.
(기여 이력 자체가 A-2 컨설팅의 신뢰 자산이 된다.)

### B-2. Graph-native 도메인 특화 언어 (로또)

Zero의 철학을 좁은 도메인(워크플로 정의, 인프라 설정, 게임 로직)에 적용.
범용 언어보다 작게 만들 수 있고 검증 로직이 강력하다. 난이도 높음. 도메인 전문성이 있으면 유망.

### 추천 로드맵

```
[1~2개월] A-4 콘텐츠        → 분석 시리즈 공개, 인지도·신뢰 확보 (리스크 0)
   ↓
[2~4개월] A-1 MVP           → ts-morph + MCP 서버, 오픈소스 공개 = 최고의 영업
   ↓
[3~6개월] A-2 컨설팅        → 유입 문의를 유료 감사로 전환 (현금흐름)
   ↓
[6개월~]  A-3 SaaS 제품화   → 컨설팅 수익 + 검증된 니즈로 제품화
```

각 단계가 다음 단계의 영업 자산이 되고, 초기 투자 비용이 거의 없다.

**정직한 경고:** "zerolang으로 서비스를 만들어 돈 벌기"는 현재 불가능하다.
실험체이고 브레이킹 체인지가 기본값이며 보안 취약점을 전제한다.
**Zero는 "제품"이 아니라 "교재"로 쓸 때 가치가 가장 크다.**

## 12. 정리: 오빠에게 무슨 도움이 되나

**바로 도움되는 것**

- 에이전트용 도구 설계의 최고 교과서 (`skill-data/agent.md`, `graph.md`)
- 버전 매칭 스킬 패턴 — 도구가 자기 문서를 서빙
- 토큰 절약 설계 — 계층적 읽기 API
- 낙관적 락(`expect`)과 구조적 편집 연산 — 에이전트 실수 억제
- dry-run 기본값 — 안전장치 설계

**한계**

- 0.3.x 실험체 → 실서비스 금지
- 생태계 없음 (npm / composer 상당물 없음)
- 브레이킹 체인지가 기본 방침 (`AGENTS.md`: "Do not preserve legacy behavior by default")
- 빌드 요구사항이 무겁다 (Node 24+, Zig, C 툴체인)

## 13. 이 문서의 검증 상태 (정직한 기록)

- 이 문서는 `notes/` 아래의 **한국어 학습/전략 노트**이며, 컴파일러 · 표준 라이브러리 ·
  예제 · 문서 사이트 빌드 · conformance 대상 파일을 **전혀 건드리지 않는다.**
- `AGENTS.md`는 저장소를 변경하는 에이전트 턴마다 `pnpm run conformance` 실행을 요구하지만,
  이 작업이 이루어진 환경에는 **Zig가 설치되어 있지 않고 Node가 22.22.2**여서
  (저장소 요구사항은 24 이상) `make -C native/zero-c` 및 conformance 스위트를 실행할 수 없었다.
  변경 범위가 문서 파일 추가뿐이므로 컴파일러 동작에 영향은 없으나, 사실대로 기록해 둔다.
- 재현하려면: Node 24+ 와 Zig를 설치한 환경에서 `pnpm install && make -C native/zero-c`
  이후 `pnpm run agent:checks`.
