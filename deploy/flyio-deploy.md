# /flyio deploy — Fly.io 배포 실행

본 명령어는 본 시스템 (<your-app>) Fly.io 배포를 단일 호출로 완결한다.
사용자 명시 "배포해" / "deploy" 의도 즉시 사용.

## 사용법

```
/flyio deploy
/flyio deploy --skip-test   # 회귀 테스트 건너뜀 (긴급 hotfix 시만)
```

## 행동 절차

### 단계 0 — 인증·연결 점검

```bash
flyctl auth whoami
# fail → flyctl auth login 또는 setx FLY_API_TOKEN 안내 + 정지

flyctl status --app <your-app>
# fail → fly.toml 또는 app name 확인 + 정지
```

### 단계 1 — Git 상태 점검

```bash
git status
git log --oneline -3
```

- working tree clean 확인 (dirty 시 사용자에게 commit 여부 확인)
- 최근 커밋 식별 (배포 대상)

### 단계 2 — 회귀 테스트 (--skip-test 미지정 시)

```bash
python -m pytest tests/smoke/ tests/regression/ -m "not live" --tb=short -q
# 예상: 27 PASS in ~37s
```

- fail → 사용자 결단 (수정 또는 `--skip-test` 재호출) 대기
- pass → 단계 3 진행

### 단계 3 — Fly.io 배포

```bash
flyctl deploy --remote-only --ha=false --app <your-app>
```

옵션:
- `--remote-only`: Fly.io builder 사용 (로컬 Docker 불필요)
- `--ha=false`: 단일 머신 (본 시스템 Volume 정합)

성공 시: `==> v<N> deployed successfully`

### 단계 4 — 라이브 검증 (최대 120s)

```bash
for i in $(seq 1 12); do
  sleep 10
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 https://<your-app>.fly.dev/api/health || echo 000)
  echo "attempt $i: HTTP=$code"
  if [ "$code" = "200" ]; then
    echo "✅ Live OK"
    break
  fi
done
```

- 200 OK → 단계 5
- 120s 초과 → `flyctl logs` 자동 출력 + 정지

### 단계 5 — 핵심 엔드포인트 검증

```bash
# 묵향 선생 풀이
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"fullname_ko":"홍길동"}' \
  https://<your-app>.fly.dev/api/name/reading | head -c 300

# domain combinations
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"year":1990,"month":5,"day":15,"hour":10}' \
  https://<your-app>.fly.dev/api/saju/compute | head -c 300
```

- 200 OK + 본문 정상 → 배포 성공
- 5xx → `flyctl logs` + 사용자 보고

### 단계 6 — E2E 라이브 테스트 (선택)

```bash
python -m pytest tests/e2e/ -m live --tb=no -q
# 예상: 6 PASS in 15-20s
```

### 단계 7 — 사용자 보고

마크다운 형식:

```
## 배포 완료

### Git
- 배포 커밋: <hash> <subject>

### 회귀
- tests/smoke + tests/regression: 27 PASS in 37s

### Fly.io
- App: <your-app>
- Region: nrt (Tokyo)
- Release: v<N>
- 이미지: ~88MB

### 라이브
- /api/health: 200 OK
- /api/name/reading: 200 OK (묵향 선생 풀이 확인)
- E2E: 6 PASS

### URL
https://<your-app>.fly.dev/
```

## 실패 시 행동

### 빌드 실패

```bash
flyctl logs --app <your-app> -n
```

`Step <N>/<M>: <command>` 에서 fail → Dockerfile 검토 권고

### 런타임 fail (502)

```bash
flyctl logs --app <your-app> -n | head -50
```

- `ModuleNotFoundError` → requirements.txt 누락
- `Port mismatch` → fly.toml `internal_port` vs Dockerfile `CMD` 확인
- `OOM` → `flyctl scale memory 1024` 검토

### Volume mount fail

```bash
flyctl volumes list --app <your-app>
```

- `saju_data` 누락 → `flyctl volumes create saju_data --region nrt --size 1`
- 다른 region에 생성됨 → fly.toml `primary_region = "nrt"` 확인

## 자동 롤백 (선택)

라이브 검증 5번 연속 fail 시 자동 롤백 옵션:

```bash
flyctl releases rollback --app <your-app>
```

사용자 명시 의도 없으면 자동 X — 사용자에게 결단 요청.

## 안전장치

### 채팅 입력 금지

- API 토큰 (FLY_API_TOKEN)
- Secret 값 (BIZROUTER_API_KEY 등)

사용자 본인 PC에서:
- `flyctl auth login` (OAuth)
- `flyctl secrets set KEY="value" --app <your-app>`
- 또는 `setx FLY_API_TOKEN "..."` (PowerShell)

### 회귀 실패 시

자동 배포 X. 사용자 결단 (수정 또는 `--skip-test`) 대기.

### main 브랜치 외 배포

본 명령어는 현재 브랜치 deploy. main 외 브랜치는 staging 또는 preview 의도일 가능성 — 사용자 확인 후 진행.

## 관련 파일

- `fly.toml`: 본 시스템 Fly.io 표준 (region·VM·mount)
- `Dockerfile`: ADR-057 multi-stage (88MB)
- `.github/workflows/deploy.yml`: CI/CD에서 본 단계 자동 수행 (push to main)

## 관련 ADR

- ADR-056: Railway → Fly.io 마이그레이션
- ADR-057: paths-filter + Docker 멀티스테이지

## 주의

- 본 명령어는 **본 프로젝트 (<your-app>) 전용**. 다른 앱은 `--app <name>` 명시
- `--skip-test`는 긴급 hotfix 시만. 일반 배포는 회귀 의무
- 배포 후 사용자 출력 결과 정직 보고 (ADR-010) — 실패 은폐 X
