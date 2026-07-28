<div align="center">

# claude-strategic-compact

**논리적인 작업 경계에서 능동적으로 `/compact`를 제안하는 Claude Code skill + hook**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Requires Node.js](https://img.shields.io/badge/node-%3E%3D18-339933?logo=node.js&logoColor=white)](https://nodejs.org)
[![Based on ECC](https://img.shields.io/badge/based%20on-affaan--m%2FECC-blue)](https://github.com/affaan-m/ECC)

한국어 · [English](README.en.md)

</div>

---

Claude Code는 컨텍스트가 다 차면 자동으로 컴팩션을 실행합니다 — 그 시점이 하필 작업 중간이더라도 상관없이요. 이 프로젝트는 그 대신 **정말로 의미 있는 시점**에 수동 `/compact`를 하도록 유도합니다: 리서치가 끝났을 때, 마일스톤을 하나 완료했을 때, 다른 작업으로 전환하기 직전 — 토큰 카운터가 우연히 어떤 선을 넘는 순간이 아니라요.

[affaan-m/ECC](https://github.com/affaan-m/ECC)의 `skills/strategic-compact/`(MIT 라이선스)를 가져와 독립 실행 가능한 형태로 재구성했고, 동작 하나를 바꿨습니다: 원본 훅은 조용한 상태 로그 한 줄만 남기지만, 이 버전은 실제로 행동하라는 지시를 내립니다 — 자세한 내용은 [원본과 달라진 점](#원본과-달라진-점)을 참고하세요.

## 목차

- [동작 방식](#동작-방식)
- [원본과 달라진 점](#원본과-달라진-점)
- [설치](#설치)
- [설정](#설정)
- [범위](#범위)
- [라이선스](#라이선스)

## 동작 방식

`scripts/hooks/suggest-compact.js`는 `PreToolUse` 훅을 통해 모든 `Edit` / `Write` 도구 호출마다 실행되며, 두 가지 신호를 조합합니다.

| 신호 | 발동 조건 | 비고 |
|---|---|---|
| **컨텍스트 크기** (주 신호) | 세션 transcript의 실제 토큰 사용량이 창 크기에 비례한 임계치를 넘음 — 200k 창 기준 160k, 1M 창 기준 250k | 이후 +60k 토큰 성장마다 재발동 |
| **도구 호출 횟수** (보조 신호) | 50회, 이후 25회마다 | 상대적으로 약한 지표 — 큰 파일 몇 개만 읽어도 호출 몇 번 안에 창이 다 찰 수 있어서, 주 신호라기보단 백업용 |

두 신호 중 하나가 발동하면 Claude Code에 지시문이 주입됩니다. [Claude Code 공식 문서](https://code.claude.com/docs/en/commands) 기준으로 `/compact [instructions]`의 인자는 **자동 요약이 무엇을 우선 보존할지 알려주는 짧은 지시문**이지, 미리 써둔 완성된 요약이 아닙니다 — 실제 요약은 Claude Code가 진짜 대화 내용을 보고 직접 생성합니다. 전체 동작 계약, 컴팩션 판단 기준표(구현 중간처럼 컴팩트하면 안 되는 시점 포함), `/compact` 이후 무엇이 남고 무엇이 사라지는지는 [`skills/strategic-compact/SKILL.md`](skills/strategic-compact/SKILL.md)에 정리되어 있습니다.

## 원본과 달라진 점

| | 원본 (ECC) | 이 저장소 |
|---|---|---|
| 설치 대상 | ECC 자체 프레임워크 구조, 전용 인스톨러 필요 | `~/.claude/`에 바로 복사, 인스톨러 불필요 |
| 훅 메시지 | 조용한 상태 로그("컨텍스트가 X% 찼음") | 능동적 지시문: 지금 이 응답에서 바로 말할 것 |
| `/compact` 인자 | 별도 언급 없음 | 문서화된 `/compact` 시맨틱에 맞춰 *focus instruction*으로 명확히 제한 — 서술형 요약 전체가 아님 |

## 설치

```bash
cp -r skills/strategic-compact ~/.claude/skills/strategic-compact
mkdir -p ~/.claude/scripts/hooks ~/.claude/scripts/lib
cp scripts/hooks/suggest-compact.js ~/.claude/scripts/hooks/
cp scripts/lib/*.js ~/.claude/scripts/lib/
```

그다음 [`settings.snippet.json`](settings.snippet.json)의 훅 등록 내용을 `~/.claude/settings.json`의 `hooks.PreToolUse`에 병합하세요 — 이미 다른 `PreToolUse` 훅이 설정되어 있다면 **배열에 추가만 하고 덮어쓰지 마세요.**

`PATH`에 Node.js가 있어야 합니다(훅은 `node` 스크립트입니다). 정상적으로 로드되는지 확인:

```bash
node --check ~/.claude/scripts/hooks/suggest-compact.js
```

## 설정

아래 환경변수는 모두 선택 사항입니다.

| 변수 | 기본값 | 용도 |
|---|---|---|
| `COMPACT_THRESHOLD` | `50` | 도구 호출 횟수 기반 첫 제안까지의 호출 수 |
| `COMPACT_CONTEXT_THRESHOLD` | `160000`(200k 창) / `250000`(1M 창) | 크기 기반 제안이 발동하는 컨텍스트 토큰 수, `0`이면 이 신호를 끔 |
| `COMPACT_CONTEXT_INTERVAL` | `60000` | 제안이 재발동하기까지 필요한 추가 토큰 성장량 |
| `COMPACT_STATE_TTL_DAYS` | `14` | temp 디렉터리의 세션별 상태 파일을 정리하기까지의 일수 |
| `ECC_CONTEXT_WINDOW_TOKENS` | 자동 감지 | `[1m]` 마커가 없는 대형 컨텍스트 창 모델을 위한 명시적 창 크기 오버라이드 |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | — | Claude Code 자체의 창 크기 오버라이드; 폴백으로 사용됨 |

## 범위

이 저장소는 skill과 hook만 담고 있습니다. `scripts/lib/agent-data-home.js`와 `scripts/lib/path-safety.js`는 `scripts/lib/utils.js`가 import 시점에 이를 require하기 때문에(`ECC_AGENT_DATA_HOME` 경로 해석용) 함께 포함됐을 뿐, 그 이상의 용도로 쓰이진 않습니다.

## 라이선스

MIT — [LICENSE](LICENSE) 참고. 원저작물 Copyright (c) 2026 Affaan Mustafa, 수정 부분 Copyright (c) 2026 jelitz.
