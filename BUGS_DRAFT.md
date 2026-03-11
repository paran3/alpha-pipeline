# 발견된 버그 목록 (임시)

## Critical (프로덕션 배포 불가)

| # | 파일 | 버그 |
|---|------|------|
| 1 | `runner/backtest.py:23,56,67` | **전역 `_ticker_returns` 리스트 경쟁 조건** — `run_comparison`이 `asyncio.gather`로 여러 백테스트를 동시 실행하면, 모두 같은 전역 리스트를 읽고 쓰므로 결과가 뒤섞임 |
| 2 | `runner/backtest.py:77` | **`emit()` 호출에 `await` 누락** — `async def emit()`인데 `await` 없이 호출. "complete" 이벤트가 절대 발송되지 않고, `RuntimeWarning` 발생 |
| 3 | `docker-compose.yml` | **`REDIS_URL` 환경변수 미설정** — config.py 기본값이 `redis://localhost:6379`인데, 컨테이너 안에서는 `redis://redis:6379`여야 함. Docker 환경에서 API/Worker 모두 Redis 연결 실패 |
| 4 | `strategy/sma_cross.py:41` | **SMA 전략 look-ahead bias** — `signal * daily_return`에서 signal을 `.shift(1)` 하지 않음. 당일 종가로 시그널을 계산하고 당일 수익률에 적용 → 백테스트 성과 부풀림 |
| 5 | `strategy/macd.py:22` | **MACD signal 기간 기본값 오류** — `settings.MACD_SIGNAL`(9) 대신 `settings.MACD_SLOW`(26)를 사용. MACD 전략이 완전히 다른 시그널을 생성 |
| 6 | `metrics/performance.py:29` | **Sharpe ratio 연율화 누락** — `excess.mean() / excess.std()`만 계산하고 `* sqrt(252)` 미적용. 보고되는 Sharpe가 실제보다 ~16배 낮음 |

## High (결과 신뢰성/데이터 무결성 영향)

| # | 파일 | 버그 |
|---|------|------|
| 7 | `runner/progress.py:19,27,42` | **SSE 진행률 전역 `_cursor`** — 동시 실행 시 다른 run의 진행률이 클라이언트에 전송됨 |
| 8 | `runner/progress.py:17-28` | **SSE 상태가 인프로세스 메모리** — Worker가 `emit()`하면 Worker 메모리에만 저장. API 컨테이너의 `stream()`은 항상 빈 상태를 읽음 |
| 9 | `mq/redis_queue.py:49` | **`rpop` 사용으로 job 유실** — Worker가 pop 후 크래시하면 작업이 영구 소실. ack 메커니즘 없음 |
| 10 | `runner/job_queue.py:41-42` | **예외 무시** — `except Exception: pass`로 모든 에러를 삼킴. 백테스트 실패가 완전히 은폐됨 |
| 11 | `docker-compose.yml:20-21` | **Redis healthy 대기 없음** — `depends_on`에 `condition: service_healthy` 미사용. Redis 준비 전에 API/Worker가 연결 시도 |
| 12 | `data/universe.py:28` | **`as_of` 파라미터 무시** — 항상 마지막 날짜 기준으로 유니버스를 구성. 생존자 편향(survivorship bias) 발생 |
| 13 | `strategy/macd.py:42-45`, `rsi.py:59-62` | **거래비용 타이밍 불일치** — turnover은 unshifted signal 기준, return은 shifted signal 기준. 비용이 하루 먼저 부과됨 |
| 14 | `runner/dispatcher.py:34` | **`asyncio.gather` 에러 처리 없음** — 하나가 실패하면 나머지 완료된 결과도 전부 유실 |
| 15 | `strategy/registry.py:26-28` | **전략 캐시가 kwargs 무시** — 같은 이름이면 다른 파라미터로 요청해도 이전 인스턴스 반환 |

## Medium

| # | 파일 | 버그 |
|---|------|------|
| 16 | `storage/result_store.py:34-39` | **중첩 구조 round 미적용 + numpy 타입 직렬화 실패 가능** — `json.dumps` 시 `numpy.float64` 등에서 `TypeError` |
| 17 | `storage/result_store.py:39` | **결과에 TTL 없음** — Redis 메모리가 무한 증가 |
| 18 | `mq/redis_queue.py:27-31`, `result_store.py:23-27` | **Redis 연결 미해제** — lifespan에서 cleanup 없음 |
| 19 | `runner/backtest.py:47` | **`asyncio.get_event_loop()` deprecated** — 3.10+에서 `get_running_loop()` 사용해야 함 |
| 20 | `runner/backtest.py:63,72,75` | **CPU-bound Pandas 작업이 이벤트 루프 블로킹** — `compute`만 executor로 보내고 나머지는 메인 루프에서 실행 |
| 21 | `strategy/rsi.py:27-28` | **RSI 계산 시 avg_loss=0 대체가 avg_gain>0 케이스에서 NaN 전파** — RSI가 100이어야 할 때 NaN |
| 22 | `data/cache.py:17,21` | **캐시가 mutable reference 반환** — 호출자가 `.copy()` 안 하면 캐시 오염 |

## Low

| # | 파일 | 버그 |
|---|------|------|
| 23 | `worker.py:37` | **50ms 타이트 폴링** — idle 시 초당 ~20회 Redis 호출 |
| 24 | `data/loader.py:52` | **첫 행 NaN을 0으로 채움** — 데이터 갭 시 다일 수익률 왜곡 가능 |
| 25 | `metrics/performance.py:40-46` | **CAGR 짧은 기간 불안정** — 1일 1% 수익이 연율 1124%로 계산 |

## 테스트 관련

| # | 파일 | 이슈 |
|---|------|------|
| 26 | `test_metrics.py:32` | **`test_sharpe_annualised`가 현재 실패하는 테스트** — Sharpe 연율화 버그(#6)를 정확히 잡지만, 버그가 있으니 테스트 스위트가 통과 불가 |
| 27 | `test_signals.py:45-64` | **look-ahead 테스트가 무의미** — threshold `-5.0`이 너무 느슨해서 어떤 경우든 통과 |
| 28 | 전반 | **Redis 큐, 결과 저장소, Dispatcher, Worker, API 엔드포인트 테스트 전무** |
