# Codex AGENTS.md 커스텀 인스트럭션 정리

> 원문: [Custom Instructions with AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)

## 1. 개요

- Codex는 작업을 수행하기 전에 `AGENTS.md` 파일을 읽어 프로젝트별 가이드와 컨텍스트를 적용한다.
- 전역(global) 기본값 위에 저장소(repository) 단위 설정을 덮어쓰는 **레이어 구조**로 동작한다.
- 즉, "팀 공통 규칙 + 내 개인 규칙 + 특정 서비스 전용 규칙"을 계층적으로 쌓을 수 있다.

## 2. 탐색(Discovery) 및 우선순위

Codex는 시작할 때 인스트럭션 체인을 구성한다. (실행당 1회. TUI에서는 보통 세션 실행당 1회)

### 2-1. 전역 스코프 (Global scope)

- Codex 홈 디렉토리(기본값 `~/.codex`, `CODEX_HOME` 설정 시 해당 경로)에서 탐색
- `AGENTS.override.md`가 있으면 그것을 읽고, 없으면 `AGENTS.md`를 읽는다
- 이 레벨에서는 **비어있지 않은 첫 번째 파일 하나만** 사용

### 2-2. 프로젝트 스코프 (Project scope)

- 프로젝트 루트(보통 Git 루트)에서 시작해 현재 작업 디렉토리까지 내려가며 탐색
- 프로젝트 루트를 찾지 못하면 현재 디렉토리만 확인
- 각 디렉토리마다 `AGENTS.override.md` → `AGENTS.md` → `project_doc_fallback_filenames`에 지정한 이름 순으로 확인
- **디렉토리당 최대 1개 파일만** 포함

### 2-3. 병합 순서 (Merge order)

- 루트부터 아래 방향으로 파일들을 이어 붙이고, 사이는 빈 줄로 연결
- 현재 디렉토리에 가까운 파일일수록 결합된 프롬프트의 **뒤쪽**에 오기 때문에 앞선 가이드를 덮어쓴다
- 결합된 파일 크기가 `project_doc_max_bytes`(기본 32 KiB)에 도달하면 중단하며, 빈 파일은 건너뛴다

## 3. 전역 가이드 만들기

`~/.codex/AGENTS.md`를 만들어 지속적인 기본값을 설정한다.

```bash
mkdir -p ~/.codex
```

```markdown
# ~/.codex/AGENTS.md

## Working agreements

- Always run `npm test` after modifying JavaScript files.
- Prefer `pnpm` when installing dependencies.
- Ask for confirmation before adding new production dependencies.
```

- 기본 파일을 수정하지 않고 일시적으로 전역 설정을 바꾸고 싶다면 `AGENTS.override.md`를 사용한다.

## 4. 프로젝트 인스트럭션 레이어링

전역 기본값을 상속하면서 저장소 규칙을 유지하려면 저장소 루트에 `AGENTS.md`를 추가한다.

```markdown
# AGENTS.md

## Repository expectations

- Run `npm run lint` before opening a pull request.
- Document public utilities in `docs/` when you change behavior.
```

하위 디렉토리에는 `AGENTS.override.md`로 특화된 규칙을 둘 수 있다. (예: 결제 서비스 디렉토리)

```markdown
# services/payments/AGENTS.override.md

## Payments service rules

- Use `make test-payments` instead of `npm test`.
- Never rotate API keys without notifying the security channel.
```

중첩 오버라이드 동작 확인:

```bash
codex --cd services/payments --ask-for-approval never "List the instruction sources you loaded."
```

## 5. 코드 리뷰 연동

`AGENTS.md`에 `## Code Review Rules` 섹션을 추가하면 Codex의 GitHub 코드 리뷰 동작을 커스터마이징할 수 있다. 규칙은 **간결하고 구체적으로** 작성한다.

```markdown
## Code Review Rules

### Experiment cohorts

- Do not filter treatment comparisons on post-exposure behavior, including conversion or retention.
  Safe path: build cohorts from assignment or exposure; report conversion as an outcome.
```

## 6. 커스터마이징 옵션

`~/.codex/config.toml`에서 대체 파일명과 바이트 제한을 조정할 수 있다.

```toml
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]
project_doc_max_bytes = 65536
```

| 옵션 | 설명 | 기본값 |
| --- | --- | --- |
| `project_doc_fallback_filenames` | `AGENTS.md` 대신 인식할 대체 파일명 목록 | - |
| `project_doc_max_bytes` | 결합된 인스트럭션의 최대 바이트 수 | 32 KiB (32768) |

`CODEX_HOME`을 지정해 프로젝트 전용 Codex 홈을 쓸 수도 있다.

```bash
CODEX_HOME=$(pwd)/.codex codex exec "List active instruction sources"
```

## 7. 검증 및 트러블슈팅

설정이 잘 적용됐는지 확인하는 명령:

```bash
codex --ask-for-approval never "Summarize the current instructions."
```

| 증상 | 원인 및 해결 |
| --- | --- |
| 아무것도 로드되지 않음 | 의도한 저장소에 있는지 확인하고 `codex status`가 기대한 워크스페이스 루트를 보고하는지 확인. 인스트럭션 파일이 비어있지 않아야 한다 (빈 파일은 무시됨) |
| 잘못된 가이드가 적용됨 | 상위 디렉토리나 Codex 홈에 `AGENTS.override.md`가 있는지 확인. 오버라이드 파일을 이름 변경/삭제하면 일반 파일로 폴백된다 |
| 대체 파일명을 무시함 | `project_doc_fallback_filenames`에 오타 없이 등록했는지 확인 후, 설정이 반영되도록 Codex를 재시작 |
| 인스트럭션이 잘림 | `project_doc_max_bytes`를 올리거나, 큰 파일을 중첩 디렉토리로 분할해 핵심 가이드가 살아남게 한다 |
| 프로필 혼동 | Codex 실행 전에 `echo $CODEX_HOME` 확인. 기본값이 아니면 편집한 곳과 다른 홈 디렉토리를 보고 있는 것 |

## 8. 정리

- **파일 우선순위**: `AGENTS.override.md` > `AGENTS.md` > 폴백 파일명
- **스코프 우선순위**: 현재 디렉토리에 가까울수록 우선 (뒤에 붙어 덮어씀)
- **전역은 1개 파일만**, **프로젝트는 디렉토리당 1개 파일씩** 누적
- 설정 변경 후에는 Codex 재시작이 필요할 수 있다
