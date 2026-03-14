# Architecture Enforcer

모듈 경계, 레이어 의존성, 네이밍 컨벤션, 구조적 무결성을 검증하는 Claude Code 플러그인입니다.

## 개요

`architecture-enforcer`는 `.arch-rules.json` 설정 파일을 기반으로 코드베이스의 아키텍처 규칙을 검증하고 위반 사항을 리포팅하는 Claude Code CLI 플러그인입니다. 6개 이상의 프로그래밍 언어를 지원하며, CI/CD 파이프라인 및 pre-commit 훅과 통합할 수 있습니다.

## 주요 기능

| 기능 | 설명 |
|------|------|
| 레이어 의존성 검사 | 프레젠테이션 → 비즈니스 → 데이터 등 레이어 간 의존성 규칙 강제 |
| 모듈 경계 검사 | 모듈 간 허용/금지 의존성 규칙 적용 |
| 순환 의존성 탐지 | DFS 기반 순환 의존성 탐지 및 해결 방안 제안 |
| 네이밍 컨벤션 | PascalCase, camelCase, kebab-case 등 파일 이름 규칙 강제 |
| 결합도 분석 | 모듈 간 결합도 측정 및 제한 |
| 아키텍처 감사 | 건강 점수, 메트릭, 개선 권장사항 리포트 |
| 하네스 린팅 | 에이전트가 직접 실행 가능한 수정 지침이 포함된 위반 리포트 |
| 6계층 아키텍처 모델 | Types → Config → Repo → Service → Runtime → UI 고정 레이어 |
| Providers 패턴 | 로깅, 인증, 텔레메트리 등 횡단 관심사의 단일 인터페이스 |
| MCP 서버 | 코딩 중 실시간 아키텍처 규칙 검증 도구 |

## 지원 언어

TypeScript, JavaScript, Python, Java, Go, Rust

## 지원 아키텍처 스타일

- **Layered Architecture** — 전통적 3계층 아키텍처
- **Hexagonal Architecture** — 포트 & 어댑터
- **Modular Architecture** — 자기 완결형 모듈
- **DDD with Bounded Contexts** — 도메인 주도 설계
- **6-Layer Harness Model** — Types → Config → Repo → Service → Runtime → UI

## 설치

프로젝트의 `.claude/plugins/` 디렉토리에 clone합니다:

```bash
git clone https://github.com/wowoyong/claude-plugin-architecture-enforcer.git .claude/plugins/architecture-enforcer
```

### 업데이트

```bash
cd .claude/plugins/architecture-enforcer && git pull
```


## 사용법

### 커맨드

```bash
# 아키텍처 규칙 초기화 (대화형)
/arch-init
/arch-init --preset layered
/arch-init --preset hexagonal
/arch-init --preset harness     # 6계층 하네스 모델

# 아키텍처 규칙 검사
/arch-check
/arch-check --fix
/arch-check --diff            # 변경된 파일만 검사
/arch-check --json            # JSON 출력
/arch-check --module src/services

# 하네스 린팅 (에이전트 친화적 출력)
/arch lint                    # 전체 하네스 린트 실행
/arch lint --format compact   # CI/CD용 간결 출력
/arch lint --format agent-only # 수정 지침만 출력
/arch lint --diff             # 변경된 파일만 검사

# Providers 패턴 설정
/arch providers               # 횡단 관심사 Providers 설정

# 아키텍처 감사 (깊이 있는 분석)
/arch audit
```

### 설정 파일

프로젝트 루트에 `.arch-rules.json`을 생성하여 규칙을 정의합니다. `/arch-init` 명령으로 대화형으로 생성할 수 있습니다.

```json
{
  "version": "1.0",
  "projectType": "layered",
  "language": "typescript",
  "rootDir": "src",
  "layers": {
    "order": ["presentation", "application", "infrastructure"],
    "strict": true
  },
  "modules": {
    "src/components": { "allowedDeps": ["src/hooks", "src/services", "src/types"] },
    "src/services": { "allowedDeps": ["src/repositories", "src/types"] }
  },
  "rules": [
    { "id": "no-circular-deps", "type": "no-circular", "severity": "error" },
    { "id": "component-naming", "type": "naming", "pattern": "src/components/**/*.tsx", "convention": "PascalCase" }
  ]
}
```

## 플러그인 구조

```
architecture-enforcer/
├── .claude-plugin/
│   └── plugin.json               # 플러그인 메타데이터 및 설정
├── skills/
│   └── architecture/
│       └── SKILL.md              # 전체 아키텍처 워크플로우 (설정 스키마 포함)
├── commands/
│   ├── arch-check.md             # /arch-check 명령어 정의
│   └── arch-init.md              # /arch-init 명령어 정의
├── agents/
│   ├── boundary-checker.md       # 모듈 경계 및 의존성 분석 에이전트
│   ├── architecture-auditor.md   # 아키텍처 건강 감사 에이전트
│   └── harness-linter.md         # 에이전트 친화적 하네스 린터
├── mcp/
│   └── server-spec.md            # MCP 서버 도구 명세
├── docs/
│   └── layers.md                 # 6계층 아키텍처 모델 레퍼런스
└── README.md                     # 이 파일
```

## 하네스 엔지니어링 지원

### 에이전트 친화적 린팅

하네스 린터(`/arch lint`)는 모든 위반 사항에 대해 에이전트가 직접 실행할 수 있는 수정 지침을 생성합니다. 각 위반 리포트에 포함되는 정보:

- **What**: 위반 설명
- **Where**: 파일, 라인, 컬럼
- **Why**: 위반된 아키텍처 규칙의 설명과 목적
- **Fix**: 정확한 코드 변경 또는 실행할 명령 (에이전트가 직접 실행 가능)
- **Docs**: 관련 문서 참조 링크

출력 형식은 JSON으로 구조화되어 LLM 컨텍스트 주입에 최적화되어 있습니다.

### 6계층 아키텍처 모델

고정된 6계층 모델로 레이어 배치의 모호성을 제거합니다:

```
Layer 1: Types    — 순수 타입 정의 (런타임 코드 없음)
Layer 2: Config   — 설정 로딩, 환경 변수, 피처 플래그
Layer 3: Repo     — 데이터 접근, 외부 API 클라이언트
Layer 4: Service  — 비즈니스 로직, 유스케이스
Layer 5: Runtime  — 앱 부트스트랩, 미들웨어, DI 컨테이너
Layer 6: UI       — 컴포넌트, 페이지, CLI 핸들러
```

의존성은 **하향만 허용**: 상위 번호 레이어가 하위 번호 레이어를 임포트할 수 있지만, 역방향은 금지됩니다. 상세 내용은 `docs/layers.md`를 참조하세요.

### Providers 패턴

횡단 관심사(로깅, 인증, 텔레메트리, 피처 플래그)를 단일 `Providers` 인터페이스로 접근합니다:

```typescript
// 모든 파일에서 동일한 방식으로 접근
import { providers } from '../providers';
const { logger, auth, telemetry } = providers;
```

직접 라이브러리를 임포트하는 대신 Providers를 통해 접근하면:
- 구현체 교체가 한 곳에서 가능
- 테스트에서 모든 횡단 관심사를 한 번에 모킹 가능
- 새 개발자가 사용 가능한 서비스를 즉시 파악 가능

`/arch providers` 명령으로 Providers 패턴을 자동 설정할 수 있습니다.

### MCP 서버 도구

MCP 서버가 활성화되면 코딩 중 실시간으로 아키텍처 규칙을 검증할 수 있습니다:

| 도구 | 설명 |
|------|------|
| `check_layer_violation` | 임포트가 레이어 규칙을 위반하는지 확인 |
| `get_layer_rules` | 파일/모듈의 레이어 설정 및 허용 임포트 조회 |
| `check_module_boundary` | 모듈 간 의존성이 허용되는지 검증 |
| `suggest_correct_layer` | 타입/함수/파일이 속해야 할 레이어 제안 |
| `get_architecture_map` | 프로젝트 전체 아키텍처 맵 반환 |
| `lint_file` | 단일 파일에 대한 하네스 린팅 실행 |

상세 명세는 `mcp/server-spec.md`를 참조하세요.

## CI/CD 연동

```bash
# pre-commit 훅 (exit code: 0=OK, 1=errors, 2=config error)
/arch-check --severity error

# 베이스라인 기반 (레거시 프로젝트용)
/arch-check --baseline          # 현재 위반 사항 저장
/arch-check --compare           # 새로운 위반만 리포트
```

## 라이선스

MIT License
