# 한국 전력수급 대시보드: API 자동 수집 실습 세션 자료

지난 지식공유 세션(분석 대시보드 구동 방식: Parquet · DuckDB-Wasm · Netlify)의 후속 **실습 세션** 자료입니다.

이번 세션은 **공식 API가 있는 자료만** 다룹니다. (화면에서 긁어오는 방식은 내용이 복잡해져 보류했고, 관련 자료들은 [`extras/scraping/`](extras/scraping/README.md)에 남겨 두었습니다.)

```
[매일] GitHub Actions → scripts/fetch_today.py 가 공공데이터포털 API 호출 → site/data/supply_5min.parquet 에 쌓고 커밋
        → 같은 워크플로가 site/ 를 GitHub Pages 에 배포 → 링크를 여는 브라우저가 DuckDB-Wasm 으로 계산해서 그래프를 그림
```

**핵심 메시지:** 이 API는 '오늘'만 알려 줍니다. 어제 값은 다시 받을 수 없으니 **매일 받아서 쌓는 일**이 곧 자동화입니다.

## 쓰는 API (공공데이터포털 · 한국전력거래소, 이용허락범위 제한 없음)

| | 오늘전력수급현황조회_GW (메인) | 현재전력수급현황조회_GW (맛보기) | 전력거래실적 (심화) |
|---|---|
| 상세 페이지 | [15158703](https://www.data.go.kr/data/15158703/openapi.do) |
| 주소 | `apis.data.go.kr/B552115/Sukub5mToday/getSukub5mToday` | 
| 돌려주는 것 | 오늘 0시부터 지금까지 5분 단위 | 
| 형식 | XML | 
| 요청변수 | `serviceKey` `dataType` `pageNo` `numOfRows` |

* 응답 항목(수급 2종): `baseDatetime` 기준일시 · `currPwrTot` 현재수요 · `suppAbility` 공급능력 · `forecastLoad` 최대예측수요 · `suppReservePwr` 공급예비력 · `suppReserveRate` 공급예비율(%) · `operReservePwr` 운영예비력 · `operReserveRate` 운영예비율(%). 단위는 MW.
* 개발계정은 자동 승인, **하루 100회** 호출 한도입니다.
* 두 수급 API는 **2026-06-30에 기존 PUBC 방식에서 GW 방식으로 바뀌었습니다.** 예전 블로그·예제의 주소는 동작하지 않을 수 있습니다.
* 과거 자료: 「현재전력수급현황조회」 페이지가 안내하는 파일 내려받기(https://openapi.kpx.or.kr/sukub.do, 3개월 단위)가 있습니다. 파일 형식은 이 자료에서 확인하지 못했습니다.

## 폴더 구성

```
energy-dashboard/
├─ site/                       ← GitHub Pages 가 그대로 나눠주는 폴더
│  ├─ index.html               대시보드 (DuckDB-Wasm + Plotly, SQL 실습창 포함)
│  └─ data/                    supply_5min.parquet · meta.json  (첫 수집 후에 생김)
├─ scripts/
│  ├─ kpx.py                   API 호출 + JSON/XML 응답 읽기 (키는 환경변수 KPX_API_KEY)
│  └─ fetch_today.py           오늘 5분 데이터를 받아 검증하고 파일에 쌓기
├─ notebooks/workshop.ipynb    참가자용 Colab 노트북 (연습문제 + 정답 접기)
├─ .github/workflows/daily.yml 매일 22:17 · 23:37 KST 에 수집하고 GitHub Pages 에 배포
├─ extras/scraping/            보류: 화면 뒤 주소를 찾아 받아 오는 방식의 지난 준비물
├─ extras/netlify/             보관: 예전 Netlify 배포 설정 (netlify.toml)
└─ requirements.txt            requests · pandas · pyarrow
```

## 세션 진행안 (90분)

| 시간 | 파트 | 내용 | 노트북 |
|---|---|---|---|
| 0:00 | 복습 (10) | 지난 발표 16쪽 흐름도에 **"API 호출 → 쌓기"** 단계를 추가. API = 주소 + 요청변수 → 응답 | 첫 셀 |
| 0:10 | Part 1 (15) | 인증키로 첫 호출 (JSON). 응답 읽는 법(header / body / items). 일부러 키를 틀려 오류 읽기 | Part 1 |
| 0:25 | Part 2 (20) | 오늘 하루치(XML) → DataFrame → **검증**(중복·간격·값 범위·빈 행) → 차트 | Part 2 |
| 0:45 | Part 3 (10) | Parquet 저장. 그냥 이어 붙이면 중복 → **같은 시각은 덮어쓰기**(upsert). "내일이 되면?" | Part 3 |
| 0:55 | Part 4 (15) | DuckDB SQL, pandas ↔ SQL 나란히. 일별 최대수요와 시각 · 시간대별 평균 · 빠진 곳 찾기 | Part 4 |
| 1:10 | Part 5 (5) | 배포된 대시보드의 'SQL 써보기'에 Part 4의 SQL 붙여 넣기 (표 이름 `supply` 동일) | Part 5 |
| 1:15 | Part 6 (10) | GitHub Actions: Secret 등록 · Pages 켜기 → **Run workflow** → 커밋·배포·대시보드 갱신 확인 (강사 시연) | Part 6 |
| 1:25 | Part 7 (5) | 체크리스트, 이 API의 한계. (선택) 전력거래실적 API 심화 | Part 7 |

시간이 모자라면 Part 6은 시연만 하고, 심화(Part 7)는 과제로 돌립니다.

### 사전 공지 (세션 1주 전)
1. Google 계정 (Colab 용). 노트북 링크를 함께 공지
2. **공공데이터포털 가입 → 「오늘전력수급현황조회_GW」「현재전력수급현황조회_GW」 활용신청 → 일반 인증키(Decoding) 복사.** 신청 직후에는 키가 아직 등록되지 않아 오류가 날 수 있으니 **세션 전날까지** 하도록 안내
3. (선택) 심화를 해 볼 분은 「전력거래실적」도 활용신청
4. GitHub 계정은 필수가 아닙니다. 세션 후 직접 따라 해 보고 싶은 분만. (Part 6은 강사 시연)

## 강사 사전 점검 (꼭 해보세요)

1. **실제 응답을 한 번 받아 보기.** 이 자료는 공개된 API 명세(swagger)를 보고 만들었고 **실제 인증키로 호출해 보지는 못했습니다.** 흉내 서버로 코드가 '명세대로 오는 응답'을 읽는 것까지만 확인했습니다. 키를 받으면 노트북 Part 1~2를 먼저 실행해서 응답의 모양(header / body / items / item, 날짜 표기, 빈 행 여부)이 가정과 같은지 확인하세요. 다르면 고칠 곳은 `scripts/kpx.py`의 `parse`, `scripts/fetch_today.py`의 `COLUMNS`·`to_frame`, 노트북 Part 1의 `root["body"]["items"]["item"]` 경로입니다.
2. **저장소 만들기.** `TourStat` 전체가 아니라 **`energy-dashboard` 폴더만** 올립니다. (상위 폴더에는 실태조사 원자료가 있습니다.)
3. **Secret 등록.** 저장소 *Settings → Secrets and variables → Actions* 에 `KPX_API_KEY` (Decoding 키). **Workflow permissions** 는 *Settings → Actions → General* 에서 `Read and write`.
4. **Run workflow 로 첫 수집·배포.** *Actions → collect-supply → Run workflow* 후 `update`·`deploy` 두 작업의 초록 체크와 로그의 `저장: …` 를 확인. Actions 서버는 해외에 있어 접속이 되는지 확인이 필요합니다. 막히면 강사 PC에서 `fetch_today.py` 를 돌려 push 한 뒤, Run workflow 창에서 **collect 체크를 끄고** 실행하면 수집 없이 배포만 합니다. (수집이 실패하면 `deploy` 는 건너뛰어집니다.)
5. **며칠 먼저 쌓아 두기.** 세션 며칠 전부터 수집을 켜 두면 Part 4에서 `DATA_URL`(배포된 `…/data/supply_5min.parquet`)로 여러 날 데이터를 쓸 수 있습니다.
6. **Colab에서 API 접속 확인.** 노트북 Part 1을 Colab에서 실행해 보세요. 키가 아직 안 되는 참가자를 위해 Part 2에 **연습용 가짜 응답 셀**을 넣어 두었습니다.
7. **GitHub Pages 켜기.** 저장소 *Settings → Pages → Build and deployment → Source* 를 **GitHub Actions** 로 바꿉니다. 켜지 않으면 수집은 되어도 `deploy` 작업이 실패합니다. 배포 뒤 주소는 `https://<계정>.github.io/<저장소>/` 이고 Actions 실행 화면의 `deploy` 작업에도 나옵니다. 이 주소가 Part 4의 `DATA_URL`(`…/data/supply_5min.parquet`) 앞부분이 됩니다.

## 알아둘 점

* **'오늘'만 줍니다.** 놓친 시간은 되돌릴 수 없습니다. 하루가 끝나기 전에 받아야 해서 22:17·23:37(KST)에 두 번 돌립니다. 예약 실행은 늦어질 수 있어, 그래도 **밤 11시 반쯤부터 자정까지**는 비는 것이 정상입니다. 노트북 연습 4-C에서 이 빈 곳을 SQL로 찾습니다.
* **아직 오지 않은 시각의 행**이 수요 0이나 빈 값으로 올 수 있다고 가정하고, `fetch_today.py` 는 실적이 있는 행만 쌓습니다. 실제 응답에서 확인해 주세요.
* **값이 이상하면 멈춥니다.** 비었거나, 시각을 못 읽거나, 현재수요가 1만~15만 MW 밖이면 오류로 끝나고 Actions가 빨간 X로 알려 줍니다. 같은 시각이 이미 있으면 새 값으로 덮어쓰므로 여러 번 돌려도 중복이 없습니다.
* **인증키는 코드에 쓰지 않습니다.** 환경변수(`KPX_API_KEY`)로만 읽고, 오류 메시지에도 요청 주소(키가 들어 있음)를 넣지 않습니다. 노트북은 `getpass` 입력창을 씁니다. Encoding 키를 붙여넣어도 `unquote` 로 풀어서 이중 인코딩을 피합니다.
* **저장소가 조금씩 커집니다.** 하루 두 번 파일이 바뀌어 커밋됩니다. 1년이면 파일 자체는 수 MB 안팎으로 보이지만(추정), 장기 운영하려면 월별로 파일을 나누는 것을 고려하세요.
* **배포는 GitHub Pages입니다.** 수집 → 커밋 → 배포가 한 워크플로(`daily.yml`)에서 이어집니다. (Netlify 무료 플랜은 배포 1회에 15크레딧, 월 300크레딧이라 하루 두 번 배포하면 한도에 닿아서 바꿨습니다. [공식 안내](https://docs.netlify.com/manage/accounts-and-billing/billing/billing-for-credit-based-plans/how-credits-work/) 설정은 `extras/netlify/` 에 보관했습니다.)
  * 무료로 쓰려면 **공개 저장소**여야 합니다. 비공개 저장소의 Pages는 유료 플랜에서만 되는 것으로 알고 있으니 확인하세요. 코드와 수집한 데이터가 공개된다는 뜻이기도 합니다.
  * 대시보드(`site/index.html`)를 고친 뒤에는 *Run workflow* 에서 **collect 체크를 끄고** 실행해 다시 배포하세요. (push 만으로는 배포되지 않습니다.)
  * Netlify처럼 캐시 헤더를 직접 정할 수 없어, 갱신이 몇 분 늦게 보일 수 있습니다.
* 공개 저장소는 60일간 활동이 없으면 예약 실행(cron)이 꺼질 수 있습니다.
* 출처 표기(한국전력거래소, 공공데이터포털)는 대시보드 하단에 들어 있습니다.

## 로컬에서 실행

```bash
pip install -r requirements.txt
export KPX_API_KEY='공공데이터포털 인증키(Decoding)'
python scripts/fetch_today.py                       # 오늘 5분 데이터를 받아 site/data/ 에 쌓기
python -m http.server --directory site 8000         # http://localhost:8000
```

## 이 자료와 지난 세션의 연결

| 지난 세션 (분석 대시보드 구동 방식) | 이번 실습 |
|---|---|
| SQL = DB 에게 보내는 주문서, pandas 와 같은 일 (5쪽) | Part 4 에서 pandas / SQL 나란히 실행 |
| Parquet = 계산 재료, JSON = 참고 정보 (12쪽) | 5분 수급값은 조건에 따라 달라지는 값이므로 Parquet. 수집 시각 같은 메타 정보(`meta.json`)는 JSON |
| DuckDB-Wasm + Netlify, 서버 없이 브라우저가 계산 (14쪽) | 같은 구조의 `site/index.html` (호스팅만 GitHub Pages) |
| 내 컴퓨터 → GitHub push → Netlify (16쪽) | 그 앞에 **GitHub Actions 가 API를 호출해 push 를 대신해 주는 단계**를 추가하고, 배포도 같은 워크플로가 GitHub Pages 로 합니다 |
