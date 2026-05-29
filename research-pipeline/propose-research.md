---
description: 본 시스템 결손 영역 점검 + 딥리서치 프롬프트 제안 + md 영속화 (squeeze-report 후속 패턴)
argument-hint: (인자 없음 — 자동 시스템 결손 점검)
---

# /propose-research — 딥리서치 프롬프트 제안 + md 정리 자동화

본 명령어는 사용자가 `/squeeze-report` 후 반복하는 **"딥리서치로 조사할 것
프롬프트 제안해 + md도 정리해"** 패턴을 단일 명령어로 추상화한 것.

## 사용법

```
/propose-research
```

인자 없음. 본 시스템 결손 영역 자동 점검 + 신규 프롬프트 후보 추출 +
영속화 + 커밋·푸시.

## 행동 원칙 — Phase A 창의 단독 + Phase B 검증 분리

본 명령어는 **창의 작업(결손 발굴·신규 작성)**과 **사실 검증 작업(경로 실존·중복 검사·삭제 후보 정합성)**이 섞여 있다. 두 작업의 위험 성격이 다르므로 단계별로 분리한다.

| Phase | 모델 | 역할 | 권한 |
|:---|:---|:---|:---|
| **A (창의)** | **Opus 4.7 (메인)** | 결손 발굴·신규 PROMPT 초안·기존 PROMPT 정정 후보 식별 | 본문화·삭제 권한 없음 (초안만) |
| **B (검증)** | **claude-haiku-4-5 (서브에이전트)** | Phase A 산출물의 사실 검증 — 모듈 경로 실존·기존 PROMPT 중복·삭제 후보 모듈 풀 직접 확인·ADR 정합 | 신규 후보 발굴 금지 |
| **C (본문화)** | **Opus 4.7 (메인)** | Phase B 통과 산출물만 vault/templates 작성·rm·commit·push | 검증 우회 금지 |

이유:
- 결손 영역 발굴은 **시스템 전체 구조 이해 + 창의적 추론** 필요 → Phase A는 Opus 단독
- 단계 6 "잘못 추정 프롬프트 제거" + 단계 1 "신규 결손 추출"은 **같은 추정 편향이 양방향 작동**할 위험 → Phase B 독립 Haiku로 차단
- engine/face/ 오추정, dream 결손 오추정, hwapae.py 결손 오추정 같은 **모듈 풀 미확인 추정 패턴**을 Phase B의 결정론 검증(Glob+wc+grep)으로 회수
- `/squeeze-report` 후속 검증과는 별개의 **선제 안전망**

## 행동 절차

### 단계 0 — 본 시스템 결손 영역 자동 점검

오케스트레이터가 다음 시스템 영역 점검:

#### 0a. 명시적 TODO 검색
```bash
# engine/agents/__init__.py 등에서 명시 TODO
grep -rE "TODO|미구현|결손|결락|placeholder|NotImplemented" engine/ --include="*.py"
```

#### 0b. 직전 squeeze-report DEFER 영역 확인
```bash
# 가장 최근 처리된 reports/ 파일의 frontmatter
ls -t vault/reports/*.md | head -3
# deferred_pending_research·deferred_pending_decision·deferred_pending_source 확인
```

#### 0c. 모듈 결손 vs 충실 구조 비교
- engine/divination/ 모듈 풀 (LLM vs 결정론 엔진 분리 정합)
- engine/saju/ 모듈 풀 (만세력·신살·십성·운성 등)
- engine/safety/ 모듈 풀 (가드레일 50+)
- engine/agents/ 모듈 풀 (멀티에이전트 오케스트레이션)

#### 0d. 기존 PROMPT 풀 중복 검사
```bash
ls vault/templates/PROMPT_*.md
```

각 PROMPT의 frontmatter `purpose` + `related_module` 읽어
**신규 후보가 기존과 중복되지 않는지 검증 의무**.

### 단계 1 — [Phase A 창의 / Opus 단독] 결손 영역 후보 추출

> Phase A 작업. 메인 컨텍스트(Opus 4.7) 단독 실행. 본문화 권한 없음 — **초안만**.

본 시스템 정체성 (CLAUDE.md §0 + ADR-007~):
- <도메인 1>·<도메인 4>·<도메인 3>·궁합·<도메인 5>·<도메인 6>·<도메인 7> 운세 SaaS
- 꿈해석 멀티에이전트 (14 핵심 + 6 보조)
- 임상 측정 통합 (BDI-K·CES-D·PSQI·STAI-K·ISI·IRT)
- 50+ 안전 가드레일 (GDPR·DSR·인시던트·SLO)
- 결정론 엔진 + LLM 작문 분리 정책

후보 추출 기준 (우선순위 순):

1. **명시 TODO 직격**: engine/agents/__init__.py 등 명시된 미구현 영역
2. **squeeze-report DEFER 영역 정밀화**: 직전 보고서 처리에서 학술 출처
   부재로 DEFER된 영역 (보고서 본문 임계값 부재 등)
3. **결정론 엔진 결손**: LLM Vision 단독 모듈에 결정론 엔진 부재
   (예: palm_reading.py 외 palm_scoring.py 신설)
4. **학파 명시 채택 패턴 (ADR-015 반복)**: 옵션 A 디폴트 + 옵션 B 명시 채택
5. **사실성 검증 영역 (ADR-010)**: 가짜 인용 가능성이 높은 학설 인용 영역
6. **운영 데이터 의존 영역**: 트래픽 누적 후 학술 근거 단계만 영속화 가능

### 단계 2 — [Phase A 창의 / Opus 단독] 기존 PROMPT 풀 중복 검사 (1차 자가점검)

> Phase A 작업. Opus가 자체적으로 명백한 중복만 제거. 정밀 검증은 Phase B에서 별도 Haiku가 수행.

각 신규 후보에 대해:
1. `vault/templates/PROMPT_*.md` frontmatter 확인 (1차)
2. 명백한 중복: 신규 후보 **초안에서 제외**
3. 새 영역으로 판단: 신규 PROMPT 초안 작성 (단계 3)

### 단계 3 — [Phase A 창의 / Opus 단독] 신규 PROMPT **2종 페어** 작성 (★)

> 각 후보별로 **반드시 2개 파일을 페어로 작성**한다. 메타 노트와 딥리서치 직접 입력본을 분리해야 사용자가 입력본을 그대로 복사·붙여넣기로 외부 딥리서치 도구에 의뢰할 수 있다.

#### 3a. 메타 노트 (`vault/templates/PROMPT_<주제>.md`)

본 프로젝트 컨텍스트·채택 절차·면책을 담은 Obsidian Vault 노트. 형식:

```yaml
---
type: prompt_template
target: deepresearch
purpose: <한 줄 설명>
created: <YYYY-MM-DD>
related_module: <engine/... 경로>
related_adr:
  - ADR-XXX (관련 ADR)
priority: high | medium | low
status: draft
related_report: ../reports/<관련 보고서> (있을 시)
deepresearch_input: PROMPT_<주제>.deepresearch.txt  # ★ 페어 입력본 파일명
post_traffic: true | false  # 운영 데이터 의존 여부
---
```

본문 구조:
1. **사용법**: 본 프롬프트의 결손 영역 명시 + "딥리서치 입력본은 페어 .deepresearch.txt 파일 참조" 명시
2. **결손 영역 표**: 본 프로젝트 현재 상태 vs 결손
3. **본 시스템 채택 절차**: `/squeeze-report` 호출 후 ACCEPT 후보 본문화 경로
4. **면책**: ADR 정합 의무

> 메타 노트에는 딥리서치 의뢰 본문을 **포함하지 않는다**. 입력본은 3b의 별도 파일에만 존재한다.

#### 3b. 딥리서치 직접 입력본 (`vault/templates/PROMPT_<주제>.deepresearch.txt`)

사용자가 외부 딥리서치 도구(Perplexity Deep Research, Gemini Deep Research, ChatGPT Deep Research, Anthropic Claude.ai 등)에 **그대로 복사·붙여넣기**로 의뢰할 수 있는 순수 입력본. **다음 4가지 의무 준수**:

1. **YAML frontmatter 금지** — 일반 텍스트만
2. **Obsidian wiki 링크/마크다운 표 일부 허용** (딥리서치 도구가 마크다운 렌더 가능)
3. **본 프로젝트 명칭·경로·CLAUDE.md 누설 금지** — 외부 일반화 의무
   - ❌ `ADR-\d+`, `engine/divination/`, `engine/saju/`, `<your-app>`, `fly\.dev`, `/squeeze-report`, `/domain-priorities`, `CLAUDE\.md`, `vault/` 등 본 SaaS 식별자
   - ✅ "한국 운세 SaaS", "결정론 모듈 N건", "학파 다원 매트릭스" 같은 일반 추상
4. **자족적(self-contained)** — 외부 에이전트가 본 프로젝트를 모르더라도 의뢰만 보고 조사 가능해야 함

본문 구조 (의뢰서 표준 형식):
1. 한 줄 목적 진술 ("...를 조사하라")
2. 배경·환경 컨텍스트 (외부 일반화)
3. 요구사항 (번호 매김, 각 항목별 "반드시 답할 것" 명시)
4. 출력 형식 (YAML 스켈레톤 또는 마크다운 표)
5. 검증 기준 (출처 URL 의무, 가짜 인용 금지 등)

#### 3c. 검증 자기점검 (단계 4.5 Phase B로 위임 전 1차)

작성 후 다음 항목 자기점검:
- 입력본에 본 SaaS 식별자 (grep으로 `ADR-[0-9]+`, `engine/divination`, `engine/saju`, `<your-app>`, `fly\.dev`, `/squeeze-report`, `/domain-priorities`, `/propose-research`, `CLAUDE\.md`, `vault/`, `BizRouter`, `Gemini` 등) 누설 없는지
- 입력본에 YAML frontmatter 없는지
- 메타 노트가 입력본을 본문에 중복 포함하지 않는지 (DRY)

### 단계 4 — [Phase A 창의 / Opus 단독] 기존 PROMPT 정정 후보 식별

> Phase A 작업. **정정 후보 목록만 작성**. 실제 정정은 Phase B 통과 후 Phase C에서.

본 시스템 실 구현 재점검에서 이전 추정 오류 발견 시 정정 후보로 표시:
- frontmatter `related_module` 정정 필요
- 본문 결손 영역 명세 정정 필요
- 다른 PROMPT가 기존 모듈을 잘못 추정한 경우

예시 (실제 발생): face-shape-classifier 프롬프트가 `engine/face/`로
잘못 명시 → 실제는 `engine/divination/face_reading.py` LLM Vision 기반
→ 경로 + 결손 명세 정정 후보로 표시 (실제 적용은 Phase B 통과 후)

---

### 단계 4.5 — [Phase B 검증 / Haiku 서브에이전트] 사실 검증 분리 호출

> **Phase B 핵심 게이트**. Phase A의 모든 산출물(메타 노트 + 딥리서치 입력본 페어)을 별도 Haiku 에이전트로 독립 검증.

추가 검증 항목 (입력본 페어 의무 점검):
- 각 PROMPT가 `.md` + `.deepresearch.txt` 페어로 작성되었는가? (둘 중 하나라도 부재 시 REJECT)
- `.deepresearch.txt`에 본 프로젝트 식별자 누설 없는가? (grep 패턴: `step0\d\d`, `\.ps1`, `\.claude/`, `CLAUDE\.md`, 기타 본 프로젝트 고유 명칭)
- `.deepresearch.txt`에 YAML frontmatter 없는가?
- `.deepresearch.txt`가 자족적인가? (외부 에이전트가 본 프로젝트 컨텍스트 없이 의뢰만으로 조사 가능한가)

Agent dispatch 시 `model: "haiku"` 명시 의무. 검증 에이전트 프롬프트:

```
당신은 검증 에이전트입니다. Phase A 산출물(신규 PROMPT 초안 N건 + 정정 후보 M건 + 삭제 후보 K건)을 독립 검증만 합니다. 신규 후보 발굴·창작·본문화 권한 없습니다.

Phase A 산출물:
- 신규 PROMPT 초안: <목록>
- 정정 후보: <목록>
- 삭제 후보: <목록>

### 검증 의무 (모든 항목별)

1. **모듈 경로 실존 검증** (결정론):
   - 모든 `related_module` 경로를 Glob/Bash로 직접 확인
   - 존재하지 않으면 REJECT
   - 존재하나 빈 placeholder면 LOW_CONFIDENCE

2. **모듈 풀 직접 확인** (오추정 차단 의무):
   - `engine/divination/`, `engine/saju/`, `engine/agents/`, `dream_lex/` 등 풀 Glob + wc -l
   - "결손" 주장 후보의 해당 모듈 풀 라인 수·파일 수 실측
   - 100줄 이상 + 핵심 함수 docstring 명시 → 결손 아님 → REJECT
   - 30줄 미만 + placeholder → 결손 확인 → ACCEPT
   - 삭제 후보도 같은 기준으로 "정말 결손 아님"인지 재검증

3. **기존 PROMPT 풀 중복 정밀 대조**:
   - 모든 `vault/templates/PROMPT_*.md` frontmatter `purpose` + `related_module` 추출
   - 신규 후보와 코사인 유사도 또는 동일 모듈 명시 여부 점검
   - 중복 발견 시 REJECT (또는 기존 보강 권고)

4. **ADR 정합 점검** (핵심 5건 + sanitize 4중 안전망):
   - ADR-002 (도메인 회피): 단일 학파/단정 강요 여부
   - ADR-006 (자문 거절): 의료·법률·금융 인과 예언 여부
   - ADR-010 (사실성 분리): 빈 약속·가짜 인용 가능성
   - ADR-014 (단정 회피): <도메인 2> 단정 등
   - ADR-015 (옵션 병행): 옵션 A 디폴트 침해
   - ADR-094 (단정 부사 sanitize): "반드시·틀림없이·확실히" 차단
   - ADR-113 (palm 어휘 sanitize): <도메인 6> 단정 어휘 차단
   - ADR-115 (다국어 hallucination sanitize): 영어·포르투갈어 누설 차단
   - ADR-116 (face 단정 어휘 sanitize): "대운·금전수·길흉화복" 차단
   - ADR-117 (한국어 문법 sanitize): 어미 중복·단어 중복 차단

5. **자기 합리화 차단 점검** (★ 단계 6 핵심):
   - 삭제 후보 K건이 "Phase A가 새로 만들 신규 N건과 의도적으로 충돌하지 않는가"
   - 즉, Phase A가 자기 신규 작성을 정당화하려 기존 K건을 삭제 후보로 올렸을 가능성
   - 신규-삭제 후보 페어 매칭 후 의심 패턴 REJECT

### 출력 형식 (YAML)

verdicts:
  new_prompts:
    - id: "N1"
      verdict: "ACCEPT|REJECT|LOW_CONFIDENCE"
      module_exists: true|false
      module_pool_lines: <int>
      duplicate_of: null | "PROMPT_X.md"
      adr_compliance: {ADR-002: OK, ...}
      rationale: "..."
  corrections:
    - id: "M1"
      verdict: "ACCEPT|REJECT"
      ...
  deletions:
    - id: "K1"
      verdict: "ACCEPT|REJECT"
      module_pool_actual_state: "..."
      self_rationalization_risk: "LOW|HIGH"
      ...

summary:
  new_accepted: X
  new_rejected: Y
  corrections_accepted: A
  deletions_accepted: B
  deletions_rejected_for_self_rationalization: C
  should_proceed: true|false

**금지**: 신규 후보 발굴·창작·본문화·rm 실행.
```

검증 결과를 `vault/reports/propose-research-verification-<date>.md`에 저장.

### 단계 4.6 — [Phase C 본문화 / Opus 메인] Phase B 결과 수용

Phase B 통과 항목만 본문화:
- ACCEPT 신규: `vault/templates/PROMPT_*.md` + `vault/templates/PROMPT_*.deepresearch.txt` **페어 모두 작성** (둘 중 하나만 작성 금지)
- ACCEPT 정정: 기존 PROMPT 정정 (페어 둘 다 정정 필요 시 둘 다)
- ACCEPT 삭제: rm 실행 (페어 두 파일 모두 삭제)
- REJECT 삭제 (자기 합리화 의심): **삭제 보류** + 의심 사유를 reports에 영속 기록
- LOW_CONFIDENCE: roadmap INDEX의 🟡 검증 보류 섹션에 임시 등재
- 본 프로젝트 식별자 누설 발견 시: 입력본만 재작성 (메타 노트는 본 프로젝트 컨텍스트 보존)

### 단계 5 — [Phase C 본문화 / Opus 메인] roadmap INDEX 갱신

`vault/roadmap/INDEX.md` 🟢 외부 입력 대기 섹션 갱신:
- 신규 항목 추가
- 본 SaaS N 도메인 분리 유지 (<도메인 1>·<도메인 4>·<도메인 3>·궁합·<도메인 5>·<도메인 6>·<도메인 7>·점성술·윷점)
- 우선도 + 결손 영역 명시
- 기존 PROMPT 풀 일관성 유지

### 단계 6 — [Phase C 본문화 / Opus 메인] 삭제 실행 (Phase B 통과분만)

> ★ **자기 합리화 차단 게이트**. Phase B 검증을 통과한 삭제 후보만 실제 rm.

본 시스템 결손이 아닌 영역의 PROMPT가 발견되고 **Phase B에서 self_rationalization_risk: LOW** 판정 시:
```bash
rm vault/templates/PROMPT_<Phase B ACCEPT한 후보>.md
```

예시 (실제 발생): dream-interpretation-keywords 프롬프트가 결손으로
추정됐으나 dream_lex/ 30+ 학파 모듈이 이미 풍부 → Phase B가 모듈 풀 직접 확인 후 ACCEPT → Phase C에서 제거

Phase B가 REJECT한 삭제 후보:
- 삭제 보류
- `vault/reports/propose-research-verification-<date>.md`의 `deletions_rejected` 섹션에 의심 사유 영속 기록
- 다음 호출 시 같은 패턴 반복 방지

### 단계 7 — [Phase C 본문화 / Opus 메인] vault Obsidian Sync 자동 (git commit X)

> **본 SaaS 핵심 정책**: `vault/` 전체는 `.gitignore` 등재됨. ADR·reports·templates·roadmap은 **Obsidian Sync로 자동 영속화**되며 git commit 의무 **없음**.
>
> 본 명령어 산출물(`vault/templates/PROMPT_*.md` + `.deepresearch.txt` + `vault/roadmap/INDEX.md`)은 **vault/ 하위이므로 git commit 절대 금지**. Obsidian Sync가 자동 처리.

확인 명령 (vault git 무시 검증):
```bash
# vault/ 변경 사항이 git status에 노출되지 않는지 확인
git status --short vault/ 2>&1 | head -5
# → 출력이 빈 줄이거나 ignored 표시여야 정상
```

git commit·push 명령 **실행 금지**. 본 명령어는 vault/ 단독 수정이므로 단계 7 자체가 **no-op**.

예외: 만약 `engine/`·`web/`·`front/`·`tests/`·`.claude/`·`CLAUDE.md` 등 **코드 영역**까지 동시 정정한 경우에만 해당 파일만 추려 별도 git commit (PROMPT 페어는 add 대상 X).

```bash
# 코드 영역 정정이 있을 때만:
git add <코드 파일들만>  # vault/ 절대 X
git commit -m "..."
git push origin main
```

### 단계 8 — 사용자 보고 형식

마크다운 보고서:

```
## 정리 결과

### 본 시스템 결손 영역 재점검
| 도메인 | 이전 추정 | 실제 상태 | 결정 |
|---|---|---|---|
| ... | ... | ... | ... |

### 신규 프롬프트 N건
| 우선도 | 항목 | 파일 |
|---|---|---|
| ... | ... | ... |

### 최종 딥리서치 프롬프트 풀 (N건, 도메인 분리, **페어 2종**)
> 본 SaaS N 도메인 (<도메인 1>·<도메인 4>·<도메인 3>·궁합·<도메인 5>·<도메인 6>·<도메인 7>·점성술·윷점) 중 결손이 있는 도메인만 표에 등재. 도메인 분리는 본문화 시점 도메인 풀 정합.

| 도메인 | 우선도 | 항목 | 메타 노트 | 딥리서치 입력본 |
|---|---|---|---|---|
| <도메인 1> | ... | ... | `PROMPT_<주제>.md` | `PROMPT_<주제>.deepresearch.txt` |
| <도메인 2> | ... | ... | ... | ... |

### 커밋
- <해시>: docs(vault) ...

### 추천 우선순위
| 순위 | 항목 | 사유 |
|---|---|---|
```

## 안전장치

### 결손 영역 오추론 방지

이전 호출에서 실 코드 미확인으로 인한 오추론 패턴 발견:
- engine/face/ 경로 오추정 (실제 engine/divination/face_reading.py)
- dream 결손 오추정 (실제 dream_lex/ 30+ 모듈 풍부)
- hwapae.py 결손 오추정 (실제 364줄 구현)

**본 명령어 의무**:
1. 모든 신규 후보에 대해 `engine/divination/`·`engine/saju/`·`engine/agents/`
   실 모듈 풀 직접 확인 (Glob + grep + wc)
2. 기존 PROMPT의 `related_module` 경로 실존 검증
3. 분명한 결손 (모듈 부재)과 LLM/결정론 분리 결손 구분

### 중복 PROMPT 방지

신규 작성 전 기존 PROMPT 풀 frontmatter 의무 확인.
중복 시 기존 보강이 신규 작성보다 우선.

### 사용자 결정 영역 명시

🔵 사업 단계 (UI·가격·결제·마케팅) 또는 명시적 Human Input 대기는
별도 프롬프트로 작성하지 않음 (roadmap INDEX 🔵 섹션에만 추가).

## 출력 원칙

- 결손 영역 발굴 (Phase A) — **창의적 추론** Opus 4.7 단독
- 사실 검증 (Phase B) — **자기 합리화 차단** Haiku 서브에이전트 분리 (★)
- 본문화 (Phase C) — **결정론 절차** Opus 4.7 메인이 Phase B 통과분만 적용
- 사용자 보고는 **검증 가능 형식** — 표 + 파일 경로 + Phase B verdicts 인용

## 주의

- 본 명령어는 `/squeeze-report` 후속에 자주 호출됨 — squeeze-report
  결과의 DEFER 영역을 우선 점검
- 결손 영역이 0건이면 정직히 "본 시스템 현 결손 0건" 보고 + 신규 프롬프트 X
- 기존 PROMPT 풀 일관성 유지 의무 (도메인 분리·우선도)
- 모든 PROMPT는 ADR-002·006·010·014·015 정합 의무 명시
- **Phase B 우회 금지** — Phase A 단독으로 신규 작성·삭제 직접 실행 금지. 모든 본문화는 Phase B 통과 의무
- Phase B 검증 비용 — Haiku 1회 호출 (~$0.005). Phase A 자기 합리화로 잘못 작성/삭제 발생 시 회수 비용보다 저렴
- **★ 페어 작성 의무**: 모든 신규 PROMPT는 메타 노트(.md) + 딥리서치 직접 입력본(.deepresearch.txt) 2종 페어로 작성. 메타 노트만 작성하면 사용자가 본문에서 의뢰 텍스트를 수동 추출해야 하는 결함 (2026-05-17 propose-research 1차 호출에서 발견·정정)
- **★ 입력본 외부 일반화 의무**: .deepresearch.txt에 본 프로젝트 식별자(파일 경로·hook 이름·CLAUDE.md 규칙) 누설 금지. 외부 딥리서치 에이전트가 본 프로젝트를 모르고도 의뢰만으로 조사 가능해야 함
