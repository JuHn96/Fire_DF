# 📘 웹 영상 스트리밍 · CCTV · 프레임 처리 · 해시 · 정규화(NF) 총정리

---

## 1. VideoCapture / 프레임 개념 정리

### 1-1. VideoCapture란?

- OpenCV에서 카메라 또는 영상 스트림을 받아오는 객체입니다.
- `cap = cv2.VideoCapture(0)` 처럼 생성합니다.
- `ret, frame = cap.read()` 를 호출할 때마다 **프레임 1장(이미지 1장)** 을 가져옵니다.
- `frame` 은 `(height, width, 3)` 형태의 BGR 이미지 배열입니다.

### 1-2. FPS와 프레임 간격

- 카메라에는 **FPS(Frames Per Second)** 가 있습니다.
  - 30 FPS → 1초에 30장
  - 15 FPS → 1초에 15장
- 프레임 간 시간 간격 = `1 / FPS` 초
  - 30 FPS → 약 0.033초마다 1프레임
- Python 코드가 느려서 `read()` 호출이 늦어지면,
  - 카메라는 계속 프레임을 만들고,
  - `read()` 는 **그 시점의 최신 프레임만 가져오고, 중간 프레임은 버려질 수 있음**.

### 1-3. “1초 쉬고 다시 read() 하면?”

- 카메라는 그 1초 동안 계속 프레임들을 생산하고 있음.
- 우리가 1초 동안 아무것도 안 하다가 `read()`를 호출하면  
  **그동안 쌓였던 것 중 가장 최신 프레임 1장만 가져온다.**
- 과거 프레임을 하나씩 다 처리하는 구조가 아니라  
  “항상 최신 화면 하나를 읽어오는 구조”라고 이해하면 편합니다.

---

## 2. CCTV / IP 카메라 스트리밍 구조

### 2-1. IP CCTV 내부 구조

- 요즘 사용하는 **IP 카메라(CCTV)** 는 대부분 내부에 작은 컴퓨터가 들어 있습니다.
  - OS(리눅스 계열)
  - 영상 인코더(H.264/H.265 등)
  - **RTSP / HTTP / ONVIF 서버 기능**
- 그래서 이런 주소로 접근할 수 있습니다.

예시 RTSP URL:

```text
rtsp://user:password@192.168.0.50:554/h264/ch1/main/av_stream
```

- `user:password` : 카메라 로그인 계정
- `192.168.0.50` : 카메라 IP
- `554` : RTSP 기본 포트
- `/h264/ch1/main/av_stream` : 제조사별 스트림 경로

### 2-2. “CCTV에 스트리밍 서버가 있는가?”

- **IP 카메라 기준으로는 YES.**
  - 카메라 안에 RTSP/HTTP 스트리밍 서버가 내장되어 있고,
  - Python, NVR, 관제 프로그램, 모바일 앱이 **클라이언트**가 되어 접속합니다.
- 반대로, **아날로그 CCTV(동축케이블+녹화기(DVR))** 는
  - 카메라에는 서버가 없고
  - DVR이 디지털로 바꾸고, RTSP 서버 역할을 합니다.

### 2-3. RTSP를 거의 항상 지원하는 경우

- 상업용/건물용 **정식 IP 카메라 브랜드**:
  - 한화테크윈(삼성), Hikvision, Dahua, Uniview, Axis 등
  - 이런 장비들은 **RTSP 99% 이상 지원**이 기본입니다.
- NVR(네트워크 녹화기) 역시 자체적으로 RTSP를 제공합니다.

### 2-4. RTSP를 지원하지 않는 경우

- 저가형 IoT 홈캠, 일부 클라우드 전용 카메라:
  - 구글 Nest Cam, Ring 등
  - 스마트폰 앱으로만 접속 가능, RTSP URL 미제공
  - 클라우드 서버를 통해서만 영상 전달

---

## 3. Python AI 감지 + 웹 프론트 구조

### 3-1. 추천 전체 구조

```text
[카메라(IP CCTV / 웹캠)]
        ↓ (RTSP / 장치 캡처)
[Python (OpenCV + AI 모델)]
        ↓ (프레임 / 감지 결과)
[웹 서버 (Flask / FastAPI / Node 등)]
        ↓ (HTTP / WebSocket)
[브라우저(웹 프론트)]
```

### 3-2. 역할 분리 아이디어 (두 명이 작업할 때)

- Python 담당
  - 카메라(RTSP/웹캠)에서 프레임 캡처
  - 화재 감지 AI 모델 적용
  - 결과(불탐지 여부, 박스 좌표 등)를 API로 제공
- 웹/프론트 담당
  - Python의 API에서 데이터를 가져옴
  - 실시간 모니터링 화면, 로그 화면, 알람 UI 구현

**핵심 포인트**:  
웹에서 카메라를 직접 제어하는 것이 아니라,  
Python이 “영상/AI 담당 백엔드 서비스” 역할을 하고  
웹은 “표현(뷰)”만 담당하게 만드는 구조가 가장 안정적입니다.

---

## 4. 웹에서 실시간 영상(프레임) 보여주는 방식들

### 4-1. MJPEG 스트리밍

- 서버가 JPEG 이미지를 연속적으로 흘려보내고,
- 브라우저는 `<img>` 태그로 받아서 마치 동영상처럼 보이게 하는 방식.

Python(Flask) 예시 개념:

```python
def generate():
    while True:
        ret, frame = cap.read()
        _, jpeg = cv2.imencode('.jpg', frame)
        yield (b'--frame\r\n'
               b'Content-Type: image/jpeg\r\n\r\n' + jpeg.tobytes() + b'\r\n')
```

HTML:

```html
<img src="/video">
```

- 장점: 구현이 단순함, 대부분 브라우저에서 동작
- 단점: 효율은 떨어지고, 초당 프레임이 아주 높지는 않음

### 4-2. WebSocket으로 프레임 전송

- Python에서 JPEG(또는 PNG) 바이트를 WebSocket으로 계속 보내고
- 브라우저에서 `Blob → ObjectURL → <img>` 로 교체하며 표시

- 장점: 양방향 통신 가능, 제어 메시지(일시정지, 감지 ON/OFF 등)을 같이 주고받기 쉽다.
- 단점: 구현이 MJPEG보다 약간 복잡함.

### 4-3. WebRTC

- 실시간 영상 통화용 기술.
- 매우 낮은 지연, 대규모 서비스에서도 사용하는 방식.
- 하지만 구현이 가장 복잡하고, 시그널링 서버/STUN/TURN 등 많은 구성이 필요.

---

## 5. 비밀번호 해시(Hash) 개념 및 Argon2

### 5-1. 해시란?

- 입력값을 **고정 길이의 이상한 문자열**로 바꾸는 함수.
- 특징:
  - 같은 입력 → 항상 같은 해시
  - 되돌릴 수 없음(일방향)
  - 비밀번호 저장 시 원문을 저장하지 않고 **해시만 저장**함.

### 5-2. 암호화 vs 해시

- 암호화(Encryption)
  - 복호화가 가능
  - 데이터를 다시 원래대로 되돌릴 수 있음
- 해시(Hash)
  - 복호화 불가능
  - 비밀번호처럼 “원래 값을 다시 알 필요가 없는 경우”에 사용

### 5-3. MySQL에 비밀번호 저장 방식

- 비밀번호를 평문으로 저장 ❌
- 해시 함수를 사용해서 암호화된 형태로 저장 ✅
- 추천 알고리즘:
  - bcrypt
  - Argon2id
  - PBKDF2

### 5-4. Argon2 해시 길이

- Argon2 해시 문자열 예시는 보통:

```text
$argon2id$v=19$m=65536,t=3,p=4$<salt-base64>$<hash-base64>
```

- 전체 문자열 길이: 대략 **95 ~ 120자 사이**
- Salt, 파라미터 정보 등이 모두 같이 문자열 안에 들어 있음.
- 그래서 MySQL에서는 `VARCHAR(255)` 정도로 컬럼 생성하는 것이 일반적.

예:

```sql
password VARCHAR(255) NOT NULL COMMENT '비밀번호 해시 값(평문 저장 금지).'
```

---

## 6. N:M(다대다) 관계와 교차 테이블

### 6-1. 이론 vs 실제 구현

- 이론적으로:  
  - 학생 ↔ 강의  
  - 게시글 ↔ 태그  
  - 사용자 ↔ 역할(roles)  
  같은 관계는 **N:M(다대다)** 관계라고 부름.
- 하지만 MySQL 같은 관계형 DB에서는
  - N:M을 *직접* 테이블 구조로 표현할 수 없음.
  - 항상 **교차 테이블(중간 테이블)을 만들어서**
    - `1:N + 1:N` 관계로 분해해야 함.

### 6-2. 예: 게시글과 태그

- Posts (게시글)
- Tags (태그)
- PostTags (교차 테이블)

```text
Posts      1  ─── N  PostTags  N ─── 1  Tags
```

- PostTags:
  - post_id (FK, 복합 PK 일부)
  - tag_id (FK, 복합 PK 일부)

---

## 7. 2NF(제2정규형)와 잘못된 OrderProducts 예시

### 7-1. 2NF 정의 요약

- 이미 1NF를 만족하고,
- 모든 비키 속성(기본키가 아닌 컬럼)이  
  **기본키 전체에 완전 함수 종속**해야 한다.
- 즉, **복합키의 일부에만 종속되는 속성이 있으면 2NF 위반**.

### 7-2. 잘못된 OrderProducts 구조 (정규화 전)

#### 잘못된 OrderProducts 테이블

| 속성명        | 설명                    |
|--------------|-------------------------|
| order_id     | 주문 ID (PK 복합키)     |
| product_id   | 상품 ID (PK 복합키)     |
| user_id      | 주문한 사용자 ID        |
| product_name | 주문된 상품 이름        |
| product_price| 주문된 상품 가격        |
| quantity     | 수량                    |
| created_at   | 주문 생성 시간          |

- 기본키: `(order_id, product_id)` (복합키라고 가정)

#### 문제점 정리

- `user_id`, `created_at` 은 **order_id만 알면 결정**됨
  - 즉, `product_id` 와는 상관 없이 `order_id`에만 종속
  - → **부분 함수 종속 (order_id ⊃ user_id, created_at)**
- `product_name`, `product_price` 는 **product_id만 알면 결정**됨
  - → 역시 복합키의 일부( product_id )에만 종속

- 결론:
  - 비키 속성들이 기본키 `(order_id, product_id)`의 **일부에만 종속**됨
  - → **제2정규형(2NF) 위반**

---

## 8. 올바른 정규화 구조 (Order / Product / OrderProducts 분리)

### 8-1. Order 테이블

| 속성명      | 설명                         |
|------------|------------------------------|
| id (PK)    | 주문 고유 ID                 |
| user_id    | 주문한 사용자 ID             |
| total_price| 총 주문 금액                 |
| created_at | 주문 생성 시간               |

### 8-2. Product 테이블

| 속성명   | 설명               |
|---------|--------------------|
| id (PK) | 상품 고유 ID       |
| name    | 상품 이름          |
| price   | 상품 기본 가격     |

### 8-3. OrderProducts (교차 테이블, N:M 해소)

| 속성명      | 설명                                      |
|------------|-------------------------------------------|
| order_id   | Order.id 참조 (FK, 복합 PK 일부)          |
| product_id | Product.id 참조 (FK, 복합 PK 일부)        |
| quantity   | 주문한 상품 수량                          |
| price      | 주문 당시 해당 상품 가격(스냅샷 저장용)   |

- OrderProducts의 기본키: `(order_id, product_id)`
- 장점:
  - 한 주문(Order)에 여러 상품(Product)을 연결 가능
  - 한 상품이 여러 주문에 포함될 수 있음 → N:M 관계를 올바르게 표현
  - 주문 정보(Order), 상품 정보(Product), 교차 정보(OrderProducts)가 역할별로 분리됨
  - 2NF / 3NF 모두 만족하는 구조에 가깝다.

---

## 9. 정규화 요약 (1NF / 2NF / 3NF)

### 9-1. 제1정규형 (1NF)

- 모든 컬럼이 **원자값(Atomic)** 이어야 함
  - 하나의 셀에 여러 값(리스트, 콤마 구분 ID 등)이 들어가면 안 됨.
- 반복되는 그룹 제거 (예: phone1, phone2, phone3 같은 컬럼들)

### 9-2. 제2정규형 (2NF)

- 1NF를 만족한 상태에서,
- **복합키의 일부에만 종속되는 속성(부분 함수 종속)** 제거
- 주로 N:M 관계를 교차 테이블로 분해하면서 해결

### 9-3. 제3정규형 (3NF)

- 기본키가 아닌 컬럼이 다른 기본키가 아닌 컬럼에 종속되면 안 됨
  - 예: `student(id, dept_id, dept_name)`  
    - 여기서 `dept_name` 은 `dept_id`에 종속
    - `dept_id` 자체는 기본키가 아니므로 → 이행적 종속
- 해결 방안:
  - Dept 테이블을 따로 만들고,  
    Student에는 `dept_id`만 두고,  
    `dept_name` 은 Dept에서 관리.

---

## 10. 전체 개념 요약

1. **VideoCapture / 프레임**
   - `read()` 호출 시마다 현재 시점의 프레임 1장을 읽어온다.
   - FPS에 따라 프레임이 생산되며, 코드는 그중 최신 것을 가져오는 구조.

2. **CCTV / IP 카메라**
   - 대부분 RTSP 기반 스트리밍 서버 내장.
   - Python은 RTSP URL로 접속해서 프레임을 가져온다.

3. **Python + 웹 구조**
   - Python: 캡처 + AI 모델 + 결과 처리(API 제공)
   - 웹: Python API를 이용해 실시간 화면/로그/알람 UI 구성.

4. **해시 / Argon2**
   - 해시는 일방향, 비밀번호 원문은 절대 저장하지 않는다.
   - Argon2 해시 문자열은 보통 95~120자 정도, MySQL은 `VARCHAR(255)` 권장.

5. **N:M 관계**
   - 이론적으로 존재하지만, 실제 DB에서는 반드시 **교차 테이블**로 구현.
   - 예: Posts ↔ Tags, Orders ↔ Products 등.

6. **2NF 정규화**
   - 복합키의 일부에 종속된 컬럼(부분 함수 종속)을 제거해야 함.
   - Order / Product / OrderProducts처럼 엔티티를 분리하여 해결.

---
