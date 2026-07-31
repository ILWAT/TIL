# Claude 프론트매터 (YAML)

## 1. Skill (SKILL.md)

- 스킬 파일 최상단에 YAML 블록으로 정의
- 필수 필드: `name`, `description`
- `description`은 언제 이 스킬을 트리거할지 결정하는 핵심 정보라 최대한 구체적으로 작성
- `argument-hint` 등 선택 필드로 인자 힌트 제공 가능
- 이 메타데이터를 기반으로 Claude가 상황에 맞는 스킬을 자동 매칭

## 2. Subagent (.claude/agents/*.md)

- `name`, `description`, `tools`, `model` 필드로 구성
- `description`은 이 에이전트를 언제 호출해야 하는지 판단 기준
- `tools`로 해당 서브에이전트가 쓸 수 있는 도구 범위를 제한 가능
- `model`을 지정하면 해당 서브에이전트만 다른 모델로 실행 가능 (sonnet/opus/haiku 등)
- 지정 안 하면 부모 세션의 모델을 상속

## 3. 공통 규칙

- 각 파일 유형 모두 `---`로 감싼 YAML 블록을 파일 최상단에 위치
- `name`은 보통 파일명(kebab-case)과 일치시키는 것이 관례
- CLAUDE.md 자체는 별도 프론트매터 없이 순수 마크다운으로 작성
- 프론트매터 필드가 곧 Claude Code 런타임이 파일을 해석하는 방식이므로 오탈자에 민감
