# AI 사용 보고서 — 어디갈래?(WSWG)

> 산출물 ③-2. 본 프로젝트에서 생성형 AI를 **어디에·어떻게·어떤 규칙으로** 활용했는지 정리한다.
> 발표 PPT에 요약 슬라이드로 포함 가능.

---

## 1. 사용한 AI 도구

| 도구 | 주 용도 |
|------|---------|
| **Claude Code** | 설계 문서 작성, 화면 목업(HTML) 생성, 백엔드/프론트 구현, 리팩터링 |
| **OpenAI Codex (CLI)** | 독립적 코드 리뷰·검증("second opinion"), 막힌 버그 디버깅 |
| **Gemini** | 보조 검토 |
| **(런타임) OpenAI 임베딩** | 제품 기능 자체 — 관광지 의미 검색(`text-embedding-3-small`, pgvector) |

> 앞의 셋은 **개발 도구**로서의 AI, 마지막은 **제품에 내장된 AI 기능**이다.

---

## 2. AI 협업 규칙 — `AGENTS.md` 단일 원본

여러 AI 도구가 같은 코드베이스를 만지므로, **작업 규칙을 한 파일로 통일**했다.

- `backend/AGENTS.md`, `wswg_frontend/AGENTS.md` 가 **단일 원본(single source of truth)**.
- `CLAUDE.md`, `GEMINI.md` 는 이 파일을 `@import`로 가리키기만 함 → 규칙 분산·드리프트 방지.
- 규칙에 담은 것: 기술 스택, 디렉토리 구조, 도메인 네이밍, **GitHub Flow + rebase 후 일반 merge**, 커밋 컨벤션, **TDD(테스트 먼저)**, 금지 사항(`main` 직접 push·시크릿 커밋 금지).

> 효과: 사람이든 AI든 동일한 컨벤션으로 커밋·PR을 만들어, 리뷰 비용과 충돌을 줄였다.

---

## 3. 활용 영역별 상세

### 3.1 설계 문서
- 요구사항 정의서(SRS), 유스케이스 명세(22종), 화면 정의서, ERD, **클래스 다이어그램·간트 차트**를 코드/스키마 기반으로 작성·동기화.
- 예: "`schema.sql`을 읽고 ERD를 12테이블로 동기화하라" → 누락된 `attraction_embeddings`·`group_join_requests`·`batch_run_log` 반영.

### 3.2 화면 디자인 (목업)
- [claude-design-prompts.md](claude-design-prompts.md): S1~S9 + 관리자 **전 화면 생성 프롬프트 모음**.
- **§0 공통 제약**(디자인 토큰·브랜드 컬러·컴포넌트 강제)을 모든 프롬프트 앞에 붙여 일관된 9종 목업 산출.

### 3.3 백엔드 구현
- TourAPI 적재(write-through 캐시), AI 자동생성 3단계 파이프라인, 실시간 공동편집(WebSocket+Redis Stream) 등 핵심 로직 구현·리팩터링.
- **TDD 준수**: 새 로직 PR은 Testcontainers(PostgreSQL/PostGIS) 통합 테스트 동반.

### 3.4 프론트 구현
- 화면 10종, 서비스 레이어(`VITE_USE_MOCK` 모킹), WebSocket 클라이언트(camelCase 어댑터), Naver Maps 연동.

### 3.5 코드 리뷰·디버깅
- Codex로 독립 리뷰(SQL 안전성·side effect·trust boundary)와 막힌 버그의 root-cause 분석.

---

## 4. 대표 프롬프트 예시

```text
# 설계 동기화
"backend/src/main/resources/db/postgres/schema.sql을 읽고,
 docs/ERD.md와 비교해 누락된 테이블을 찾아 ER 다이어그램을 최신화해줘.
 Mermaid erDiagram으로, FK·UNIQUE·CHECK 제약을 주석으로 표기."

# 화면 목업(공통 제약 + 화면별)
"[§0 공통 제약: 디자인시스템.md 토큰만 사용, 브랜드블루 #2C6FE3, Pretendard …]
 위 제약 하에 S6 여행 편집(협업) 화면을 데스크탑+모바일 HTML로 만들어줘."

# 구현 + TDD
"관광지 상세 API에 write-through 캐시를 추가하되,
 외부 둘 중 하나 실패 시 502를 반환. 실패하는 테스트부터 작성(Red→Green)."

# 독립 리뷰
"이 PR diff를 Codex로 리뷰: SQL 인젝션·조건부 side effect·트랜잭션 경계 위주로."
```

---

## 5. 제품 내장 AI 기능 (런타임)

| 기능 | AI 활용 |
|------|---------|
| **AI 여행 자동 생성** | GPT가 사용자 조건(지역·기간·스타일)으로 일자별 후보 동선 생성 |
| **의미 기반 관광지 추천** | OpenAI 임베딩(1536차원)으로 관광지를 벡터화, pgvector HNSW 코사인 검색으로 "분위기" 유사 추천 |

---

## 6. 효과와 한계

### 효과
- **설계-구현 정합성**: 코드/스키마를 직접 읽혀 문서를 만들어 드리프트 감소.
- **속도**: 15일 일정에 핵심 기능 4종 + 문서 산출물 다수를 병행.
- **일관성**: AGENTS.md 규칙으로 다중 AI·다중 작업자 간 컨벤션 통일.

### 한계·대응
- AI 산출물은 **항상 사람이 검증** — TDD 테스트·Codex 교차 리뷰·PR 승인 게이트로 보완.
- 외부 키/시크릿은 AI에 노출·커밋 금지(`.gitignore`·`.env`), 환경변수로만 주입.
- 다이어그램 PNG 변환 등 일부 수작업(mermaid.live 내보내기)은 사람이 마무리.

---

> 참고 문서: [AGENTS.md(backend)](../backend/AGENTS.md) · [AGENTS.md(frontend)](../wswg_frontend/AGENTS.md) · [claude-design-prompts.md](claude-design-prompts.md)
