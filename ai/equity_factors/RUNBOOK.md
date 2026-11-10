# 일일 실행·발행 절차

대상 원본은 `YoundongKim/newspaper-pipeline`의 `static/ai/equity_factors/`, 공개 AI 데이터는 `YoundongKim/newspaper-ai-public`의 `ai/equity_factors/`입니다. 사용자 요청으로 일일 조사와 이 범위의 공개 반영이 승인됐습니다.

1. 최신 `README.md`, `factor_registry.json`, `index.json`, 최근 daily JSON을 읽습니다. KST 실행일과 실제 조사 기준시각을 확정합니다. 실행이 늦으면 실제 시간을 쓰고 03시 스냅샷이라고 허위 표시하지 않습니다.
2. 국내·미국 실제 거래일/시장 상태를 확인합니다. 국내 KRX 완료 세션, 이를 선행한 미국 완료 세션, 현재 미국 진행 세션을 나눕니다. 주말/휴장일에도 조사하되 가격을 만들거나 거래일을 중복 누적하지 않습니다.
3. 가격·거래량·시장 대비 등락·환율·주요 미국 지수/종목·종목별 외국인/기관 수급을 확인합니다. HBM·메모리·CAPEX·실적·정책·기업 사건을 실제로 검색합니다. 최근 공개 `ai/articles.json`은 보조 근거이며 03시에는 당일 06시 기사 수집이 아직 안 됐을 수 있습니다. 수집을 기다리지 말고 공개 원출처를 직접 조사합니다.
4. 아래 report 계약에 맞춰 `daily/실행일.json`을 만듭니다. 기존 동일 날짜 파일은 먼저 읽고 보존하며 정정 시 `revision_note`와 실제 as_of를 남깁니다. 일자 전체를 빈 템플릿으로 덮어쓰지 않습니다.
5. 원본 루트의 `equity_factor_store.py`와 기존 `history.json`, `index.json`, `latest.json`을 먼저 로컬에 준비합니다. 새 daily JSON만 추가한 뒤 `python equity_factor_store.py rebuild --require-base`를 실행합니다. 개별 기록 검증, 새 daily MD·latest·history·rolling·index가 증분 재생성되며 과거 기록을 보존합니다. `python equity_factor_store.py validate`도 통과해야 합니다. 기존 base를 읽지 못했는데 새 파일만으로 history/index를 재생성하면 안 됩니다. 실행 환경이 없으면 코드 내용을 읽고 동등한 검증과 증분 재집계를 수행하되 검증 한계를 기록합니다.
6. 새 보고서와 파생 파일을 **한 커밋**으로 원본 main에 추가하고, 같은 커밋에서 `public_ai_publish_trigger.txt`를 현재 시각·이번 report_date로 갱신해 기존 공개 publish workflow를 실행합니다. Git tree는 최신 main을 base로 하며 ref 업데이트에 expected_sha를 써서 다른 자동화의 변경을 보존합니다. 충돌하면 새 head를 읽고 재적용하며 강제 push하지 않습니다. 기존 기사·DB·다른 예약은 수정하지 않습니다.
7. Publish Public AI Export 실행 완료와 공개 저장소의 report_date/as_of를 확인합니다. 가능한 실행시간 안에 Pages의 latest.json 또는 GitHub 원본을 읽어 내용까지 검증합니다. 요청/커밋만으로 발행 완료라고 하지 않습니다. 미완료이면 원본 커밋, 실행 URL, 남은 단계만 남기고 다음 실행이 그 단계부터 복구합니다. 같은 보고서 재생성·중복 예약을 만들지 않습니다.
8. 사용자에게 종목별 주요 요인·확신도·반례·누적 표본 수·쇼츠 후보와 공개 링크를 짧게 알립니다. 결측·휴장만 있는 날은 불필요한 성공 알림을 늘리지 말고 GitHub에 상태를 남깁니다. 인증/권한 오류가 있으면 실제 장애를 알리고 공개 반영 완료를 주장하지 않습니다.

## 일일 JSON 계약

bootstrap `daily/2026-11-10.json`을 실제 형식 예로 사용합니다. 핵심 필드는 다음과 같습니다.

- 최상위: `schema_version`, `copyright_policy.full_article_text_exported=false`, `report_date`(KST 조사일), `as_of`(offset 포함), `run_kind`(scheduled/recovery/bootstrap), `status`(complete/partial/no_new_session), `kr_trade_date`, `us_previous_completed_trade_date`, `summary`, `us_current`, `market_context`, `equities`, `cross_stock_comparison`, `shorts_candidates`, `sources`, `limitations`.
- `us_current`: `session_date`, `status`(not_open/intraday/closed/holiday/weekend/unavailable), `as_of`, `metrics`. 현재 미국장 뉴스는 `equities.next_checks` 또는 별도 `next_kr_signals`에 넣고 직전 국내장의 원인으로 쓰지 않습니다.
- `market_context`/`equities.features` 측정값: `id`, `value`(숫자 또는 null), `unit`, `session_date`, `observed_at`, `available_at`(불명확하면 null), `status`, `source_ids`, `predetermined`(한국장 개장 전에 실제로 공개된 선행치만 true). 결측에는 `missing_reason`. 미국 지수 종가/장중, 환율 주간/야간, 거래소 구분을 명시합니다.
- `equities`: 회사별 `ticker`(005930/000660), `company`, `kr_trade_date`, `price`(venue=KRX, session=regular_close, currency=KRW, previous_close, open, close, return_pct, gap_return_pct, intraday_return_pct, source_ids), `summary`, `top_driver_ids`(0~3개), `factor_assessments`, `features`, `next_checks`. 같은 거래일을 재검토해도 history는 종목별 1행입니다. 새 국내 세션이 없으면 equities=[]와 no_new_session 상태를 쓸 수 있습니다.
- 요인: `factor_id`(registry ID), `direction`(positive/negative/mixed/unclear), `relation`(candidate_driver/confirmation/countervailing/background), `confidence`(high/medium/low), `evidence_type`(observed/reported_interpretation/hypothesis), `event_window`(before_kr_open/during_kr/after_kr_close/unknown), `event_at`(확인된 때만), `reason`, `counterevidence`, `source_ids`. `new_or_repeated`로 신규/지속 배경을 남깁니다. 사후 수급은 confirmation/countervailing으로 표시합니다.
- `sources`: 고유 id, 원문 HTTPS URL, 제목, publisher, published_at(확인 시각/null), source_type. 제목을 바꾸어 원문을 오인시키지 않습니다.
- 쇼츠 후보: `hook`, `why_surprising`, `verified_facts`, `caveat`, `next_check`, `source_ids`. 충분한 사실이 없으면 빈 배열. 영상/음성 제작·게시 작업은 하지 않습니다.

수익률은 `(종가/전일 종가-1)*100`입니다. 시초가가 없으면 갭/장중 수익률을 null로 두고 한계를 씁니다. 실제 추정 없이 요인별 기여율, 설명력 %, 매매 신호를 생성하지 않습니다. 기존 bootstrap 이후 같은 한국 거래일을 조사할 때 해당 거래일 기록을 검토하여 근거를 보완하고, 소급 시간 누수 없이 새 report_date를 기록합니다.
