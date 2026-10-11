---
name: ios-dev-orchestrator
description: "iOS 앱 개발 하네스 전체를 조율하는 오케스트레이터. 기술 리서치, 기능 구현, 단위테스트 작성, QA 검증, 개발 문서 작성, 커밋 메시지 작성, PR 작성을 담당하는 7개 전문 에이전트(ios-researcher, ios-developer, ios-unit-tester, ios-qa, ios-doc-writer, git-commit-writer, pr-writer) 중 필요한 조합을 라우팅한다. '기능 개발해줘', '이 화면 만들어줘', 'QA 해줘', '단위테스트 작성해줘', '문서 작성해줘', '커밋 메시지 써줘', 'PR 만들어줘', '이 라이브러리 조사해줘', 'SDK 붙이는 법 알아봐줘', '기술 검토해줘' 등 iOS 개발 워크플로우 관련 요청이면 반드시 이 스킬을 사용. 후속 작업(이전 구현 수정, 재검증, 커밋만 다시, PR 본문 보완, 테스트 보완)에도 사용."
---

# iOS Dev Orchestrator

iOS 앱 개발의 7개 전문 에이전트를 요청 범위에 맞게 라우팅하고 조율하는 통합 스킬.

## 실행 모드: 하이브리드

이 하네스는 두 가지 성격의 작업을 다룬다: (1) 실시간 피드백 교환이 품질을 좌우하는 "구현→테스트→검증" 루프, (2) 완성된 결과물을 읽고 독립적으로 산출물을 만드는 단발성 작업(문서/커밋/PR). 모든 요청에 팀을 구성하면 "커밋 메시지만 써줘" 같은 요청에도 팀 생성 오버헤드가 발생하므로, 요청 범위에 따라 모드를 분기한다.

| 시나리오 | 모드 | 이유 |
|---------|------|------|
| 기능 구현 (+테스트+QA 포함) | 에이전트 팀 | 구현자↔테스터↔QA 간 실시간 피드백 루프로 재작업 최소화 |
| 단일 작업 (커밋/PR/문서/테스트만/QA만) | 서브 에이전트 | 결과 전달만 필요, 팀 통신 불필요 |

## 에이전트 구성

| 에이전트 | 타입 | 역할 | 스킬 | 출력 |
|---------|------|------|------|------|
| ios-researcher | 커스텀 | 서드파티 SDK/API 기술 리서치 (context7 우선, 웹 검색 보완) | tech-research | 조사 리포트 |
| ios-developer | 커스텀 | Swift/SwiftUI/UIKit 구현 | ios-development | 변경된 `.swift` 파일 |
| ios-unit-tester | 커스텀 | 단위 테스트 작성 | ios-unit-testing | `*Tests.swift` |
| ios-qa | 커스텀 (general-purpose 성격) | 통합 정합성 검증 | ios-qa | 검증 리포트 |
| ios-doc-writer | 커스텀 | 개발 문서 작성 | ios-doc-writing | README/docs/DocC |
| git-commit-writer | 커스텀 | 커밋 메시지 작성 | git-commit-message | 커밋 메시지, (승인 시) 커밋 |
| pr-writer | 커스텀 | PR 작성 | pr-description | PR 제목/본문, PR URL |

## 워크플로우

### Phase 0: 요청 범위 판단 + 컨텍스트 확인

**1. 요청 범위 분류** — 사용자 요청에서 필요한 에이전트 집합을 판단한다:

| 요청에 포함된 표현 | 필요 에이전트 |
|-------------------|--------------|
| 라이브러리/SDK 조사, API 문서 확인, 기술 검토, 마이그레이션 가이드 조사, 라이브러리 비교 | ios-researcher |
| 기능/화면/버그수정 구현 | ios-developer |
| 단위테스트/유닛테스트 | ios-unit-tester |
| QA/검증/버그 찾기/빌드 확인 | ios-qa |
| 문서/README/아키텍처 문서 | ios-doc-writer |
| 커밋/커밋 메시지 | git-commit-writer |
| PR/풀리퀘스트 | pr-writer |

- **단일 카테고리만 해당** → Phase B(서브 에이전트 단발 실행)로 직행. Phase 2~3(팀 구성) 생략.
- **"기능 개발/구현"이 포함되고 테스트·QA가 명시되지 않음** → 기본값으로 구현+테스트+QA를 함께 수행한다(품질 보장이 기본이므로). 사용자가 "테스트는 나중에", "빠르게 구현만"이라고 명시하면 ios-developer만 단독 실행.
- **구현 요청에 낯선 서드파티 SDK/라이브러리가 명시적으로 포함됨** (예: "결제 SDK 붙여서 구현해줘", "이 라이브러리로 마이그레이션해줘") → ios-researcher를 Phase A 팀에 포함시켜 ios-developer보다 먼저 조사 결과를 만들게 한다. 이미 익숙한 표준 프레임워크(SwiftUI/UIKit/Foundation 등)만 쓰는 요청이면 리서치 없이 진행한다.
- **커밋/PR은 절대 기본값에 포함하지 않는다** — 시스템 정책상 git 커밋/PR 생성은 사용자가 명시적으로 요청했을 때만 수행한다. "구현해줘"만으로 자동 커밋하지 않는다.
- **여러 카테고리가 함께 요청됨** (예: "구현하고 테스트, QA까지, 그리고 커밋 메시지도") → Phase A(팀) 실행 후 Phase B(서브)로 이어서 진행.

**2. 기존 산출물 확인** — `_workspace/` 디렉토리 존재 여부를 확인한다:
- **미존재** → 초기 실행. Phase 1로 진행
- **존재 + 부분 수정 요청** ("이 부분만 다시", "QA 피드백 반영해줘", "커밋 메시지만 다시") → 부분 재실행. 해당 에이전트만 재호출하고 `_workspace/`의 관련 파일만 갱신
- **존재 + 새로운 기능 요청** → 새 실행. 기존 `_workspace/`를 `_workspace_{YYYYMMDD_HHMMSS}/`로 이동한 뒤 새 `_workspace/` 생성

부분 재실행 시 이전 산출물 경로(`_workspace/*`)를 해당 에이전트 프롬프트에 포함해, 기존 결과와 피드백을 읽고 반영하도록 지시한다.

### Phase 1: 준비
1. `_workspace/00_input.md`에 사용자 요구사항과 대상 범위(파일/기능명)를 기록한다.
2. 프로젝트에 기존 Xcode 프로젝트/Swift Package/Tuist 설정이 있는지 확인한다 (`find . -name "*.xcodeproj" -o -name "Package.swift" -o -name "Project.swift"`). 없고 신규 구현이 필요하면 `ios-developer`에게 프로젝트 초기 구조 제안을 함께 요청한다.
3. 프로젝트 `CLAUDE.md`에 `## 프로젝트 컨벤션` 섹션이 있는지 확인한다. 값이 직접 적혀 있으면 각 에이전트 호출 프롬프트에 요약해 포함시킨다. 값 대신 기존 아키텍처 문서의 경로/링크만 있으면 그 문서를 `Read`로 열어 확인한 뒤 요약해 전달한다. 컨벤션 섹션 자체가 없으면, 저장소에 이미 아키텍처/컨벤션 문서(`docs/ARCHITECTURE.md` 등)가 있는지 먼저 찾아 있으면 그걸 따르고, 없으면 `templates/PROJECT_CONVENTIONS.md`를 프로젝트 `CLAUDE.md`에 채워넣도록 사용자에게 안내한다 — 이번 실행은 에이전트들이 기존 코드 탐색 또는 기본값(SwiftUI+MVVM)으로 진행하되 그 가정을 최종 보고에 명시한다.

---

## Phase A: 구현 + 테스트 + QA (에이전트 팀 모드)

기능 구현이 포함된 요청일 때 실행한다. 단위테스트/QA가 요청 범위에 없으면 해당 팀원을 팀 구성에서 제외한다.

### Phase A-2: 팀 구성

```
TeamCreate(
  team_name: "ios-dev-team",
  members: [
    { name: "ios-researcher", agent_type: "ios-researcher",   // 낯선 SDK/라이브러리가 있을 때만 포함
      prompt: "{조사 대상 SDK/라이브러리}에 대해 조사하라. 결과를 ios-developer와 리더에게 전달하라." },
    { name: "ios-developer", agent_type: "ios-developer",
      prompt: "{사용자 요구사항 요약}. ios-researcher가 팀에 있으면 그 조사 결과를 먼저 확인한 뒤 구현하라. 구현 완료 후 변경 파일 목록과 공개 인터페이스를 ios-unit-tester, ios-qa, 리더에게 전달하라." },
    { name: "ios-unit-tester", agent_type: "ios-unit-tester",
      prompt: "ios-developer의 구현이 완료되면 해당 공개 인터페이스에 대한 단위 테스트를 작성하고 실행하라." },
    { name: "ios-qa", agent_type: "ios-qa",
      prompt: "ios-developer의 구현과 ios-unit-tester의 테스트 결과를 대상으로 통합 정합성을 검증하라. 문제 발견 시 ios-developer에게 파일:라인과 함께 구체적 수정을 요청하라." }
  ]
)
```

(단위테스트/QA/리서치가 요청 범위 밖이면 해당 멤버를 제외한다. ios-developer 혼자만 남으면 팀 대신 서브 에이전트로 단독 호출한다.)

### Phase A-3: 작업 등록

```
TaskCreate(tasks: [
  { title: "기술 리서치", description: "{조사 대상}", assignee: "ios-researcher" },   // 리서치가 필요할 때만
  { title: "기능 구현", description: "{요구사항 상세}", assignee: "ios-developer", depends_on: ["기술 리서치"] },   // 리서치 없으면 depends_on 생략
  { title: "단위 테스트 작성", description: "구현된 공개 인터페이스 테스트", assignee: "ios-unit-tester", depends_on: ["기능 구현"] },
  { title: "통합 정합성 검증", description: "경계면 교차 비교 + 빌드/테스트 실행", assignee: "ios-qa", depends_on: ["기능 구현", "단위 테스트 작성"] }
])
```

### Phase A-4: 생성-검증 루프

팀원들이 자체 조율하며 작업한다. `ios-qa`가 문제를 발견하면 `SendMessage`로 `ios-developer`에게 구체적 수정을 요청하고, 리더는 이를 새 작업(`재작업: {항목}`, `depends_on` 없음, `assignee: ios-developer`)으로 등록해 추적한다.

**재시도 한도: 최대 2회.** 2회 재작업 후에도 동일 문제가 해결되지 않으면 루프를 중단하고, 미해결 항목을 최종 리포트에 명시한 채 다음 Phase로 진행한다 (무한 루프 방지).

**산출물 저장:**
| 팀원 | 경로 |
|------|------|
| ios-researcher (있을 때) | `_workspace/00_research_{topic}.md` (조사 리포트) |
| ios-developer | `_workspace/01_developer_summary.md` (변경 파일 목록 + 설계 노트) |
| ios-unit-tester | `_workspace/02_tester_report.md` (테스트 파일 목록 + 통과율) |
| ios-qa | `_workspace/03_qa_report.md` (통과/실패/미검증) |

리더는 `TaskGet`으로 진행률을 모니터링하고, 팀원이 유휴 상태가 되면 알림을 받아 필요 시 개입한다.

### Phase A-5: 팀 정리

모든 작업 완료(또는 재시도 한도 도달) 후:
1. `SendMessage`로 팀원들에게 종료를 알린다.
2. `TeamDelete`로 팀을 정리한다.
3. `_workspace/`는 보존한다 (문서화/커밋/PR 단계에서 참조).

---

## Phase B: 단발 작업 (서브 에이전트 모드)

문서 작성/커밋/PR 작성, 또는 요청 범위가 단일 카테고리일 때 실행한다. Phase A를 거쳐 왔다면 그 산출물(`_workspace/01~03_*`)을 입력으로 사용한다.

| 조건 | 호출 |
|------|------|
| 리서치/기술 검토만 요청 | `Agent(name: "ios-researcher", subagent_type: "ios-researcher", prompt: "{조사 대상}")` → `_workspace/00_research_{topic}.md` |
| 문서 작성 요청 | `Agent(name: "ios-doc-writer", subagent_type: "ios-doc-writer", prompt: "{대상 범위}. Phase A 산출물이 있으면 _workspace/01_developer_summary.md를 함께 참고하라.")` → `_workspace/04_doc_writer_output.md`에 작성 파일 목록 저장 |
| 커밋 메시지/커밋 요청 | `Agent(name: "git-commit-writer", subagent_type: "git-commit-writer", prompt: "{대상 범위}. 실제 커밋 실행은 사용자가 명시적으로 요청한 경우에만 수행하라.")` → `_workspace/05_commit_message.md` |
| PR 작성 요청 | `Agent(name: "pr-writer", subagent_type: "pr-writer", prompt: "{대상 범위}. _workspace/03_qa_report.md(있으면)를 Test plan 작성에 참고하라.")` → `_workspace/06_pr_output.md` |
| 단위테스트만 요청 | `Agent(name: "ios-unit-tester", subagent_type: "ios-unit-tester", ...)` |
| QA만 요청 | `Agent(name: "ios-qa", subagent_type: "ios-qa", ...)` |
| 구현만 요청 (테스트/QA 제외 명시) | `Agent(name: "ios-developer", subagent_type: "ios-developer", ...)` |

여러 작업이 동시에 요청되면(예: "문서랑 커밋 메시지 둘 다") 의존성이 없는 한 단일 메시지에서 병렬 호출한다 (`run_in_background: true`).

**중요 — 실행 순서 의존성:** `git-commit-writer`는 `ios-qa` 검증(수행했다면)이 통과된 이후에 실행한다. `pr-writer`는 `git-commit-writer`의 커밋이 완료된 이후에 실행한다. 이 순서를 지키지 못할 상황(예: QA 없이 바로 PR 요청)이면 사용자에게 건너뛴 단계를 알린다.

### Phase B 통합
1. 각 서브 에이전트의 반환값과 `_workspace/*` 파일을 수집한다.
2. 사용자에게 결과를 요약 보고한다.

## 데이터 흐름

```
[리더] Phase 0: 범위 판단 ──┬─ 단일 카테고리 ──────────────→ Phase B (서브)
                            └─ 구현 포함 ──→ (낯선 SDK면 ios-researcher 선행) ──→ Phase A (팀) ──→ _workspace/00~03 ──→ Phase B (서브: 문서/커밋/PR)
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| ios-researcher가 신뢰할 만한 출처를 못 찾음 | "확인 안 됨"으로 리포트에 명시하고 다음 Phase 진행. ios-developer에게는 불확실성을 알리고, 필요하면 사용자에게 직접 확인을 요청 |
| ios-developer 구현 실패/중단 | 리더가 감지 → 상태 확인 → 재시작. 실패 지속 시 사용자에게 알리고 부분 구현 상태로 보고 |
| ios-qa 재작업 루프 2회 초과 | 루프 중단, 미해결 항목을 리포트에 명시하고 다음 Phase 진행 |
| 서브 에이전트 1개 실패 | 1회 재시도. 재실패 시 해당 산출물 없이 진행하고 최종 보고에 누락 명시 |
| git 저장소 아님 (commit/PR 요청 시) | git-commit-writer/pr-writer가 사용자에게 `git init` 필요 여부 확인 요청 (자동 init 금지) |
| gh 인증/원격 없음 (PR 요청 시) | PR 생성 대신 제목/본문 초안만 제공 |
| Xcode 프로젝트/시뮬레이터 없어 빌드·테스트 실행 불가 | 정적 검토로 대체하고 리포트에 "런타임 미검증" 명시 (결과를 성공으로 포장하지 않음) |

## 테스트 시나리오

### 정상 흐름 (전체 파이프라인)
1. 사용자: "프로필 화면에 아바타 업로드 기능 추가해줘, 테스트랑 QA까지 해줘"
2. Phase 0: 구현+테스트+QA 범위로 판단, `_workspace/` 신규 생성
3. Phase A: 3인 팀 구성 → ios-developer 구현 → ios-unit-tester 테스트 작성 → ios-qa 검증에서 경계면 이슈 1건 발견 → SendMessage로 ios-developer에게 재작업 요청 → 1회 재작업 후 통과
4. Phase A-5: 팀 정리, `_workspace/01~03` 보존
5. 사용자가 "커밋 메시지도 써줘" 후속 요청 → Phase 0에서 부분 재실행(단일 카테고리)으로 판단 → Phase B에서 git-commit-writer 단독 호출
6. 결과: 구현 완료 + 테스트 통과 + QA 리포트 + 커밋 메시지 초안

### 에러 흐름
1. 사용자: "결제 로직 리팩토링하고 PR까지 만들어줘"
2. Phase A에서 ios-qa가 강제 언래핑으로 인한 크래시 위험 발견 → ios-developer에게 재작업 요청 → 수정 후 재검증 통과
3. Phase B에서 pr-writer 호출 시 원격 저장소 미설정 확인
4. pr-writer가 PR 생성 대신 제목/본문 초안만 제공하고, 사용자에게 원격 저장소 설정 필요함을 안내
5. 최종 보고에 "PR 미생성 — 원격 저장소 설정 필요"를 명시

### 리서치 포함 흐름
1. 사용자: "OO 결제 SDK 붙여서 결제 화면 구현해줘"
2. Phase 0: 낯선 서드파티 SDK가 명시되어 ios-researcher를 팀에 포함하기로 판단
3. Phase A: ios-researcher가 context7로 SDK 문서를 조회해 `_workspace/00_research_payment-sdk.md` 작성 → ios-developer가 이를 참고해 구현 → 이후 단위테스트/QA는 정상 흐름과 동일
4. context7에 해당 SDK가 색인되어 있지 않으면 ios-researcher가 WebSearch로 공식 문서를 찾아 보완하고, 그마저 불확실하면 리포트에 "확인 안 됨" 항목으로 남긴 채 진행
