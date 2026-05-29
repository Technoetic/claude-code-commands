# /flyio capacity — Fly.io 인프라 부하 측정 + 동시 접속 한계 보고

본 명령어는 본 시스템 (<your-app>) Fly.io 배포 인프라의 실 부하를 측정하고
**현 사양으로 수용 가능한 동시 접속 한계**를 정량 보고한다.

## 사용법

```
/flyio capacity
/flyio capacity --quick    # 빠른 검사 (단일 burst)
/flyio capacity --stress   # 압박 부하 (단계적 RPS 증가)
```

## 행동 절차

### 단계 0 — 현 사양 확인

```bash
flyctl status --app <your-app>
flyctl scale show --app <your-app>
flyctl machine list --app <your-app>
```

확인:
- VM 사양 (shared-cpu-1x 512MB 기본)
- 머신 수 (보통 1대)
- Region (`nrt`)
- `auto_stop_machines` 활성 여부

### 단계 1 — Cold start 측정

```bash
# 머신 강제 정지 → 첫 요청 측정
flyctl machine stop <id> --app <your-app>
sleep 5

time curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" \
  https://<your-app>.fly.dev/api/health
```

예상:
- Cold start: 1~3초 (auto_start로 머신 기동)
- Warm: <100ms

### 단계 2 — Warm baseline

```bash
# 10회 연속 (warm 상태)
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{time_total}\n" https://<your-app>.fly.dev/api/health
done | awk '{sum+=$1; if($1>max)max=$1; if(min==0||$1<min)min=$1} END {print "avg="sum/NR, "min="min, "max="max}'
```

지표:
- p50·p95·p99 응답 시간
- 정상 임계: p95 < 300ms

### 단계 3 — 동시 접속 burst (--quick)

```bash
# 10 동시 요청
seq 1 10 | xargs -P 10 -I {} curl -s -o /dev/null -w "%{http_code} %{time_total}\n" \
  https://<your-app>.fly.dev/api/health
```

확인:
- 200 OK 비율
- 응답 시간 분포
- 5xx 발생 시 정지

### 단계 4 — 단계적 RPS 부하 (--stress)

본 명령어는 RPS를 단계적으로 증가시키며 한계점 탐지:

```bash
# RPS 1, 5, 10, 20, 50, 100 단계
for rps in 1 5 10 20 50 100; do
  echo "=== RPS $rps ==="
  # 10초간 동시 RPS 발생
  for i in $(seq 1 $((rps * 10))); do
    curl -s -o /dev/null -w "%{http_code} %{time_total}\n" \
      https://<your-app>.fly.dev/api/health &
    sleep $(echo "1.0 / $rps" | bc -l)
  done
  wait
  echo "---"
done
```

또는 `wrk`/`hey`/`vegeta` 사용 (설치 시):
```bash
# hey (Go)
hey -z 10s -c 10 https://<your-app>.fly.dev/api/health
hey -z 10s -c 50 https://<your-app>.fly.dev/api/health
hey -z 10s -c 100 https://<your-app>.fly.dev/api/health
```

### 단계 5 — LLM 엔드포인트 부하 (실 사용 시뮬레이션)

본 시스템 핵심 엔드포인트는 LLM 호출 (BizRouter → Gemini Flash Lite).
LLM 응답 시간 + Rate Limit이 동시 접속 한계를 결정.

```bash
# <도메인 4> 풀이 5 동시 (LLM 호출)
for i in $(seq 1 5); do
  curl -s -X POST -H "Content-Type: application/json" \
    -d "{\"fullname_ko\":\"홍길동$i\"}" \
    https://<your-app>.fly.dev/api/name/reading \
    -o /dev/null -w "$i: %{http_code} %{time_total}\n" &
done
wait
```

확인:
- LLM 응답 시간 (보통 2~8초)
- BizRouter rate limit (분당 토큰 한계)
- 5xx 발생 시 LLM API 한계 도달

### 단계 6 — 메모리·CPU 측정

```bash
flyctl ssh console --app <your-app> -C "cat /proc/meminfo | head -5"
flyctl ssh console --app <your-app> -C "top -bn1 | head -15"
```

확인:
- 현 메모리 사용 (보통 200~350MB / 512MB)
- 부하 중 메모리 peak
- CPU steal (공유 VM 특성)

### 단계 7 — 한계점 분석

종합 보고:

| 지표 | 측정값 | 한계 추정 |
|---|---|---|
| /api/health p95 | ~Xms | RPS Y까지 안정 |
| LLM 엔드포인트 p95 | ~Xs | 동시 Y개까지 |
| 메모리 peak | XMB / 512 | OOM 임박 RPS |
| BizRouter rate limit | 분당 X 토큰 | LLM 동시 한계 |

**한계 결정 요인 (순위)**:
1. LLM API rate limit (BizRouter·Gemini) — 보통 가장 먼저 한계
2. 메모리 (512MB) — LLM 동시 처리 시 메모리 증가
3. 단일 머신 CPU — `shared-cpu-1x`
4. Cold start (auto_stop 시)

### 단계 8 — 사용자 보고

```
## Fly.io 인프라 부하 측정 결과

### 현 사양
- VM: shared-cpu-1x 512MB
- Region: nrt (Tokyo)
- 머신 수: 1대
- Volume: saju_data 1GB

### Baseline (warm)
- /api/health p50: Xms
- /api/health p95: Xms
- /api/name/reading p50: Xs (LLM)

### 동시 접속 한계
- 정적 엔드포인트: ~Y RPS (CPU 한계)
- LLM 엔드포인트: 동시 Z개 (BizRouter rate limit)
- 메모리 OOM 임박: 동시 W개

### 권고
| 사용자 규모 | 권고 |
|---|---|
| ~10 동시 | 현 사양 충분 |
| ~50 동시 | `flyctl scale memory 1024` 또는 `flyctl scale count 2` |
| ~100 동시 | shared-cpu-2x + Redis 캐시 + LLM 비동기 큐 |
| ~500 동시 | 별도 아키텍처 설계 (CDN·전용 LLM 서버) |

### 비용 예상 (월)
- 현 사양: ~$1.94/월 (256h 기동 시)
- 1024MB: ~$3.88/월
- 2대: ~$3.88/월
- 24/7 1대 512MB: ~$5.70/월
```

## 안전장치

### 자기 부하로 인한 사용자 영향

본 명령어는 라이브 서비스에 부하를 생성. 다음 시간대 사용 권고:
- 새벽 (KST 02:00~05:00)
- 사용자 트래픽 0인 시점

`--stress` 모드는 실 사용자 영향 가능 — 사용자 명시 의도 확인 후 진행.

### Rate Limit 차단 위험

본 시스템 LLM API 키 (BizRouter·MiniMax) 부하 측정으로 분당 한계 소모.
실 사용자 풀이가 잠시 fail할 수 있음.

권고:
- 측정 한계 RPS 도달 시 즉시 중단
- 측정 후 5분 대기 (rate limit 회복)

### 비용 발생

Fly.io 트래픽 비용: 첫 100GB/월 무료, 이후 $0.02/GB.
부하 측정으로 100GB 초과 우려 X (보통 측정 < 1GB).

## 실 트래픽 모니터링 (참고)

```bash
flyctl logs --app <your-app> -n | grep -E "GET|POST" | wc -l
flyctl dashboard --app <your-app>  # 브라우저 대시보드
```

## 관련 ADR

- ADR-056: Fly.io 마이그레이션 (사양 결정 근거)
- ADR-057: Docker 88MB (cold start 단축)

## 주의

- 본 명령어는 **본 프로젝트 (<your-app>) 전용**
- 라이브 부하 측정 — 사용자 영향 가능, 명시 의도 확인
- 결과는 시점 측정 — Fly.io infra 변동으로 재측정 필요
- 정확한 한계는 운영 데이터 누적 후 재평가 (ADR-010 정직 원칙)
