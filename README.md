#stock_simul_1min 

# stock_sim_2day — 필터 없는 이틀치 종목 매매 시뮬레이션 (데이터 재사용 버전)

`stock_practice_1min`(1분봉 매매연습)과 완전히 별도로 배포되는 프로그램입니다.
**Toss API는 stock_practice_1min만 호출하고, 이 프로그램은 Toss API를 전혀 호출하지
않습니다.** 대신 stock_practice_1min이 장중 내내 이미 수집해 로컬에 저장해 둔
원본 1분봉 데이터(`data_1min/{날짜}/_raw/*.json`)를 그대로 읽어서, 필터 없이
전 종목을 후보로 만들어 별도 Worker에 업로드합니다.

## 동작 원리

1. stock_practice_1min이 시총 10조+ 스크리닝을 통과한 전체 종목을 장중
   폴링하며 `_raw` 폴더에 이미 전부 저장해 둠 (필터는 그 다음 단계에서만 적용됨)
2. 이 프로그램은 그 `_raw` 원본을 읽기만 함 — API 호출 없음
3. 시가/고가/저가/종가는 이미 수집된 1분봉 자체에서 계산 (첫봉 시가, 구간 최고/최저,
   마지막봉 종가). 전일 종가는 로컬에 남아있는 전 거래일 `_raw` 데이터의 마지막
   종가를 사용하고, 없으면 등락률만 생략
4. 진폭/양봉 여부는 참고용 메타로만 계산 — **선별 조건으로 쓰이지 않음** (필터 없음)
5. `data_2day_sim/{날짜}/`에 저장, 3거래일 롤링 보관, 최근 2일치를 worker가 노출
6. `candle-sim-2day` Worker(별도 KV `SIM2DAY_KV`)로 업로드

## stock_practice_1min과의 차이점

| 항목 | 기존 (stock_practice_1min) | 신규 (stock_sim_2day) |
|---|---|---|
| Toss API 호출 | 장중 폴링 + 일봉 조회 | **없음 — 로컬 데이터만 재사용** |
| 시총 스크리닝 기준 | 10조원 이상 | 동일 (stock_practice_1min이 이미 적용) |
| 최근 2일치 노출 | 동일 | 동일 |
| 3일 롤링 보관 | 동일 | 동일 |
| 진폭 필터 | (고가-저가)/저가 ≥ 5% | **제거 — 필터 없음** |
| 양봉 필터 | 당일 종가 > 시가 | **제거 — 필터 없음** |
| 후보 종목 수 | 필터 통과 종목만 | stock_practice_1min이 그날 수집한 전체 종목 |
| 실행 방식 | 08:00~20:00 장중 상시 폴링 | **20:10 하루 1회 실행되는 짧은 후처리 스크립트** |

## 파일 구성

- `sim2day_build_and_upload.py` — 메인 스크립트 (Task Scheduler가 매일 20:10 1회 실행,
  Toss API 미호출, stock_practice_1min의 로컬 데이터만 읽음)
- `retry_upload.py` — 업로드만 실패했을 때 재시도하는 유틸리티
- `worker.js` — Cloudflare Worker 백엔드 (새 KV 네임스페이스 `SIM2DAY_KV`)
- `wrangler.toml` — Worker 배포 설정
- `2day_sim_trading_practice.html` — 프론트엔드 (Worker 주소만 교체, 나머지 동일)
- `run_collector.bat` / `register_task.ps1` — Windows 작업 스케줄러 등록용 (20:10 예약)

## 배포 절차

1. **Cloudflare Worker 새로 생성**
   ```
   wrangler kv:namespace create SIM2DAY_KV
   ```
   출력된 id를 `wrangler.toml`의 `id = "REPLACE_WITH_NEW_KV_NAMESPACE_ID"`에 붙여넣기.
   ```
   wrangler secret put UPLOAD_KEY
   wrangler secret put GEMINI_API_KEY
   wrangler deploy
   ```
   배포된 실제 주소가 `https://candle-sim-2day.kim80091.workers.dev`와 다르면
   `sim2day_build_and_upload.py`의 `WORKER_URL`과 HTML의 `WORKER` 상수를 실제 주소로
   교체하세요.

2. **로컬 PC 설정**
   - 이 폴더를 `C:\Users\David\Desktop\stock_sim_2day`에 두기
     (`stock_practice_1min` 폴더가 `C:\Users\David\Desktop\stock_practice_1min`에
     있다는 전제로 `SOURCE_DATA_DIR` 기본값을 잡아뒀습니다. 경로가 다르면
     `sim2day_build_and_upload.py` 상단의 `SOURCE_DATA_DIR`를 수정하거나 환경변수
     `SIM2DAY_SOURCE_DIR`를 설정하세요)
   - Toss API 키(`.env`)는 이 프로그램에 필요 없습니다 — API를 안 쓰기 때문입니다
   - 환경변수 `SIM2DAY_UPLOAD_KEY`를 Worker의 `UPLOAD_KEY`와 동일하게 설정
   - 관리자 권한 PowerShell에서 `register_task.ps1` 실행
     → 매일 20:10(stock_practice_1min의 20:00 장마감 처리 10분 뒤) 자동 실행

3. **프론트엔드**
   - `2day_sim_trading_practice.html`을 GitHub Pages 등 기존 방식대로 배포
   - 최소 하루 이상 stock_practice_1min + 이 스크립트가 돌아야 "최근 2일치" 목록이
     채워집니다

## 주의

- 이 프로그램은 stock_practice_1min이 그날 수집을 다 끝낸 뒤에 실행돼야 정확한
  데이터를 읽습니다. `register_task.ps1`은 stock_practice_1min의 20:00 장마감
  처리보다 10분 늦은 20:10에 예약합니다. stock_practice_1min 쪽 처리가 그보다
  오래 걸린다면 예약 시각을 더 늦추세요.
- stock_practice_1min의 3일 롤링 보관 주기 안에서만 전일 종가를 로컬에서 구할 수
  있습니다. 그 범위를 벗어난 날짜를 나중에 처리하면 등락률(chg_pct)이 빠질 수
  있습니다(양보다 불편이 적은 부분이라 그냥 생략 처리됩니다).
- 필터가 없어서 매일 저장/업로드되는 종목 수가 기존보다 훨씬 많아집니다
  (stock_practice_1min이 그날 수집한 전체 유니버스가 그대로 후보이므로). Worker
  KV 저장량이 늘어나는 점 참고하세요.
