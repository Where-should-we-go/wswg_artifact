# `trips.data` JSONB 스키마 — 일정 뷰 계약서

**범위**: 여행 문서 `trips.data`(JSONB) 안의 일정 항목. 같은 원본을 **① 일정(레일+노드)** 과 **② 캘린더(시간 그리드)** 두 뷰가 함께 읽는다.
**기준 문서**: [docs/여행기록도메인.md](../docs/여행기록도메인.md)(도메인 정답), [docs/데이터사전.md](../docs/데이터사전.md). 이 문서는 프론트 일정 뷰가 의존하는 **필드 계약**을 구체화한 것.
**확정일**: 2026-06-22

---

## 1. 전체 형태

```jsonc
trips(10) = {
  trip_id, title, group_id, user_id,
  start_date: "2026-07-01", end_date: "2026-07-03",
  data: {                 // ← JSONB 컬럼
    items: [ Item, Item, ... ]
  }
}
```

> `members`/`presence`(협업)·`cover`/`icon`/`region`/`styles`/`budgetLabel`(페이지 메타)는
> 프론트 표현용. 영속 저장 위치는 백엔드와 별도 합의(일부는 컬럼, 일부는 data 메타).
> **이 문서의 핵심은 `data.items[]` 한 항목(Item)의 스키마다.**

---

## 2. Item 스키마 (블록 1개)

| 필드 | 타입 | 필수 | 설명 |
|------|------|:---:|------|
| `id` | string | ✅ | 블록 고유 ID(프론트 정렬·협업 carret 타깃). 예 `"b-1"` |
| `content_id` | number \| null | ✅ | 관광지(attractions 소프트참조)면 number, **TourAPI에 없으면 `null`**(직접 추가 맛집 등) |
| `title` | string | ✅ | 블록 제목 |
| `type` | enum(한글) | ✅ | `관광` \| `식당` \| `이동` \| `숙소` \| `메모` (UI 색/이모지는 typeKey 헬퍼로 매핑) |
| `visitDate` | "YYYY-MM-DD" | ✅ | **일차 그룹핑 키.** start_date 기준 Day N 산출 |
| `order` | number | ✅ | 같은 날 내 정렬 순서(시간 없을 때의 기준) |
| `time` | "HH:mm" \| null | ⬜ | 시작 시각. **없으면 "시간 미정"** (비정형 허용) |
| `durationMin` | number \| null | ⬜ | **소요시간(분).** 캘린더 블록 높이·일정 오버라인 라벨의 정본 |
| `lat` / `lng` | number | ⬜ | 좌표(장소 항목만; 이동·메모는 없을 수 있음). 지도 뷰용 |
| `media` | Media[] | ⬜ | 사진/영상 배열(기본 `[]`) |
| `properties` | object | ⬜ | 자유 필드(아래 §4) |

### Media
```jsonc
{ "type": "PHOTO" | "VIDEO", "url": "string", "metadata": { "w":4032,"h":3024 } /* or {durationSec} */ }
```

---

## 3. 시간 관련 규칙 (두 뷰 공통)

- **시작/종료**: 시작 = `time`, 종료 = `time + durationMin`. `endTime`은 **저장 안 함, 파생**.
- **`time` 없음** → 일정 뷰: 오버라인에 "시간 미정" / 캘린더 뷰: 상단 **"시간 미정" 트레이**.
- **`durationMin` 없음** → 캘린더는 기본 높이(예: 60분)로 렌더. 일정 오버라인은 소요시간 라벨 생략.
- **소요시간 라벨**: `durationMin`을 사람이 읽는 문자열로 포맷(예: 60→"1시간", 90→"1시간 30분", 30→"30분"). ⚠️ 문자열 `duration:"1시간"`을 따로 저장하지 말 것 — `durationMin` 하나가 정본.

### 구간(daypart) 파생 — 저장 안 함, `time`에서 계산
| 구간 | 조건 | 노드 |
|------|------|------|
| 오전 | `time` < 12:00 | 🌅 오전 |
| 오후 | 12:00 ≤ `time` < 18:00 | ☀️ 오후 |
| 저녁 | `time` ≥ 18:00 | 🌆 저녁 |
| 미정 | `time` 없음 | (노드 끝 또는 "시간 미정" 묶음) |

> 일정 뷰의 구간 노드는 이 규칙으로 자동 생성. `type:"이동"`은 노드 분류에서 제외하고 앞 블록 뒤 회색 스트립으로.

---

## 4. `properties` 자유 필드 (관례)

| 키 | 타입 | 쓰는 타입 | UI |
|----|------|----------|----|
| `memo` | string | 전부 | 📝 pill / 콜아웃 |
| `budget` | number(원) | 식당·숙소·관광 | 💰 9,000원 (천단위) |
| `rating` | number(0–5) | 식당·관광 | ⭐ 5.0 |
| `region` | string | 장소 | 위치 meta |
| `transport` | string | 이동 | 🚄 회색 스트립 본문 |
| `checkIn` / `checkOut` | "HH:mm" | 숙소 | 체크인/아웃 |
| `예약번호` | string | 이동·숙소 | 🎫 pill |

> `properties`는 **확장 가능**. 위는 현재 합의된 관례이며, 새 키는 추가만 하고 제거는 마이그레이션 필요.

---

## 5. 두 뷰가 읽는 법

| | 일정(레일+노드) | 캘린더(시간 그리드) |
|---|---|---|
| 일차 | `visitDate` → Day N 섹션 | `visitDate` → 컬럼 |
| 정렬 | `order`(시간 있으면 time 우선) | `time` 위치 |
| 시간 | 오버라인 `time · durationMin라벨` | top=`time`, height=`durationMin` |
| 구간 | `time`→daypart 노드 | (없음) |
| 미정 | 오버라인 "시간 미정" | 상단 트레이 |
| 이동 | 회색 스트립 | 얇은 스트립 |

---

## 6. 예시 (부산 2박3일 발췌)

```jsonc
{
  "id": "b-1", "content_id": 126508, "title": "해운대 해수욕장", "type": "관광",
  "lat": 35.158, "lng": 129.16,
  "visitDate": "2026-07-01", "time": "10:00", "durationMin": 60, "order": 1,
  "media": [{ "type": "PHOTO", "url": "...", "metadata": { "w":4032,"h":3024 } }],
  "properties": { "memo": "일출 명소", "region": "부산 해운대구" }
}
// → 일정: [오전] 점, "10:00 · 1시간" / 캘린더: Day1 10:00~11:00 블록

{
  "id": "b-2", "content_id": null, "title": "속씨원한 돼지국밥", "type": "식당",
  "visitDate": "2026-07-01", "time": "12:30", "durationMin": 60, "order": 2,
  "properties": { "budget": 9000, "rating": 5 }   // content_id null → "직접 추가"
}

{
  "id": "b-7", "content_id": null, "title": "야경 동선 다시 보기", "type": "메모",
  "visitDate": "2026-07-02", "time": null, "durationMin": null, "order": 3,
  "properties": { "memo": "준영이 카메라 챙기기" }
}
// → 일정: "시간 미정" / 캘린더: 미정 트레이
```

---

## 7. 백엔드 연동 메모 (TODO)
- `data`는 `trips.data` **JSONB 컬럼**. 조회 `GET /trips/:id` → `data` 그대로.
- 편집: 항목 추가/수정/이동(time·durationMin·order 변경)은 **Redis presence/batch flush** 후 JSONB upsert (도메인 §5).
- 협업 carret: presence 채널(`{memberId, blockId}`) 수신 → 해당 `id` 블록에 표시.
- `content_id @>` GIN 검색으로 "이 관광지 간 여행" 조회 가능(도메인 §3-③).
