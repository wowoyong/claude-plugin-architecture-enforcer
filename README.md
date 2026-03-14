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

## 지원 언어

TypeScript, JavaScript, Python, Java, Go, Rust

## 지원 아키텍처 스타일

- **Layered Architecture** — 전통적 3계층 아키텍처
- **Hexagonal Architecture** — 포트 & 어댑터
- **Modular Architecture** — 자기 완결형 모듈
- **DDD with Bounded Contexts** — 도메인 주도 설계

## 설치

```bash
cp -r architecture-enforcer .claude/plugins/architecture-enforcer
```

## 사용법

### 커맨드

```bash
# 아키텍처 규칙 초기화 (대화형)
/arch-init
/arch-init --preset layered
/arch-init --preset hexagonal

# 아키텍처 규칙 검사
/arch-check
/arch-check --fix
/arch-check --diff            # 변경된 파일만 검사
/arch-check --json            # JSON 출력
/arch-check --module src/services

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
│   └── architecture-auditor.md   # 아키텍처 건강 감사 에이전트
└── README.md                     # 이 파일
```

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
