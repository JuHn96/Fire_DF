**차트관리탭 · 스크린샷 저장 · 1/2단계 이벤트 · 16×16 CCTV 구조 · 드래그로 위치 변경** 전부 다 **설계에 자연스럽게 포함 가능**
조금 더 명확하게, “어떻게 넣을지”를 정리

---

## 1. 차트탭에 들어갈 내용 정리 (로그 + 스크린샷)

### 1) 저장 로직 정리 – “이벤트만 저장, 나머지는 드랍”

* 평소에는:

  * **실시간 스트리밍만** 하고
  * DB/디스크에는 **아무것도 저장 안 함**
* 화재 이벤트 발생 시에만:

  1. AI가 해당 프레임에서 **감지 단계(1단계/2단계)** 판단
  2. 그 프레임을 **캡처(스크린샷)** 해서
  3. BE에 `POST /api/fire-events` 로 전송
  4. BE는

     * `fire_events` 테이블에 로그 1건 저장
     * 캡처 이미지는 파일로 저장하고 `screenshot_url` 만 DB에 기록

이렇게 하면 말한 것처럼 **이벤트 안 나면 다 드랍**,
**이벤트 날 때만 로그 + 캡처 저장** 구조가 깔끔하게 맞아요.

---

### 2) fire_events 테이블 확장 예시

```sql
CREATE TABLE fire_events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    camera_id INT NOT NULL,
    level TINYINT NOT NULL,          -- 1단계(주의), 2단계(경고) 같은 숫자 레벨
    message VARCHAR(255) NOT NULL,   -- "연기 감지", "불꽃 감지" 등
    screenshot_url VARCHAR(255),     -- 캡처 이미지 경로
    detected_at DATETIME NOT NULL,   -- 감지된 시간
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (camera_id) REFERENCES cameras(id)
);
```

* **로그 뜬 시간** → `detected_at`
* **단계(1단계/2단계)** → `level`
* **캡처 이미지** → `screenshot_url`

차트탭에서 이 테이블을 기준으로:

* 왼쪽: 시간순 이벤트 리스트 (시간, 카메라, 단계, 메시지)
* 오른쪽:

  * 일자별/카메라별 그래프
  * 선택한 이벤트의 캡처 이미지 미리보기

이렇게 만들면 됩니다.

---

## 2. “단계는 따로 저장, 화면에는 같이 표시” 이 부분

> 저장은 따로 저장되고 프로그램에 표시는 같이 뜨는데 단계표시만 되게 하면 될것같거든

이거는 이렇게 처리하면 깔끔합니다.

1. DB에는:

   * `level`(1/2) / `screenshot_url` / `detected_at` 등 상세 정보 저장
2. 차트/로그 탭 UI에서는:

   * 리스트에 `1단계 / 2단계`만 간단히 표시
   * 행을 클릭하면 아래쪽에 상세 (캡처 이미지, 위치, 메시지…) 노출

→ 저장은 상세하게,
→ **표시는 “단계 위주 + 선택 시 상세”** 구조로 가면 됩니다.

---

## 3. 계정 관리 방식 (자체 관리 프로그램)

> 프로그램 쓰는 곳에서 계정을 관리하는 형식

이건 이렇게 나누면 깔끔합니다.

1. **사용자용 화면**

   * 로그인 화면
   * 회원가입 화면
   * 마이페이지(정보수정)

2. **관리자용 화면**

   * 사용자 목록 (검색/정렬)
   * 사용자 활성/비활성
   * 권한(관리자 / 관제요원 등) 변경

DB 구조는:

```sql
users (id, login_id, password_hash, name, company, dept, enabled, created_at ...)
roles (id, name)                -- ADMIN, OPERATOR 등
user_roles (user_id, role_id)   -- N:M
```

지금 캡처에 있는 **회원가입 / 로그인 / 정보수정 화면**은 전부 `users` 테이블 기준으로 맞춰가면 됩니다.

---

## 4. CCTV 화면 구조 (메인 16묶음 + 서브메인 16개)

말한 구조를 테이블로 정리해보면:

### 1) 개념 정리

* **메인 화면**

  * “1번 묶음 ~ 16번 묶음”이 4×4 식으로 보이는 화면
  * 각 칸에는 그 묶음의 **대표 CCTV 썸네일** 1개씩만 보여줌
  * 더블클릭 시 그 묶음의 서브메인으로 이동

* **서브메인 화면**

  * 선택한 묶음의 CCTV 16개(1~16)가 4×4로 나열
  * 드래그 앤 드롭으로 위치 바꾸기 가능

### 2) DB 구조 예시

```sql
-- 묶음(메인 1~16)
CREATE TABLE cctv_groups (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL  -- "메인 1", "메인 2" ...
);

-- 각 묶음에 속한 CCTV
CREATE TABLE cameras (
    id INT AUTO_INCREMENT PRIMARY KEY,
    group_id INT NOT NULL,
    slot_index INT NOT NULL,      -- 1~16 (서브메인에서 칸 위치)
    name VARCHAR(50),
    rtsp_url VARCHAR(255),
    location VARCHAR(100),
    is_main_thumbnail TINYINT(1) DEFAULT 0,  -- 메인 화면 썸네일 여부
    enabled TINYINT(1) DEFAULT 1,
    FOREIGN KEY (group_id) REFERENCES cctv_groups(id)
);
```

### 3) 메인 화면 동작

* 각 그룹(1~16)에 대해:

  * `is_main_thumbnail = 1`인 카메라 1개를 찾아서 썸네일로 표시
  * 없으면 “No CCTV” 같은 빈 슬롯 표시

→ 캡처에 있었던 **“메인 CCTV 관리” 화면**은
이 `is_main_thumbnail` / `location` / `rtsp_url` 등을 수정하는 페이지라고 보면 됩니다.

---

## 5. 서브메인에서 드래그로 CCTV 위치 변경

> 서브메인에서 드래그로 CCTV 위치 변경이 되게 만들거야

이건 FE + BE 둘 다 처리가 필요합니다.

1. **FE(Vue)**

   * 4×4 그리드에 CCTV 카드 16개 렌더링
   * 드래그 앤 드롭 라이브러리(예: `vue-draggable-next`) 사용
   * 드래그가 끝나면 **새 slot_index 순서를 배열로 BE에 전송**

2. **BE API 예시**

```http
PUT /api/cctv-groups/{groupId}/reorder
Body:
[
  { "cameraId": 5, "slotIndex": 1 },
  { "cameraId": 2, "slotIndex": 2 },
  ...
]
```

BE에서는 이 순서에 맞게 `cameras.slot_index`를 업데이트만 해주면 됩니다.

---

## 6. 차트/로그 탭 UX 스케치

아직 디자인 안 한 차트탭은 이런 형태 추천드립니다.

* 상단 필터 영역

  * 기간 선택(오늘/1주일/1달/직접 입력)
  * 카메라 선택
  * 단계(1/2) 필터

* 좌측: **이벤트 로그 테이블**

  * 시간, 카메라, 단계, 메시지
  * 행 클릭 시 선택 상태

* 우측 상단: **그래프**

  * 시간대별 이벤트 빈도 (막대/라인)
  * 카메라별/위치별 이벤트 수

* 우측 하단: **선택 이벤트 상세**

  * 캡처 이미지
  * CCTV 이름/위치
  * 단계, 메시지, 감지 시간

이렇게 하면

> “로그(시간 포함) + 객체검출 캡처 + 단계 표시”
> 요구사항을 한 화면에서 전부 만족할 수 있습니다.

---

정리하면,

* ✅ 이벤트 없으면 아무것도 안 저장 → OK
* ✅ 이벤트 발생 시에만 로그 + 이미지 + 단계 저장 → DB/파일 구조로 설계 가능
* ✅ 프로그램 첫 화면 16묶음 / 서브메인 16개 / 썸네일 관리 / 드래그 위치 변경 → 전부 DB 설계 + API로 풀 수 있음
* ✅ 계정/회원관리, 정보수정, 로그인/회원가입 → 지금 캡처 구조 그대로 `users`/`roles` 테이블과 연결하면 됨
