# 유스케이스 다이어그램 — 어디갈래? (WSWG)

> 협업형 여행 플래너 서비스 **어디갈래?(WSWG)** 의 유스케이스 다이어그램입니다.
> 시스템 외부 액터(사용자/외부 API)와 시스템이 제공하는 기능(유스케이스) 간 관계를 정리합니다.

---

## 1. 액터 정의

| 액터 | 설명 |
| :--- | :--- |
| **게스트 (Guest)** | 비로그인 상태의 방문자. 관광 정보 열람 및 회원가입 가능 |
| **회원 (User)** | 로그인한 일반 사용자. 여행 계획을 직접 생성/수정/공유 |
| **동행인 (Companion)** | 공유 링크로 초대받아 일정 공동 편집에 참여하는 사용자 |
| **관리자 (Admin)** | 관광지 마스터 데이터를 관리하는 운영 주체 |
| **한국관광공사 API** | 관광지 원천 데이터를 제공하는 외부 시스템 |

---

## 2. 다이어그램

```mermaid
flowchart LR
    %% ===== Actors =====
    Guest(("👤<br/>게스트"))
    User(("👤<br/>회원"))
    Companion(("👤<br/>동행인"))
    Admin(("👤<br/>관리자"))
    TourAPI[["🌐<br/>한국관광공사 API"]]

    %% ===== System Boundary =====
    subgraph WSWG["🧳 어디갈래? (WSWG) 시스템"]
        direction TB

        subgraph AUTH["인증"]
            UC1(회원가입)
            UC2(로그인)
            UC3(로그아웃)
        end

        subgraph TOUR["관광 정보"]
            UC4(시/도·구/군 조회)
            UC5(관광지 목록 조회)
            UC6(관광지 상세 보기)
            UC7(키워드 검색)
        end

        subgraph PLAN["여행 계획"]
            UC8(여행 계획 자동 생성)
            UC9(여행 계획 조회)
            UC10(여행 계획 수정)
            UC11(여행 계획 삭제)
            UC12(공유 링크 생성)
        end

        subgraph COLLAB["협업"]
            UC13(공유 링크로 참여)
            UC14(일정 항목 실시간 편집)
            UC15(편집 내용 동기화)
        end

        subgraph MYPAGE["마이페이지"]
            UC16(내 여행 목록 조회)
            UC17(참여중 여행 조회)
        end

        subgraph ADMIN_UC["관리"]
            UC18(관광지 데이터 CRUD)
        end
    end

    %% ===== Guest =====
    Guest --- UC1
    Guest --- UC2
    Guest --- UC4
    Guest --- UC5
    Guest --- UC6
    Guest --- UC7

    %% ===== User =====
    User --- UC3
    User --- UC8
    User --- UC9
    User --- UC10
    User --- UC11
    User --- UC12
    User --- UC16
    User --- UC17

    %% ===== Companion =====
    Companion --- UC13
    Companion --- UC14

    %% ===== Admin =====
    Admin --- UC18

    %% ===== External =====
    UC5 -. "&laquo;include&raquo;" .-> TourAPI
    UC6 -. "&laquo;include&raquo;" .-> TourAPI
    UC8 -. "&laquo;include&raquo;" .-> TourAPI

    %% ===== Include / Extend =====
    UC8 -. "&laquo;include&raquo;" .-> UC5
    UC10 -. "&laquo;include&raquo;" .-> UC15
    UC14 -. "&laquo;include&raquo;" .-> UC15
    UC12 -. "&laquo;extend&raquo;" .-> UC13

    %% ===== Styling =====
    classDef actor fill:#FFE4B5,stroke:#FF8C00,stroke-width:2px,color:#000
    classDef external fill:#E6E6FA,stroke:#9370DB,stroke-width:2px,color:#000
    classDef usecase fill:#E0F7FA,stroke:#00838F,stroke-width:1.5px,color:#000

    class Guest,User,Companion,Admin actor
    class TourAPI external
    class UC1,UC2,UC3,UC4,UC5,UC6,UC7,UC8,UC9,UC10,UC11,UC12,UC13,UC14,UC15,UC16,UC17,UC18 usecase
```

---

## 3. 액터-유스케이스 권한 매트릭스

| 유스케이스 | Guest | 회원 | 동행인 | 관리자 |
| :--- | :---: | :---: | :---: | :---: |
| 회원가입 | ✅ | | | |
| 로그인 | ✅ | | | |
| 로그아웃 | | ✅ | ✅ | ✅ |
| 시/도·구/군 조회 | ✅ | ✅ | ✅ | ✅ |
| 관광지 목록 조회 | ✅ | ✅ | ✅ | ✅ |
| 관광지 상세 보기 | ✅ | ✅ | ✅ | ✅ |
| 키워드 검색 | ✅ | ✅ | ✅ | ✅ |
| 여행 계획 자동 생성 | | ✅ | | |
| 여행 계획 조회 | | ✅ | ✅ | |
| 여행 계획 수정 | | ✅ | | |
| 여행 계획 삭제 | | ✅ | | |
| 공유 링크 생성 | | ✅ | | |
| 공유 링크로 참여 | | | ✅ | |
| 일정 항목 실시간 편집 | | ✅ | ✅ | |
| 편집 내용 동기화 | | ✅ | ✅ | |
| 내 여행 목록 조회 | | ✅ | | |
| 참여중 여행 조회 | | ✅ | | |
| 관광지 데이터 CRUD | | | | ✅ |

---

## 4. 주요 관계 설명

### Include 관계 (`<<include>>`)
유스케이스 실행 시 **반드시 함께 수행**되는 종속 관계.

- **여행 계획 자동 생성** → **관광지 목록 조회**: 일정 추천을 위한 관광지 데이터 조회가 필수
- **여행 계획 수정 / 실시간 편집** → **편집 내용 동기화**: 모든 편집 동작은 Socket.io 기반 동기화를 동반
- **관광지 조회/상세/자동 생성** → **한국관광공사 API**: 원천 데이터 의존

### Extend 관계 (`<<extend>>`)
**선택적으로 확장**되는 관계.

- **공유 링크로 참여** ← **공유 링크 생성**: 회원이 공유 링크를 생성한 경우에만 동행인의 참여 유스케이스가 활성화됨

---

## 5. 시나리오 예시 — 여행 계획 협업 생성

1. **회원**이 로그인 후 `여행 계획 자동 생성` (지역·기간·인원·스타일 입력)
2. 시스템이 내부적으로 `관광지 목록 조회`를 수행 (한국관광공사 API include)
3. 생성된 일정에 대해 `공유 링크 생성`
4. **동행인**이 링크를 통해 `공유 링크로 참여`
5. 회원과 동행인이 동시에 `일정 항목 실시간 편집` 수행
6. 모든 변경은 `편집 내용 동기화`를 통해 Redis/Socket.io로 즉시 전파
