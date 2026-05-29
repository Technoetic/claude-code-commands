# /flyio — Fly.io 배포·관리 + 다른 PC 부트스트랩 (통합)

본 명령어는 두 책임을 통합한다:

1. **다른 PC 환경 부트스트랩** — 새 PC에서 본 프로젝트 (<your-app>)
   이어 작업을 즉시 시작 가능하게 환경 점검·의존성 설치·라이브 검증·
   CI 차단 자동 진단
2. **Fly.io 배포·관리 가이드** — CLI·Secrets·Volumes·도메인·SSH·GraphQL

본 시스템은 ADR-056으로 Railway → Fly.io 마이그레이션 완료.
App: `<your-app>`, Region: `nrt` (Tokyo).

## 사용법

```
/flyio              # 다른 PC 부트스트랩 자동 수행 (인자 없을 시)
/flyio deploy       # 배포 실행 (하위 명령어)
/flyio capacity     # 인프라 부하 측정
```

## 다른 PC 첫 실행 (1회 의무)

`.claude/`는 `.gitignore` 처리됨 → 다른 PC에 본 커맨드 부재.
`git clone` 직후 1회 의무:

```bash
git clone https://github.com/Technoetic/<your-app>.git
cd <your-app>

# vault 동기화 (Obsidian Sync 또는 수동) — vault/templates/SLASH_COMMAND_*.md 의무

bash bootstrap-claude-commands.sh   # vault 백업 → .claude/commands/ 복원
# Claude Code 재시작

# 이후:
/flyio   # 환경 자동 점검
```

## 행동 절차 — 부트스트랩 (인자 없을 시)

### 단계 1 — Git 상태 + origin 차이 감지

```bash
git status
git log --oneline -5
git remote -v
git fetch origin main
git log --oneline HEAD..origin/main  # 다른 PC 신규 커밋 자동 감지
```

차이 1건 이상 시:
- 사용자에게 "다른 PC에서 N커밋 추가됨, `git pull` 의도 확인" 명시
- 사용자 명시 후 pull 진행 (자동 X — conflict 위험)

### 단계 2 — 시스템 도구 검증

```bash
python --version       # 3.12+
git --version          # 2.40+
which fly || ls "C:/Users/<user>/.fly/bin/fly"
which gh
which docker
```

누락 시 안내:
```powershell
# Fly CLI (Windows)
iwr https://fly.io/install.ps1 -useb -OutFile fly-install.ps1
powershell -ExecutionPolicy Bypass -File fly-install.ps1

# 기타
winget install Python.Python.3.12
winget install Git.Git
winget install GitHub.cli
```

### 단계 3 — Python 의존성

```bash
pip install -r requirements.txt
pip install pytest playwright
python -m playwright install --with-deps chromium

python -c "import fastapi, uvicorn, anthropic, openai; print('OK')"
python -m pytest --version
```

### 단계 4 — vault 점검

```bash
ls vault/decisions/ | tail -10
ls vault/roadmap/
ls vault/templates/PROMPT_*.md
```

vault 부재 시: Obsidian Sync 또는 별도 동기화 필요 안내.

### 단계 5 — 핵심 ADR 요약

본 시스템 최근 진행 영역 (2026-05-20 시점):
- ADR-052: ES6 module 전환
- ADR-053: photo·i18n·a11y (compliance 100%)
- ADR-054: 인라인 JS 5,656 → 8 그룹 외부화
- ADR-055: main.css 4,157 → 13 영역 분리
- ADR-056: Railway → Fly.io 마이그레이션
- ADR-057: paths-filter + Docker 멀티스테이지 (88MB)

### 단계 6 — 회귀 테스트 (1-2분)

```bash
python -m pytest tests/smoke/ tests/regression/ -m "not live" --tb=no -q
# 예상: 27 passed in ~37s
```

### 단계 7 — Fly.io 라이브 검증

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://<your-app>.fly.dev/api/health
curl -s -o /dev/null -w "%{http_code}\n" https://<your-app>.fly.dev/
```

200 OK 의무.

### 단계 8 — CI/배포 상태 진단

```bash
gh run list --repo Technoetic/<your-app> --limit 5 --json status,conclusion,displayTitle,databaseId
gh secret list --repo Technoetic/<your-app>  # FLY_API_TOKEN 등록 여부
```

**fail 패턴별 자동 진단**:

| 패턴 | 원인 | 자동 해결 |
|---|---|---|
| `Error: no access token available` | FLY_API_TOKEN Secret 부재 | ❌ 사용자 본인 토큰 발급 + 등록 의무 |
| `flyctl deploy failed: build error` | Dockerfile 오류 | 🟡 로컬 빌드 재현 |
| `Verify live did not reach 200` | 앱 기동 실패 | 🟡 `fly logs` 확인 |
| `Pytest fail` | 회귀 결함 | ✅ 본 PC 재현·수정 |

### 단계 9 — Fly CLI 인증 (배포 시만)

```bash
fly auth login                          # OAuth (권장)
# 또는
setx FLY_API_TOKEN "..."                # PowerShell — 채팅 X
```

확인:
```bash
fly auth whoami
fly status --app <your-app>
```

### 단계 10 — 진행 중 phase 추천

```bash
grep -E "🟢|🟡|⚪" vault/roadmap/INDEX.md | head -10
```

본 시점 (2026-05-20) 대기 영역:
- 🟢 5건 딥리서치 PROMPT (facial-feature-phase2·physiognomy-knowledge-db·twelve-palace-bbox·saas-legal-compliance·flyio-oidc-reusable-workflows)
- ⚪ 기술 부채 (refactor-debts 5건)
- ADR 후속 (모바일 우선·인증·놀이 — ADR-058·059·060 후보)

### 단계 11 — 사용자 보고

```
## 본 PC 환경 준비 완료

### Repo
- HEAD: <hash>
- origin 차이: N건 (pull 권고 / 없음)
- 최근 ADR: ADR-057

### 도구 (✅/❌)
- Python 3.12 ✅, git ✅, fly ✅, gh ✅, docker ✅

### 회귀
- 27 PASS in ~37s

### 라이브
- /api/health 200 OK
- / 200 OK

### CI/배포
- 최근 run: <status>
- FLY_API_TOKEN Secret: 등록/부재
- 차단 이슈: <있다면 명시>

### 다음 작업 후보
- (사용자 명시 의도 대기)
```

## Fly.io 배포·관리 가이드 (참조 영역)

### CLI 명령어

```bash
# 배포
fly deploy --remote-only --ha=false              # 본 시스템 표준
fly status --app <your-app>
fly logs --app <your-app>

# 머신 관리
fly machine list --app <your-app>
fly scale memory 1024 --app <your-app>
fly scale count 1 --app <your-app>

# Secrets
fly secrets list --app <your-app>
fly secrets set KEY="value" --app <your-app>
fly secrets unset KEY --app <your-app>

# Volumes
fly volumes list --app <your-app>
fly volumes create saju_data --region nrt --size 1 --app <your-app>

# SSH
fly ssh console --app <your-app>

# 도메인
fly certs add example.com --app <your-app>
fly ips list --app <your-app>

# 릴리스
fly releases --app <your-app>
fly releases rollback --app <your-app>
```

### fly.toml 본 시스템 표준

```toml
app = "<your-app>"
primary_region = "nrt"

[build]
  dockerfile = "Dockerfile"

[env]
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 0

[[mounts]]
  source = "saju_data"
  destination = "/app/data"

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory_mb = 512
```

> `auto_stop_machines = "stop"` + `min_machines_running = 0` → 비활성 시 자동 정지
> 첫 요청 시 cold start 1~3초

### 본 시스템 필수 Secrets

| Secret | 용도 |
|---|---|
| `BIZROUTER_API_KEY` | LLM 라우터 (Vision·작문) |
| `MINIMAX_API_KEY` | <도메인 3> Vision fallback |
| `ANTHROPIC_API_KEY` | Claude SDK 직접 fallback |
| `OPENAI_API_KEY` | 선택 |

### Region 한국 부재 — 본 시스템 결정

ADR-056 시점 Fly.io 공식 region:
- `nrt` (Tokyo, Japan) ← **본 시스템**
- `hkg` (Hong Kong)
- `sin` (Singapore)

한국 region 부재. 한국 → Tokyo 지연 ~30ms 수용 가능.

### 트러블슈팅

| 문제 | 원인 | 해결 |
|---|---|---|
| `Error: not authenticated` | 토큰 누락 | `fly auth login` 또는 `setx FLY_API_TOKEN` |
| 502 Bad Gateway | 앱 미기동·포트 불일치 | `fly logs` + `fly.toml internal_port` 확인 |
| Cold start 느림 | `auto_stop_machines` 비활성 | `min_machines_running = 1` |
| Volume not mounted | `[mounts]` 절 누락 | `fly.toml` 확인 |
| Build OOM | Builder 메모리 부족 | `--build-arg` 또는 multi-stage |

### GraphQL API (고급)

엔드포인트: `https://api.fly.io/graphql`
인증: `Authorization: Bearer $FLY_API_TOKEN`

```bash
curl -s -X POST https://api.fly.io/graphql \
  -H "Authorization: Bearer $FLY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ app(name: \"<your-app>\") { id name status hostname } }"}'
```

## 안전장치

### 노출 자산 차단

본 명령어는 **API 키·토큰을 채팅에 받지 않는 것을 원칙**으로 한다. 단:
- 사용자 명시 의도 (3회 이상 동일 토큰 전송 + "그냥 진행하자" 등 강한 명시) 시
- 본 시스템이 직접 `gh secret set` 사용 가능
- 작업 후 **즉시 폐기 + 신규 발급 권고** 동반 의무

### 회귀 실패 시

단계 6 회귀 fail → 사용자에게 정확한 fail 보고서 + 진행 정지.

### 라이브 검증 실패 시

라이브 404·502 → `fly status` + `fly logs` 자동 확인 권고.

### vault 미존재 시

`vault/`는 .gitignore — 별도 동기화 의무. 미존재 시 단계 4·5·10 skip + 사용자 안내.

### 다른 PC 신규 커밋 감지 시

`git fetch` 후 origin 차이 1건 이상:
- 본 PC 작업 진행 전 `git pull` 권고
- 사용자 명시 후 pull 진행 (자동 X — conflict 위험)

## 본 명령어 자체 동기화

`.claude/`는 `.gitignore` 처리되어 자동 동기화 X. **각 PC에 본 명령어를
별도 배치 의무**. 백업 위치:

```bash
# 본 명령어 vault 영속화 (다른 PC 복원용)
cp .claude/commands/flyio/index.md vault/templates/SLASH_COMMAND_flyio_index.md
cp .claude/commands/flyio/deploy.md vault/templates/SLASH_COMMAND_flyio_deploy.md
cp .claude/commands/flyio/capacity.md vault/templates/SLASH_COMMAND_flyio_capacity.md
```

다른 PC 첫 실행 시 `bootstrap-claude-commands.sh` 자동 복원.

## 관련 ADR

- ADR-056: Railway → Fly.io 마이그레이션
- ADR-057: paths-filter + Docker 멀티스테이지
- ADR-010: 사실성 분리

## 하위 명령어

- `/flyio deploy` — 배포 실행 (회귀 → deploy → 라이브 검증)
- `/flyio capacity` — 인프라 부하 측정·동시 접속 한계 보고

## 주의

- 본 명령어는 **본 프로젝트 (<your-app>) 전용**
- 인자 없을 시 → 부트스트랩 자동 수행
- 인자 `deploy`·`capacity` → 하위 명령어 dispatch
- 1회 실행만 의도 — 정상 완료 시 종료, 사용자 다음 의도 명시 대기
