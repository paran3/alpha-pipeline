# Alpha Pipeline

알파 리서치 팀 내부 백테스팅 파이프라인.
데이터 로딩 → 전략 시그널 생성 → 백테스트 실행 → 성과 지표 계산을 담당한다.
FastAPI(API) + Redis(큐/결과 저장소) + async Worker 분산 구조.

## 목적

이 시스템은 FastAPI 서버, 백테스트 워커, Redis로 구성된 분산 파이프라인이다.
코드를 넘겨받은 상태이며, **이 파이프라인을 프로덕션에서 신뢰할 수 있는 상태로 만드는 것**이 목표다.

## 빌드 & 실행

```bash
uv sync                                # 의존성 설치
uv run pytest -v                       # 테스트
docker compose up --build              # 전체 스택 로컬 실행
uv run python scripts/generate_data.py # 샘플 데이터 생성
```

## 코드 컨벤션

- Python >=3.11, 모든 모듈에 `from __future__ import annotations` 사용
- snake_case 함수/변수, PascalCase 클래스, UPPER_SNAKE_CASE 상수
- 비동기 패턴: async/await (Redis, I/O 전부)
- 설정: Pydantic Settings — `config.py`의 `settings` 싱글턴 사용
- 테스트: pytest + pytest-asyncio, `asyncio_mode = "auto"`

## 아키텍처 핵심 규칙

- API(`main.py`)와 Worker(`worker.py`)는 별도 프로세스 → 메모리 공유 불가, Redis로만 통신
- 전략 패턴: `BaseStrategy.compute(df)` → `signal`(int) + `strategy_return`(float) 컬럼 반환
- DataFrame 파이프라인: date 인덱스, 컬럼은 `[open, high, low, close, volume, daily_return]`
- 전략 등록: `strategy/registry.py`에서 관리

## 작업 원칙

- 금융 계산 정확성 최우선 (투자 의사결정용 시스템)
- 동시 실행(concurrency) 안전성 항상 고려
- 전역 상태(global state) 사용 지양
- 변경 시 테스트 추가/확인 필수

## 산출물 관리

### REPORT.md
발견한 버그와 문제를 기록하는 파일. 사용자가 "저장해" 등 명시적으로 요청할 때 작성/갱신한다.
- 발견한 문제 목록과 각각의 심각도 판단
- 고친 것과 고치지 않은 것, 그 우선순위 판단 이유
- 이 파이프라인의 결과를 지금 신뢰할 수 있는지, 없다면 무엇이 더 필요한지

### DECISION_LOG.md
의사결정 과정을 기록하는 파일. 사용자의 판단과 그 이유를 추적한다.
- 사용한 AI 도구 목록 (Claude Code, ChatGPT, Copilot 등)
- AI가 "괜찮다"고 했지만 사용자가 의심하거나 직접 확인한 사례
- AI 제안을 거부하거나 수정한 것과 그 이유
- AI가 놓쳤는데 사용자가 직접 발견한 것
- 이유가 불분명한 의사결정은 반드시 사용자에게 확인 후 기록
