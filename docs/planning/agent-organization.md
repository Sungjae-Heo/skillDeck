# SkillDeck 구현 계획: 에이전트 조직 · 모델 배정 · 예상 기간

- 기준 문서: [`Resource/SkillDeck_PRD_v0.2.docx`](../../Resource/SkillDeck_PRD_v0.2.docx)
- 작성일: 2026-09-27
- 상태: 초안 (7장의 결정 사항 확정 전)

---

## 1. PRD 분석 요약

### 1.1 MVP 범위

PRD의 Phase 1~3 전체와 Phase 4·5 일부(Local/Git URL Import, Structural Validation, Basic Security Scan, Examples/Evals 감지)가 MVP입니다. .NET/C# 모듈 12개로 구성되며, 코드와 테스트를 합쳐 약 2.5~3.5만 줄 규모로 추정합니다.

### 1.2 모듈 의존 구조

```
Domain·계약 ─┬─ Hub/Storage ──┐
             ├─ Adapters ─────┼─ Projection/Manifest ─ Transaction/Safety ─ CLI ─┐
             ├─ Sources ──────┤                                                 ├─ Desktop(WinUI)
             └─ Validation/Security ───────────────────────────────────────────┘
```

- **가장 중요한 경로:** Adapter → Projection → Transaction. 이 부분이 틀리면 제품 전체가 흔들립니다.
- **작업량이 가장 많은 부분:** WinUI 화면

---

## 2. 개발 환경 점검 결과 (2026-09-27)

| 항목 | 상태 | 영향 |
|---|---|---|
| .NET | 런타임(10.0.12)만 있고 **SDK 없음** | Wave 0에서 SDK 설치 필요 |
| Windows Developer Mode | 켜짐 | 관리자 권한 없이 Symlink 생성 가능 |
| Agent CLI | claude, codex, opencode 1.18.32, agy 1.1.9, gemini 0.43 모두 설치됨 | 4개 Agent의 스킬 인식을 실측할 수 있음 |
| 기존 스킬 폴더 | `~/.claude/skills` 126개, `~/.agents/skills` 140개, `~/.codex/skills` 0개(폴더만 있음), `~/.config/opencode/skills`·`~/.gemini/config/skills` 없음 | Migration 실데이터로 쓸 수 있음. PRD 3.1 경로표와 차이 있음 |
| 프로젝트 위치 | OneDrive 폴더 안 | bin/obj 동기화, 링크(reparse point) 처리 문제 가능 |

---

## 3. 리스크

1. **Junction/Symlink를 각 Agent가 실제로 인식하는지 확인되지 않았습니다.** PRD에도 검증 내용이 없고, Projection 설계 전체가 여기에 달려 있습니다. 코드를 쓰기 전에 실측해야 합니다.
2. **개발 중에 실제 스킬 폴더가 망가질 수 있습니다.** `~/.claude/skills`와 `~/.agents/skills`는 개발을 진행하는 에이전트가 직접 쓰는 폴더입니다.
   - 설계 단계에서 Hub·Agent 경로의 기준 폴더를 바꿔 끼울 수 있게 만듭니다.
   - 자동 테스트는 모두 테스트용 격리 폴더에서만 돌립니다.
   - 실제 환경에서의 Migration은 사용자가 직접 수행합니다.
3. **PRD 3.1 경로표를 다시 확인해야 합니다.** Adapter Definition을 데이터 파일로 두는 설계는 유지하고, 실제 경로 값은 참고자료 R1~R4로 재확인합니다.
4. **WinUI 3는 에이전트가 결과를 확인하기 어렵습니다.** 웹처럼 DOM을 검사할 수 없어 스크린샷에 의존해야 합니다.
   - WinUI 전용 에이전트를 새로 정의합니다.
   - MVVM 구조로 만들어 ViewModel 단위 테스트가 가능하게 합니다.
5. **개발 환경이 아직 준비되지 않았습니다.** .NET SDK가 없고, 프로젝트가 OneDrive 안에 있습니다.

---

## 4. 에이전트 조직

### 4.1 운영 원칙

- **총괄·통합은 메인 세션이 맡습니다.** 서브에이전트는 다른 서브에이전트를 호출할 수 없습니다.
- **동시 실행은 최대 4개로 제한합니다.** 모든 모듈이 `.sln` 하나와 `Domain`을 공유하므로 병렬을 늘릴수록 병합 충돌 비용이 커집니다.
- 각 에이전트는 **별도 git worktree**에서 작업합니다.
- **구현과 검증은 서로 다른 계열 모델이 맡습니다.** (5장 참고)

### 4.2 Wave 구성

**Wave 0 — 설계와 사전 실측 (순차 진행)**

| 역할 | 산출물 |
|---|---|
| 설계 | 솔루션 뼈대, Domain 엔티티, `IAgentAdapter`/`IFileSystem`/`ITransaction` 인터페이스, YAML 스키마 3종(manifest/metadata/adapter), 기준 폴더 교체 구조, 테스트할 까다로운 상황 목록 |
| 설계 교차 검토 | 다른 계열 모델이 설계의 맹점 점검 |
| 사전 실측 | R1~R4 문서 검증, 링크 방식(junction/symlink/copy)별 4개 Agent의 스킬 인식 여부 확인 → AUTO 모드 기본값 결정 |

🔒 **사용자 승인 후 다음 단계로 진행**

**Wave 1 — Core Engine (병렬 3개 + 테스트 1개)**

| 역할 | 담당 |
|---|---|
| A. Hub·Storage | Scanner, Content Hash, metadata/provenance 입출력, SQLite(손상 시 재구성 가능) |
| B. Adapters·Projection | Adapter 4종, Junction/Symlink/Copy, `.agents` 공유 최적화, Diff/Plan |
| C. Manifest·CLI | `.skilldeck.yaml`, Profiles, scan/init/enable/disable/diff/apply/doctor |
| T. 테스트 | 테스트용 가짜 폴더 구조, 설계 인터페이스 기준 테스트를 구현보다 먼저 작성 |

**완료 조건:** 테스트용 격리 폴더에서 CLI로 scan → enable → diff → apply가 동작 (PRD Phase 1 완료)

**Wave 2 — Safety, Supply, Quality (병렬 3개 + 리뷰어 1개)**

| 역할 | 담당 |
|---|---|
| D. Safety | Transaction/Rollback, Backup/Restore, Reset & Rebuild, Duplicate/Conflict/Drift, Unmanaged 보호 |
| E. Supply | Local/Git URL Import, Quarantine 파이프라인, SPDX 라이선스 감지, Provenance |
| F. Quality | Validation L1~L3, 정적 Security Scan, Examples/Evals 감지, Trust 상태 |
| R. 공격형 리뷰 | D의 삭제·롤백 경로만 집중해서 공격 ("SkillDeck이 만들지 않은 파일을 지우는가?") |

**완료 조건:** 수용 기준 5~10번 테스트 통과

**Wave 3 — WinUI Desktop (병렬 2개 + 검수 1개)**

| 역할 | 담당 |
|---|---|
| G. Shell·Projects | 앱 뼈대와 메뉴, Dashboard, Projects(Preview/Apply), Profiles, Settings. 화면 구조와 데이터 바인딩 방식을 처음 정함 |
| H. Library·Wizard | Library, Skill Detail(10개 탭), Migration Wizard, Backups, Activity, Sources. G가 정한 방식을 반복 적용 |
| V. 화면 검수 | 스크린샷을 명세와 비교 |

- G는 Wave 2가 진행되는 동안 임시 서비스(mock)를 붙여 먼저 시작할 수 있습니다.
- **완료 조건:** GUI와 CLI가 같은 Core Engine을 사용 (수용 기준 12번)

**Wave 4 — 통합과 검증**

- E2E: 실제 claude/codex/opencode/agy로 Projection 인식 확인, Skill 100개 이상에서 성능 확인
- 사용자 직접: 스냅샷을 뜬 뒤 실제 환경에서 Migration 수행
- 코드 리뷰, 보안 리뷰, 문서(README, CLI 레퍼런스, THIRD_PARTY_NOTICES)

---

## 5. 모델 배정

### 5.1 사용 가능한 모델

| 공급원 | 모델 | Claude Code에서 부르는 방법 |
|---|---|---|
| Claude | Fable 5.1, Opus 5.5, Sonnet 5, Haiku 4.5 | 기본 서브에이전트 |
| opencodex → GPT | GPT-6 Astra(최상위 등급), Sol(코딩 주력), Luna(경량) | `ocx-*` 서브에이전트 |
| opencodex → OpenCode Go | kimi-k3, kimi-k2.7-code, glm-5.3, qwen3.8-max 등 40여 개 | opencodex 서브에이전트 목록에 추가 |
| Antigravity | Gemini 3.1 Pro, Gemini 3.6 Flash, Claude 4.6, gpt-oss-120b | `agy -p --model …` 백그라운드 실행 |

Claude API 가격 (입력/출력, 100만 토큰당)

| 모델 | 가격 | 특징 |
|---|---|---|
| Fable 5.1 | $10 / $50 | 가장 강력함. 어려운 작업은 한 턴에 몇 분씩 걸림 |
| Opus 5.5 | $4 / $20 | Fable의 약 40% |
| Sonnet 5 | $2 / $10 | Opus 5.5의 절반 |
| Haiku 4.5 | $1 / $5 | 컨텍스트 200K (나머지는 1M) |

### 5.2 배정 원칙

1. **상위 모델:** 실수가 늦게 드러나고 고치는 비용이 큰 곳에 씁니다. 설계, Projection, Safety, WinUI 기본 틀이 해당합니다.
2. **Sonnet급:** 명세가 분명하고 테스트가 실수를 잡아주는 곳에 씁니다.
3. **Haiku급:** 읽기와 정리 작업에 씁니다.
4. **구현과 검증은 서로 다른 계열이 맡습니다.** Claude가 짠 핵심 코드는 GPT-6가 리뷰하고, 다른 계열이 짠 코드는 Claude의 테스트로 검증합니다.

### 5.3 권장안 A: Claude + GPT

| 단계 | 역할 | 모델 | 이유 |
|---|---|---|---|
| 전체 | 메인 (총괄·통합) | Opus 5.5 | 모든 변경 사항을 보고 판단 |
| W0 | 설계 | **Fable 5.1** | 쓰는 토큰은 적은데 영향은 가장 큼 |
| W0 | 설계 교차 검토 | **GPT-6 Astra** | 설계가 틀리면 전부 틀리므로 다른 계열이 한 번 더 확인 |
| W0 | 사전 실측 | Sonnet 5 (각 CLI는 가장 싼 모델로 실행) | 실측은 "스킬이 보이는가"만 확인하므로 모델 품질과 무관 |
| W1 | A. Hub·Storage | Sonnet 5 | |
| W1 | B. Adapters·Projection | **Opus 5.5** | Windows junction, OpenCode 경로 우선순위 등 함정이 많음 |
| W1 | C. Manifest·CLI | Sonnet 5 | |
| W1 | T. 테스트 | Sonnet 5 | |
| W2 | D. Safety | **Opus 5.5** | 실제 파일 삭제와 복구를 다뤄 실수 비용이 가장 큼 |
| W2 | E. Supply | GPT-6 Sol | 두 계정에 부하 분산 |
| W2 | F. Quality | GPT-6 Sol | Security Scan 결과는 "위험 신호"라 오탐 허용 |
| W2 | R. 공격형 리뷰 | **GPT-6 Astra** (추론 수준 xhigh) | 최상위 등급이고 Claude와 다른 계열 |
| W3 | G. Shell·Projects | **Opus 5.5** | WinUI는 결과 확인이 어려워 강한 모델로 재작업을 줄임 |
| W3 | H. Library·Wizard | Sonnet 5 | |
| W3 | V. 화면 검수 | Opus 5.5 (스크린샷 직접 확인) | |
| W4 | E2E | Sonnet 5 | |
| W4 | 코드 리뷰 | Claude가 쓴 코드는 GPT-6 Sol, GPT가 쓴 코드는 Opus 5.5 | 서로 다른 계열이 검토 |
| W4 | 보안 리뷰 | Opus 5.5 | |
| W4 | 문서 | Haiku 4.5 | |
| 상시 | 코드 탐색 | Haiku 4.5 | |

- GPT-6 Sol은 최근 모델이라 실력이 검증되지 않았습니다. E의 첫 태스크 결과로 판단하고, 부진하면 Sonnet 5로 교체합니다.
- opencode와 agy는 개발용 에이전트가 아니라 **SkillDeck이 지원하는 대상**입니다. 사전 실측과 E2E에서는 계속 실행합니다.

### 5.4 대안 B: OpenCode Go와 Antigravity 추가

권장안 A에서 아래 역할만 바뀝니다.

| 역할 | 모델 | 연결 |
|---|---|---|
| A, C | 모델 선발전 1위 (기준 미달이면 Sonnet 5) | opencodex |
| E, F | 모델 선발전 1·2위 | opencodex |
| V. 화면 검수 | Gemini 3.1 Pro (high) | agy |
| 문서 | Gemini 3.6 Flash (medium) | agy |
| 사전 실측·E2E의 CLI 실행 | agy는 gemini-3.6-flash-low, opencode는 deepseek-v4.1-flash | CLI |

**모델 선발전 (Wave 0에 2~3시간 추가)**

- **이유:** OpenCode Go 모델 대부분은 2026년 4~9월 출시 모델이라 실력을 확인할 근거가 가격밖에 없습니다.
- **후보:** GPT-6 Sol, kimi-k2.7-code, kimi-k3, glm-5.3, qwen3.8-max (기준 비교용: Sonnet 5)
- **과제:** 설계 문서를 바탕으로 한 작은 실제 기능(예: Manifest 파서 + Profile 병합 + Diff 계산). 테스트는 Sonnet 5가 먼저 작성하고 모든 후보에게 같은 테스트를 줍니다.
- **평가 기준:**
  - 테스트 통과율
  - 수정을 몇 번 반복했는지
  - 소요 시간
  - 정해진 인터페이스를 임의로 바꿨는지
  - Claude Code 도구를 안정적으로 쓰는지 (파일 편집 실패, worktree 규칙 무시 여부)

**효과:** Claude 사용량이 약 3분의 1 줄어들 것으로 추정합니다(선발전 결과에 따라 달라짐). 일정은 선발전만큼 늘어납니다.

### 5.5 쓰지 않는 모델

| 모델 | 이유 |
|---|---|
| Antigravity의 Claude 4.6 (Sonnet, Opus) | Claude Code의 5세대보다 구형. Claude 사용량 한도가 바닥났을 때만 비상용으로 사용 |
| gpt-oss-120b, GPT-6 Luna, flash 계열 | 문서와 실측 외에는 맡길 역할이 없음 |
| free/preview/alpha 모델, `muse-spark-*-contributor` | 데이터 처리 조건과 안정성을 확인하지 못함 |

---

## 6. 예상 기간

전제: 사용자 1명이 하루 6~8시간씩 검토·승인하며 진행합니다. 시간은 병렬로 돌린 에이전트 시간을 모두 더한 값이 아니라 실제로 흘러가는 경과 시간입니다.

| 단계 | 권장안 A (Claude + GPT) | 대안 B (OpenCode Go·Antigravity 포함) |
|---|---|---|
| W0 환경 준비·설계·사전 실측 | 4~6시간 | 6~9시간 (선발전 포함) |
| W1 Core Engine | 8~12시간 | 8~12시간 |
| W2 Safety·Supply·Quality | 10~14시간 | 10~14시간 |
| W3 WinUI Desktop | 12~20시간 | 12~20시간 |
| W4 통합·E2E·문서 | 6~9시간 | 6~9시간 |
| **에이전트 작업 합계** | **약 40~60시간** | **약 42~64시간** |
| **달력 기준 (검토 포함)** | **12~17 작업일 (약 3~4주)** | **12~17 작업일** |

- **PRD 전체**(Registry/Discover 완성, Compatibility, Phase 6 Advanced)까지 하면 2~3주가 더 걸려 총 5~7주입니다.
- 실제 병목은 에이전트 속도가 아니라 **사용자 검토와 Windows 실측**입니다.
- **일정이 늘어날 수 있는 요인**
  - 사전 실측에서 junction이 인식되지 않으면 COPY 방식 중심으로 다시 설계해야 해서 3~5일이 추가됩니다.
  - WinUI 패키징 방식(MSIX 또는 비패키지)에서 문제가 생기면 2~3일이 추가됩니다.
  - 구독 요금제라면 Claude 에이전트를 여러 개 동시에 돌리다 **사용량 한도에 걸려 기다리는 시간**이 생길 수 있습니다. 권장안 A는 E·F를 GPT-6 Sol에 맡겨 이 부하를 나눕니다.

---

## 7. 시작 전에 정할 것

- [ ] **저장소 위치:** OneDrive 밖(예: `D:\dev\SkillDeck`)으로 옮길지
- [ ] **WinUI 3 유지 여부:** 전체에서 가장 느린 구간(4~6일)
- [ ] **MVP의 Discover 화면 범위:** PRD 12.1에는 Local/Git Import만 있지만 10.1 메뉴에는 Discover가 있음. 현재 "Local/Git 소스 탐색까지만"으로 가정
- [ ] **모델 구성:** 권장안 A(Claude + GPT) 또는 대안 B

---

## 8. 적용 체크리스트

- [ ] .NET SDK 설치 (WinUI 3 빌드 환경 포함)
- [ ] 에이전트 정의의 기본 모델 확인: test-specialist와 database-specialist는 `haiku`로 지정되어 있으므로, 호출할 때 `model: "sonnet"`을 넘겨 올림 (정의 파일보다 우선함)
- [ ] GPT-6 Astra 추론 수준 올리기: 기본값이 `low`이므로 `ocx agent effort`에서 high 이상으로 설정
- [ ] WinUI 전용 에이전트(winui-specialist) 정의
- [ ] (대안 B) opencodex 서브에이전트 목록에 후보 모델 추가
- [ ] (대안 B) agy 실행 시간 제한 늘리기: 기본 5분이므로 `--print-timeout` 설정
