# 국내 여행지 추천 프로그램 (API 활용)

날짜만 입력하면 **LLM이 여행지를 추천**하고, **지도 API로 맛집을 검색**한 뒤,
**LLM이 최종 여행 리포트(Markdown)** 를 자동 생성해주는 CLI 프로그램입니다.

---

## 1. 프로그램 개요
사용자가 여행 날짜(`YYYY-MM-DD`)를 입력하면, 여러 API를 조합하여 여행 리포트를 만들어냅니다.
단일 API 호출이 아니라 **LLM → 지도 API → LLM** 순으로 데이터를 이어붙여 인사이트를 도출하는 것이 핵심입니다.

## 2. 주요 기능 / 동작 흐름
```
[1/3] LLM으로 1차 추천 생성 (추천 도시 · 날씨 · 행사 · 이유를 JSON으로 출력)
        ↓  recommended_city 를 다음 단계 입력으로 전달
[2/3] 지도 API로 맛집 검색 (추천 도시 기준 맛집 N곳)
        ↓  맛집 목록(0건 가능)을 다음 단계 입력으로 전달
[3/3] LLM으로 최종 리포트 생성 (1차 JSON + 맛집 목록 → Markdown)
        ↓
results/ 폴더에 원본 JSON + 최종 리포트(.md) 저장
```

### 2-1. 단계별 모듈 / 함수 I/O 계약 (평가 #6)
| 단계 | 함수(예) | 입력 | 출력 | 실패 시 |
|------|----------|------|------|---------|
| 1차 추천 | `get_recommendation(date)` | `date: str` | `dict`(1차 JSON) | 파싱 실패 → 1회 재시도 후 종료 |
| 맛집 검색 | `search_places(city)` | `city: str` | `list[dict]`(맛집, 0건 가능) | 실패 → 빈 리스트 + errors 누적 |
| 리포트 생성 | `build_report(rec, places, errors)` | 1차 JSON + 맛집 + 오류목록 | `str`(Markdown) | 파싱 실패 → 1회 재시도 |
| 저장 | `save_results(date, raw, report)` | 원본·리포트 | 파일 경로 | 폴더 없으면 자동 생성 |

> 각 단계는 **독립 함수/모듈로 분리**하여, 한 단계의 입력은 이전 단계의 출력이 되도록 설계합니다.

## 3. 개발 환경 / 요구 사항
- Python **3.10 이상**
- 터미널(CLI)에서 실행 (웹 UI 없음)
- 사용 라이브러리: `requests`, `python-dotenv` 등
```
requests
python-dotenv
```

## 4. 설치 방법
```bash
git clone (저장소 주소)
cd (폴더명)
pip install -r requirements.txt
```

## 5. API 키 설정 방법 (필수)

### 5-1. 사용한 API
- **LLM API**: (OpenAI / Gemini 중 택1 — 실제 사용한 것 기재)
- **지도/장소 API**: (Kakao Local / Naver Local 중 택1 — 실제 사용한 것 기재)

### 5-2. 환경변수 / .env 설정 방법
`.env` 파일을 프로젝트 루트에 만들고 아래처럼 작성합니다. **실제 키 값은 절대 커밋하지 마세요.**
```bash
# .env (자리표시자만 사용! 실제 키 금지)
OPENAI_API_KEY="YOUR_KEY"
KAKAO_API_KEY="YOUR_KEY"
```
```bash
# macOS / Linux (현재 세션에만 적용)
export OPENAI_API_KEY="YOUR_KEY"

# Windows PowerShell (현재 세션에만 적용)
$env:OPENAI_API_KEY="YOUR_KEY"
```

### 5-3. 운영/CI 환경에서의 키 관리 (평가 #13)
- CI/CD에서는 `.env` 파일 대신 **Secrets / 환경변수 주입** 기능을 사용합니다.
  (예: GitHub Actions → `Settings > Secrets`에 등록 후 `env:`로 주입)
- 키 교체 시 코드 수정 없이 환경변수 값만 변경하면 됩니다.

### 5-4. API 키 발급처
- LLM 키: (OpenAI/Gemini 발급 페이지 안내)
- 지도 키: (Kakao Developers / Naver Developers 안내)

## 6. 실행 방법
```bash
python travel_planner.py --date "2025-03-15"
# 단축형
python travel_planner.py -date "2025-03-15"
```
실행 로그 예시:
```
[1/3] 1차 추천 생성 중(LLM)...  - recommended_city: "제주"
[2/3] 맛집 검색 중...           - 맛집 5곳 검색 완료
[3/3] 최종 리포트 생성 중...    - 리포트 생성 완료
완료! results/2025-03-15_travel_plan.md 를 확인하세요.
```

## 7. 입력값 규칙 (평가 #1)
- 형식: `"YYYY-MM-DD"` (필수 옵션)
- 형식이 올바르지 않으면 **사용법(usage)을 출력하고 종료**합니다.
```python
# 검증 예시
import argparse, datetime
p = argparse.ArgumentParser()
p.add_argument("--date", "-date", required=True)
args = p.parse_args()
try:
    datetime.datetime.strptime(args.date, "%Y-%m-%d")
except ValueError:
    p.error('날짜 형식은 "YYYY-MM-DD" 여야 합니다.')  # usage 출력 후 종료
```

## 8. REST API · GET/POST 사용 규칙 (평가 #10)
| API | 메서드 | 이유 | 예시 엔드포인트 |
|-----|--------|------|-----------------|
| LLM(추천/리포트) | **POST** | 프롬프트(본문 데이터)를 전송해 결과를 생성하므로 body가 필요 | `POST https://api.openai.com/v1/chat/completions` |
| 지도 맛집 검색 | **GET** | 검색어를 쿼리스트링으로 넘겨 데이터를 조회만 함 | `GET https://dapi.kakao.com/v2/local/search/keyword.json?query=제주 맛집` |

> **GET**: 데이터를 "조회"할 때, 파라미터를 URL 쿼리로 전달 / **POST**: 데이터를 "생성/전송"할 때, body에 담아 전달.

## 9. LLM 출력 구조화(JSON)의 이점 (평가 #11)
1차 추천을 **자유 텍스트가 아닌 JSON으로 강제**하는 이유:
- **파싱 용이**: `json.loads()`로 바로 딕셔너리 변환 → 코드에서 안전하게 값 접근
- **다음 단계 연결**: `recommended_city` 값을 그대로 지도 API의 검색어로 전달 가능
- **검증 가능**: 필수 키·타입을 프로그램에서 자동 검사할 수 있음

### 9-1. 1차 JSON 스키마 · 타입 · 예시 (평가 #2, #7)
| 키 | 타입 | 예시 값 |
|----|------|---------|
| `recommended_city` | string | `"제주"` |
| `weather` | string | `"3월 중순 평균 15°C 내외, 온화함"` |
| `events` | array of string | `["유채꽃 축제", "봄 지역 행사"]` |
| `reason` | string | `"봄꽃 즐기기 좋고 성수기 대비 비용 부담이 적음."` |

**검증 로직(평가 #7):** 각 키의 존재 여부와 타입을 확인하고, 실패 시 아래 순서로 처리합니다.
1. 필수 키 누락/타입 불일치 → "필수 키만 JSON으로 다시 출력"하도록 프롬프트 수정 후 **재시도 1회**
2. 재시도도 실패하면 → `errors`에 누적하고 종료
```python
def validate(data: dict) -> bool:
    return (isinstance(data.get("recommended_city"), str)
            and isinstance(data.get("weather"), str)
            and isinstance(data.get("events"), list)
            and isinstance(data.get("reason"), str))
```

## 10. 지도 API 추상화 설계 (평가 #8)
지도 API 교체 시(Kakao ↔ Naver) **최소 변경**만으로 대응할 수 있도록 공통 인터페이스를 둡니다.

```
[입력]  query: str (검색어)
   ↓
search_places(query) -----> (제공자별 구현: kakao_search / naver_search)
   ↓
[출력]  list[dict] — 아래 공통 필드로 정규화
        { name, address, category, url, x, y }
```
- 제공자별 응답 필드가 달라도 **동일한 출력 형식으로 변환(정규화)** 하여 반환
- 새 제공자 추가 시 `search_places` 내부 구현 함수만 추가하면 됨

## 11. 지명 표준화 / 검색어 보정 규칙 (평가 #17)
LLM이 반환한 도시명을 그대로 쓰면 검색 정확도가 떨어질 수 있어 정규화합니다.
- **동일 지명 매핑**: `"제주시" / "제주도" → "제주"`, `"강릉시" → "강릉"`
- **검색어 보정**: 도시명 뒤에 `" 맛집"` 키워드를 붙여 검색 (`"제주" → "제주 맛집"`)
- **공백/특수문자 제거** 후 매핑 테이블과 비교
```python
CITY_MAP = {"제주도": "제주", "제주시": "제주", "강릉시": "강릉"}
def normalize_city(name: str) -> str:
    name = name.strip()
    return CITY_MAP.get(name, name)
```

## 12. 실행 결과물 확인 방법 (평가 #4)
`results/` 폴더에 실행 날짜 기준으로 저장됩니다.
| 파일 | 파일명 포맷(예) | 내용 |
|------|-----------------|------|
| 원본 데이터 | `results/2025-03-15_raw.json` | 1차 JSON + 맛집 목록 + errors |
| 최종 리포트 | `results/2025-03-15_travel_plan.md` | 여행 리포트(Markdown) |

### 12-1. 원본 JSON 구조
```json
{
  "recommendation": { "recommended_city": "제주", "weather": "...", "events": ["..."], "reason": "..." },
  "places": [ { "name": "...", "address": "...", "category": "...", "url": "...", "x": 126.5, "y": 33.5 } ],
  "errors": []
}
```

### 12-2. 최종 리포트 항목 순서
추천 지역 → 추천 이유 → 날씨 요약 → 행사/축제 → 맛집 추천 → 1일 일정 제안(오전/오후/저녁) → 오류
