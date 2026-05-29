---
description: 외부 보고서에서 ADR-010 사실성 분리로 가치 추출 + 결손 영역 시 /propose-research 자동 dispatch (--loop 옵션 시 수렴까지 무한)
argument-hint: <보고서 .md 절대 경로> [--loop]
---

# /squeeze-report — 외부 보고서 가치 추출

본 명령어는 외부 딥리서치 보고서를 ADR-010 사실성 분리 원칙으로 정독해
**짜낼 수 있는 모든 합리적 적용**을 자동 추출한다. 사용자가 동일 보고서에
대해 "더 적용할 것 없어?"를 반복하면 가치가 점진 발견되는 패턴을 단일
명령어로 추상화한 것.

## 사용법

### 단발 호출 (기본)

```
/squeeze-report /path/to/report.md
```

1회 호출 = 분석 1회 + 판정 1회 + 본문화 + **단계 8 조건부 `/propose-research` 자동 dispatch**.
모든 합리적 후보를 한 번에 처리.

또는 인자 없이 호출 시 가장 최근 처리한 보고서 또는 최근 추가된 사용자
입력 보고서를 자동 식별.

### 단계 8 — `/propose-research` 자동 dispatch (★)

본 보고서 처리에서 **결손 영역**이 발견되면 단계 7 종료 직후 자동으로
`/propose-research` 호출. 사용자가 반복하던 "보강할 지식 PROMPT로 만들어줘"
패턴 자동화.

자동 dispatch 트리거:
- REJECT 사유가 "빈 약속·출처 부재·학파 전무" 등 외부 보강 가치 있을 때
- 사용자 결정 영역 (U1·U2 등) 1건 이상
- DEFER 항목 1건 이상 (재의뢰 가치 있는 영역)

dispatch 차단:
- 영구 거부 (가짜 인용·도그마·ADR 위반) 만 있으면 X (재의뢰 회복 불가)
- 본문화 100% 성공 + DEFER·REJECT 0건일 때 X (가치 추출 완료)
- 사용자 명시 "propose-research 호출 X" 시 X

자세한 절차는 단계 8 본문 참조.

### 루프 호출 (`--loop`)

```
/squeeze-report /path/to/report.md --loop
```

**수렴까지 무한 반복**. 반복 횟수 제한 없음. 짜낼 가치 0이 될 때까지 자동
진행. 안전장치는 **수렴 감지 + 사용자 명시 중단**만.

**루프 모드 종료 조건** (어느 하나라도 충족 시):
1. 분석 에이전트 `nothing_more_to_squeeze: true` 명시
2. 판정 에이전트 `accept: 0 + defer: 0 + new_candidates: 0`
3. 동일 분석 결과 2회 연속 (수렴 감지)
4. 사용자 명시 중단 (Ctrl+C 또는 "중단" 메시지)

**최대 반복 횟수 제한 없음** — 사용자 명시 의도 "짜낼 수 없을 때까지" 그대로 적용.

### 무한 루프의 비용·안전

| 위험 | 완화 |
|---|---|
| 토큰 비용 무한 | Haiku 사용 (1 iter ≈ $0.013) + 각 iteration별 비용 보고 |
| 수렴 미감지 | 동일 후보 ID 2회 연속 강제 종료 |
| 사용자 통제 불가 | iteration마다 진행 보고 + 명시 중단 옵션 |
| Hallucination 신규 후보 무한 | 판정 에이전트 REJECT로 실 본문화 차단 |

각 iteration 시작 시 다음 형식 보고:
```
=== Iteration N ===
이전 누적 비용: $X.XX (Haiku 토큰)
이전 ACCEPT: M건 / REJECT: K건 / DEFER: L건
이전 후보 ID: [N1, N2, N3, ...]
시작 중...
```

사용자는 진행 보고를 보고 언제든 명시 중단 가능. AI는 명시 중단 요청 즉시
현 iteration 완료 후 종료 (강제 중단 X — 부분 본문화 방지).

**언제 `--loop` 사용?**
- 사용자가 명시적으로 "끝까지 짜내" 의도
- 큰 보고서로 1회 정독 누락 위험
- 외부 출처 검증이 새 정보 가져올 수 있는 보고서 (학술 출처 다수)

**언제 단발 호출?**
- 작은 보고서 (1-2 절 이하)
- 이전 적용 이력 있는 보고서 재호출 (단계 0 검사로 충분)
- 비용 최소화 필요 (Haiku 호출 × N회)

## 행동 원칙 — 분석/판정 분리

본 명령어는 **분석 에이전트 ≠ 판정 에이전트** 패턴으로 자기 합리화 위험을
차단한다. 메인 컨텍스트(오케스트레이터)는 다음 역할만 수행:

1. 분석 에이전트 dispatch (Subagent 1)
2. 판정 에이전트 dispatch (Subagent 2, 분석 결과 입력)
3. 판정 통과 후보만 본문화 (메인 컨텍스트가 실제 코드 작업)

**역할 분리**:

| 에이전트 | 권한 | 금지 |
|---|---|---|
| **분석 (Subagent 1)** | 보고서 정독·후보 추출·외부 검증 | 채택 결정·코드 변경 |
| **판정 (Subagent 2)** | ADR 정합성 평가·합리성 검증·거부 사유 명시 | 보고서 재해석·후보 신규 발견 |
| **오케스트레이터 (메인)** | 본문화·커밋·라이브 검증 | 분석·판정 |

이 분리로 단일 에이전트가 "분석하며 동시에 채택 합리화"하는 위험을 차단한다.
ADR-010 사실성 분리의 메타 적용.

### 모델 선택 (비용 최적화 + 보고서 크기별 분기)

| 에이전트 | 모델 | 사유 |
|---|---|---|
| 분석 (Subagent 1) | **claude-haiku-4-5** | 정독·추출·WebFetch — 단순 작업. Opus 1/15 비용 |
| 판정 (Subagent 2) | **claude-haiku-4-5** | ADR 매칭·YAML 평가 — 결정론 작업. Opus 1/15 비용 |
| 오케스트레이터 (메인) | claude-opus-4-7 | 본문화·ADR 작성·면책 설계 — 깊은 추론 필요 |

**보고서 크기별 분기 (★ 본 세션 실 사용 기반)**:

| 보고서 크기 | 처리 방식 | 사유 |
|---|---|---|
| **≤ 200줄** (작은 보고서) | 메인 컨텍스트 (Opus) 직접 분석·판정 | Subagent dispatch 오버헤드 (~150K 토큰 컨텍스트 전송) > Opus 직접 처리 비용 |
| **200~500줄** (중간) | Subagent Haiku 분석/판정 분리 | 표준 패턴 |
| **> 500줄** (큰 보고서) | Subagent Haiku 분석/판정 분리 + 단계별 보고 | 정독 누락 위험 차단 |

분석/판정은 **창의적 합리화가 불필요**한 정형 작업이라 Haiku 충분. 메인은
Opus 4.7로 유지하여 ADR·면책·통합 결정 품질 보장.

Agent dispatch 시 `model: "haiku"` 명시 의무 (≤200줄 보고서는 dispatch 자체 skip 가능).

## 행동 절차

### 단계 0 — 이전 적용 이력 확인

같은 보고서가 이미 `vault/reports/`에 영속화되어 있는지 점검 (환경 독립
표준 도구 사용):

```bash
# Glob 도구 — 보고서 키워드 기반 파일 매칭
# 예: 보고서명에 "face·<도메인 3>" 키워드 있을 시
ls vault/reports/*.md | grep -iE "face|<도메인 3>"

# Grep 도구 — frontmatter applied_to 절 검색
grep -lE "^applied_to:" vault/reports/*.md
```

이미 영속화되어 있으면 해당 reports 노트의 frontmatter `applied_to` +
본문 "추가 본문화" 절을 확인해 **이전 추출 항목 목록** 파악. 없으면 신규
처리로 진행.

### 단계 1 — 분석 에이전트 dispatch (Subagent 1, Haiku)

Task tool로 분석 전용 에이전트 spawn. **모델: claude-haiku-4-5**. 프롬프트:

```
보고서 경로: <path>
이전 적용 이력: <단계 0 결과>

당신은 분석 에이전트입니다. 본 보고서를 정독해 후보를 추출하기만 합니다.
채택 결정·코드 변경·ADR 작성 권한은 없습니다.

### 분석 의무

1. **본문 전체 정독** (한 번에 모든 절·문단)
2. **ADR-010 3등급 사실성 분리**:
   - 🟢 팩트: 출처 검증 가능 후보
   - 🟡 구조: 시스템 설계 명제 후보
   - 🔴 도그마: 검증 불가 + 위해 / 가짜 인용 / 빈 약속
3. **외부 출처 라이브 검증** (WebFetch):
   - KCI 논문 (DBpia·koreascholar URL)
   - 통계청·공공 데이터
   - ISBN·서지 (교보문고·알라딘)
   - 검증 실패 시 가짜 인용 표시
4. **빈 약속 식별**:
   - "다음과 같다" / "JSON 형식 정리한다" 뒤 실제 데이터 부재
5. **본 프로젝트 결손 영역 매칭** (오추정 차단 의무):
   - engine/divination/ + engine/safety/ 기존 모듈 점검
   - **Read·Grep으로 실 코드 직접 확인 의무** (보고서 텍스트만으로 "미구현" 추정 금지)
   - **vault/references/ + vault/templates/ 실 파일 존재 여부 직접 확인** (보고서가 인용한 학술 출처가 이미 영속화되어 있는지)
   - 보고서 권장 vs 현 구현 비교
   - 정량 후보 추출 시 (한자 개수·entries 수 등) 실 JSON 파일을 **직접 카운트 검증** 의무
6. **본문 명시 vs 빈 약속 명확 구분**:
   - 보고서 본문에 텍스트로 명시된 규칙·예시는 후보
   - JSON 표·회귀 데이터셋이 비어있으면 빈 약속

### 본 세션 누적 오추정 패턴 (절대 차단)

다음 패턴이 본 시스템에서 반복 발견됨 — 분석 에이전트는 후보 보고 전
모두 점검 의무:

- saju-app-spec: "name_dueum.py 역매핑 미구현" 추정 → 실제 docstring 명시 구현
- saju-app-spec: "name_gaeja.py target_ohaeng 미반영" 추정 → 실제 라인 명시
- name-sibling-bulyong: "JSON 142자" 추정 → 실제 entries 125자 (메타 키 5개 포함 오산)
- name-aesthetic-data: "vault/references/korean-phonetic-research.md 미존재" 추정 → 실제 KCI 검증 통과 영속화 완료

→ 후보 보고 전 **Read·Grep으로 실 파일·실 코드 직접 확인** 의무.
"보고서 텍스트만으로 추정"은 ADR-010 사실성 분리 정신 위반.

### 출력 형식

```yaml
candidates:
  - id: "C1"
    section: "§2 위치별 음절"
    grade: "팩트"
    evidence: "보고서 본문 '준·윤으로 끝나는' 명시 + 본 시스템 미반영"
    external_verification: "baby-name.kr fetch 결과 사실"
    current_implementation: "engine/divination/name_aesthetic.py 위치 무관만"
    proposed_action: "position_match_score 함수 추가 + 끝 음절 데이터"
  - id: "C2"
    ...

rejected:
  - id: "R1"
    section: "§4 학술 인용"
    reason: "KCI fetch 결과 가짜 — 김진욱·정희원 둘 다 존재 X"
    real_source: "조성문(2025) / 신지영(2010)"
  - id: "R2"
    section: "§3 회귀 50쌍"
    reason: "빈 약속 — 본문에 데이터 없음"

decision_required_by_user:
  - id: "U1"
    section: "..."
    reason: "🔵 사업 단계 (UI/가격) — AI 단독 불가"
```

**금지**: 채택 결정·코드 변경·ADR 작성·본문화. 후보 추출만.
```

### 단계 2 — 판정 에이전트 dispatch (Subagent 2, Haiku)

분석 결과를 입력으로 판정 전용 에이전트 spawn. **모델: claude-haiku-4-5**.
프롬프트:

```
분석 결과: <단계 1 출력>

당신은 판정 에이전트입니다. 분석 후보들을 본 프로젝트 ADR 정합성으로
독립 검증해 채택/거부 판정만 합니다. 분석 재실행·신규 후보 추가 권한
없습니다.

### 판정 의무

각 후보에 대해 다음 ADR 위반 여부 점검:

**핵심 정합 ADR (필수)**:
- ADR-002 (domain school avoidance): 단일 학파 매핑 강요?
- ADR-006 (자문 거절): 의료·법률·금융·결혼·이혼·정신질환 인과 예언?
- ADR-010 (사실성 분리): 가짜 인용·빈 약속·검증 불가 단정?
- ADR-014 (<도메인 1>→<도메인 2> 예외): 16유형 단정?
- ADR-015 (옵션 B 병행): 옵션 A 디폴트 침해?

**sanitize 4중 안전망 (LLM 출력 영역, 2026-05-21~22 신설)**:
- ADR-094 (일반 단정 부사 차단): 반드시·확실히·100%·절대·틀림없이
- ADR-113 (palm 단정 어휘): 이혼·재혼·우울증·정신질환·단명
- ADR-115 (다국어 hallucination): 악센트 부호 라틴 단어 (saudável·élégant 등)
- ADR-116 (face 단정 어휘): 길흉화복·대운·금전수·길운·흉운·흉상이라
- ADR-117 (한국어 문법 중복): 평평한한·차분한한·콧방울 들린 콧방울

추가 합리성 검증:
- 본 프로젝트 트래픽 데이터 없이 추측 작업인가?
- 사용자 결정 영역(🔵)인가?
- 결정론 보장 가능?
- 회귀 테스트 가능?

### 출력 형식

```yaml
verdicts:
  - candidate_id: "C1"
    verdict: "ACCEPT" | "REJECT" | "DEFER"
    adr_compliance:
      ADR-002: "OK"
      ADR-006: "OK"
      ADR-010: "OK (출처 검증됨)"
      ADR-014: "N/A"
      ADR-015: "OK"
    rationale: "본 시스템 결손 영역 + 보고서 본문 명시 + 외부 검증 통과"
    constraints:
      - "사용자 출력에 면책 의무 자동 검증"
      - "회귀 N건 동반"
  - candidate_id: "C2"
    verdict: "REJECT"
    rationale: "ADR-002 학파 회피 위반 — 단일 학파 매핑 강요"

summary:
  total_candidates: N
  accept: M
  reject: K
  defer: L
  should_proceed: true | false
```

**금지**: 분석 재실행·신규 후보 발견·코드 변경·본문화.
```

### 단계 3 — 오케스트레이터 본문화 (메인 컨텍스트)

판정 결과 ACCEPT 후보만 메인 컨텍스트에서 직접 본문화:

- 이전 채택 절차(ADR + 모듈 + 회귀 + vault 영속화) 그대로
- REJECT 사유는 reports 노트의 `permanently_rejected`에 기록
- DEFER는 roadmap 또는 ADR-N "사용자 결정 대기"에 명시

### 단계 4 — 본 프로젝트 ADR 정합성 재확인 (분석/판정 후 추가 검토)

판정이 통과시킨 후보라도 메인 컨텍스트에서 다음 자체 점검:

- 이미 다른 ADR과 정합인지 (예: ADR-002 옵션 A 디폴트 침해 여부)
- 회귀 테스트로 ADR-010 사용자 출력 의무 자동 검증 가능한지
- 결정론 보장 가능한지

오케스트레이터가 추가 우려 발견 시:
- 작은 우려: 본문화 진행 + ADR에 면책 명시
- 큰 우려: 본문화 정지 + 사용자에게 보고

### 단계 5 — 채택 가능 항목 본문화 (오케스트레이터)

판정에서 ACCEPT 받은 항목만 본문화. 각 항목별:

#### 5a. 신규 ADR 작성
- `vault/decisions/ADR-NNN-주제.md` (다음 번호는 INDEX 확인)
- immutable + 한계 명시 + 면책
- 판정 에이전트가 명시한 adr_compliance를 ADR 본문에 인용

#### 5b. 신규 모듈 + 회귀 테스트
- `engine/divination/` 또는 `engine/safety/`
- 결정론 함수 + 면책 자동 포함
- 회귀 테스트 의무:
  - 보고서 정답값 검증 (있을 시)
  - ADR-010 사용자 출력 의무 자동 검증 (인과 표현·면책)
  - 결정론 보장

#### 5c. vault 영속화
- `vault/reports/[주제].md` — 사실성 분리 결과 + 판정 결과 인용
- `vault/done/[주제].md` — 적용 완료 기록
- `vault/references/[출처].md` — 검증된 학술/공식 출처
- INDEX 4종 (decisions·done·reports·references) 갱신

#### 5d. roadmap 정리
- 해당 항목 → done 이동 또는 placeholder 삭제
- 부분 완료 시 라벨 명시

#### 5e. 거부 영역 영구 기록
- 판정 REJECT 결과를 reports 노트의 frontmatter `permanently_rejected`에 기록
- 향후 재호출 시 같은 후보 재추출 방지

### 단계 6 — 커밋 + 푸시 + 라이브 검증

**★ vault/ 경로 의무 분리** (.gitignore 등재 — Obsidian Sync 전용):
- `vault/decisions/ADR-*.md` · `vault/reports/*.md` · `vault/templates/PROMPT_*` —
  **git commit 절대 X** (Obsidian Sync 자동 동기화)
- `engine/` · `web/` · `front/` · `tests/` · `Dockerfile` · `CLAUDE.md` —
  git commit 의무

```bash
# 코드 변경 (engine·web·front·tests) — git commit
git add <engine/... web/... tests/... 코드 변경 파일>
git commit -m "<conventional commit + 출처·검증·회귀 명시>"
git push origin main

# CI 모니터링 (paths-filter engine/·web/·front/ 변경 시 Fly.io 재배포 자동 트리거)
gh run list --limit 1 --json databaseId
gh run watch <id> --exit-status

# Fly.io 라이브 검증
curl -sf https://<your-app>.fly.dev/api/health
```

**vault/ 영속화만 있을 때 (코드 변경 X)**:
- 커밋·푸시 단계 **skip** — Obsidian Sync로 자동 동기화
- CI·라이브 검증 단계 **skip** (코드 변경 X)

본 세션 누적 검증된 패턴 (2026-05-21~22 6회 호출):
- 보고서 본문화 + vault/ ADR + reports만 있을 때 → 커밋 skip
- 보고서 본문화 + engine/ 모듈 신설 + 회귀 테스트 → 커밋 + CI + 라이브 검증

### 단계 6.5 — 루프 결정 (`--loop` 모드 시)

`--loop` 옵션이 있을 때만 실행. **반복 횟수 제한 없음 (무한 루프)**:

1. **수렴 감지** (1차 안전장치): 직전 iteration과 현 iteration의 분석 결과 후보 ID 비교
   - 동일 후보만 추출 → 수렴 → 종료
   - 새 후보 발견 → 다음 iteration 진행
2. **사용자 명시 중단** (2차 안전장치): 사용자가 명시적으로 중단 요청 시 종료
3. **누적 비용 보고**: 각 iteration 시작 시 누적 토큰 비용 사용자에게 보고
4. 위 조건 모두 미충족 → 단계 1부터 재실행 (분석 → 판정 → 본문화)

**최대 반복 횟수 강제 종료 없음**. 사용자 의도 "짜낼 수 없을 때까지" 그대로.

루프 반복 시 분석 에이전트 프롬프트에 다음 추가:
```
이전 iteration 처리 이력: <이전 iteration의 ACCEPT/REJECT/DEFER 명세>
이전 iteration에서 발견한 후보: <ID 리스트>
본 iteration은 ADR-010 양방향 검증으로 정밀 재정독:
- 이전 거부 사항 중 잘못 거부된 것 없는가? (실존 검증 누락)
- 이전 채택 사항 중 누락된 본문화 부분 없는가?
- 새 외부 정보(WebFetch 시점 차이)로 발견 가능한 후보?
```

**무한 루프 시 행동 매트릭스** (수렴 감지 의존):
| iteration | 신규 후보 | 새 ACCEPT | 행동 |
|---|---|---|---|
| 1 | N1~N5 | M | 본문화 + 다음 iteration |
| 2 | N1~N5 동일 | 0 (모두 수렴) | **자연 종료** |
| 2 | N6 신규 | M' | 본문화 + 다음 iteration |
| N | 새 후보 계속 | M'' | 계속 진행 (제한 없음) |
| 사용자 중단 | 무관 | 무관 | 현 iter 완료 후 종료 |

### 단계 7 — "짜낼 가치 0" 정직 판정

다음 조건 충족 시 **종료 후 단계 8 진입**:

- ✅ 보고서 본문 모든 검증 가능 부분 채택 완료
- ✅ 모든 가짜 인용 폐기 + 진짜 출처 영속화
- ✅ 모든 빈 약속 (JSON 표 빈약·"50쌍 다음과 같다" 비어있음) 식별
- ✅ 남은 항목은 사용자 결정 영역(🔵 사업 단계) 또는 운영 데이터 후 재평가

판정 시 다음 형식으로 보고:

```
## 본 보고서 적용 종합

| 영역 | 상태 |
|---|---|
| <부분 1> | ✅ 채택 (커밋 해시) |
| <부분 2> | ❌ 빈 약속 / 가짜 인용 / 영구 거부 |
| <부분 3> | ⏸️ 사용자 결정 / 운영 데이터 후 |

본 보고서로부터 짜낼 가치 더 이상 없음. 종료.
```

### 단계 8 — 조건부 `/propose-research` 자동 dispatch (★ 신규)

본 보고서 처리에서 발견된 **결손 영역**을 후속 딥리서치 PROMPT로 자동 영속화한다.
사용자가 `/squeeze-report` 후 반복하는 "더 정밀한 PROMPT 만들어줘" 패턴 자동화.

#### 8a. 자동 dispatch 트리거 조건 (둘 다 충족 시)

1. **결손 영역 1건 이상 존재**:
   - 판정 REJECT 사유가 "빈 약속·출처 부재·학파 전무" 등 외부 보강 가치 있는 경우
   - 또는 사용자 결정 영역 (U1·U2 등) 1건 이상
   - 또는 DEFER 항목 1건 이상 (재의뢰 가치 있는 영역)

2. **재의뢰 가치 평가 통과**:
   - 영구 거부 (가짜 인용·도그마·ADR 위반) 만 있으면 dispatch X — 재의뢰해도 회복 불가
   - 정보성 ACCEPT만 있으면 dispatch X — 이미 본문화 가치 추출 완료
   - 외부 학술 출처·공식 문서로 채울 수 있는 결손이 있을 때만 dispatch

#### 8b. dispatch 미실행 조건

다음 케이스는 **단계 8 skip**:
- 본 보고서가 이미 `permanently_rejected` 전체 영역만 있을 때
- 본문화 100% 성공 + DEFER·REJECT 0건일 때
- 사용자 명시 의도 "/propose-research 호출 X" 있을 때

#### 8c. dispatch 실행 절차

위 조건 충족 시 사용자에게 명시 보고 후 자동 진행:

```
## /propose-research 자동 dispatch

본 보고서 결손 영역 N건 발견 — 후속 PROMPT 영속화 가치 있음.

대상 결손:
- <REJECT/DEFER/U1·U2 항목 요약>

/propose-research 자동 호출 중...
```

후속 `/propose-research`는 다음 컨텍스트를 받아 실행:
- 본 `/squeeze-report` 처리 결과 (vault/reports/<주제>.md)
- REJECT 사유·DEFER 사유·사용자 결정 영역 명시
- 본 보고서가 인용한 빈 약속·접근 불가 출처 목록

`/propose-research`는 본 컨텍스트로 다음 영역 자동 점검:
- 본 결손 영역이 기존 PROMPT 풀과 중복인지 (Phase B Haiku 검증)
- 본 결손 영역 채울 신규 PROMPT 페어 작성 가치
- 또는 기존 PROMPT 강화 (`related_module`·결손 명세 정정)

#### 8d. dispatch 결과 보고

`/propose-research` 완료 후 본 사용자 보고에 다음 추가:

```
### /propose-research 후속 결과

| 신규 PROMPT | 정정 PROMPT | 영구 기록 |
|---|---|---|
| <목록> | <목록> | <목록> |

추후 사용자가 외부 딥리서치 의뢰 시 신규 PROMPT 페어 사용 → 결과 받으면 /squeeze-report 재호출.
```

#### 8e. 비용

`/propose-research`는 Phase A (Opus 메인) + Phase B (Haiku) + Phase C (Opus 메인) 단계.
1회 자동 dispatch ≈ $0.02 (Haiku 검증 1회 + Opus 합성).

#### 8f. 안전장치

- **반복 호출 차단**: `/propose-research`가 본 호출에서 신규 PROMPT 작성 후 사용자가
  외부 의뢰 결과를 `/squeeze-report`로 재호출 → 그 재호출이 또 단계 8 trigger 가능
  → **양방향 무한 루프 위험**. 차단: 본 `/squeeze-report` 호출이 이미 `/propose-research`
  자동 dispatch 결과 영역을 처리하는 경우 (frontmatter 추적) 단계 8 skip
- **사용자 명시 중단**: 사용자가 "그만"·"propose-research 호출 X" 명시 시 즉시 skip
- **Haiku 비용**: 1회 호출 ≈ $0.02. 1일 100회 호출해도 $2 — 무한 호출 위험 적음

## 안전장치

### 무한 루프 방지

**기본 (단발) 모드**:
- 1회 호출당 발견한 모든 합리적 후보를 **한 번에** 처리
- 단계 5에서 보고서 본문 전체 정밀 정독
- 사용자 재호출은 새 정보 발견 시 가치 있음 (KCI fetch 시점 차이 등)

**`--loop` 모드 (수렴까지 무한)**:
- 최대 반복 횟수 제한 **없음** (사용자 의도 "짜낼 수 없을 때까지")
- 동일 후보 2회 연속 시 즉시 수렴 종료 (1차 안전장치)
- iteration마다 진행 보고 (누적 비용·후보 ID 사용자 공유)
- 각 iteration은 독립 Haiku 호출 (1 iter ≈ $0.013)
- 사용자 명시 중단 가능 (2차 안전장치)

**실 사례 검증** (본 세션 직전 호출):
- saju-mbti-correlation 1차 호출: 학술 4건 가짜 의심으로 전면 거부
- 재호출: KCI 라이브 검증 → 2건 실존 정정
- → 단발 호출의 한계 + 재호출 가치 입증
- → `--loop` 옵션이 이를 자동화

### 거부 영역 영구 기록

ADR-010 거부 사례는 다시 재평가 시 효율 위해 frontmatter에 명시:
```yaml
permanently_rejected:
  - 가짜 인용 (저자·연도 검증 실패)
  - 빈 약속 ("JSON 형식 정리" 뒤 데이터 없음)
  - ADR-002 학파 회피 위반
```

### 사용자 결정 영역 명시

🔵 사업 단계 (UI·가격·결제·마케팅) 또는 명시적 Human Input 대기는 AI 단독
진행 금지. 본 명령어는 그 영역 도달 시 명시 정지.

## 출력 형식

마크다운 보고서:

```
# /squeeze-report 결과 — <보고서명>

## 사실성 분리
- 🟢 팩트 N건: ...
- 🟡 구조 N건: ...
- 🔴 도그마 N건: ...

## 외부 검증 결과 (WebFetch)
- ...

## 본문화 진행
- ADR-NNN <주제>: <설명>
- 신규 모듈: <경로> (회귀 N PASS)
- 신규 데이터: <경로>
- vault: ADR + done + reports + references

## 커밋
- <해시>: <메시지 1줄>

## 라이브 검증
- ...

## 짜낼 가치 종합
| 영역 | 상태 |
| ... | ... |

본 보고서 처리 완료. <종료 사유>.
```

## 주의

- 본 명령어는 **합리적 작업만 자동화**. 비합리한 작업(학설 도그마 채택·UI 결정·가짜 인용 답습)은 자동 거부
- ADR-010·002·006·014·015 정신 유지
- 모든 사용자 출력에 면책 자동 포함 의무
- 회귀 테스트 동반 의무 (코드 변경 시)
