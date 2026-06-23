# DSA 기술 개발문서 (Technical Specification)

> **문서 유형**: 개발팀 내부 기술 문서
> **대상**: 백엔드·프론트엔드·DevOps 팀원, AI 코드 어시스턴트
> **버전**: v1.1 (2026-06-23 미팅 반영)
> **범위**: DSA 관리 프로그램(웹) 전용 — 재원생 앱(Flutter)은 별도 문서

---

## 목차

1. [시스템 개요](#1-시스템-개요)
2. [기술 스택](#2-기술-스택)
3. [아키텍처](#3-아키텍처)
4. [DB 설계 원칙](#4-db-설계-원칙)
5. [API 설계 원칙](#5-api-설계-원칙)
6. [외부 연동 명세](#6-외부-연동-명세)
7. [공통 모듈 설계](#7-공통-모듈-설계)
8. [기능별 구현 명세](#8-기능별-구현-명세)
9. [보안·인증·권한](#9-보안인증권한)
10. [배포·인프라](#10-배포인프라)
11. [개발 착수 선결 조건](#11-개발-착수-선결-조건)
12. [Phase별 개발 범위](#12-phase별-개발-범위)
13. [리스크 및 주의사항](#13-리스크-및-주의사항)

---

## 1. 시스템 개요

### 1.1 프로젝트 설명

D.Lab (대성학력개발연구소) 전용 학원 관리 프로그램 신규 개발.
현재 대성전산이 개발·운영 중인 DSA를 자체 시스템으로 대체한다.

### 1.2 운영 주체 및 관계사

| 주체 | 역할 | 비고 |
|------|------|------|
| 대성학력개발연구소 (DSD) | D.Lab 사이트 운영, 발주처 | |
| 대성전산 | 현재 DSA·디멤버·수납 프로그램 개발·운영 | DB 포함 전체 소유 |
| 자이엘코리아 | Zyxel 방화벽 장비 관리 | 김상원 팀장 / 010-6412-8131 |
| 개발팀 | 신규 DSA 개발 | |

### 1.3 개발 범위 요약

- **DSA 관리 프로그램 (웹)** — 이 문서의 대상
- **재원생 전용 앱 (Flutter)** — 별도 문서 (`APP_기술_개발문서.md`)
- **공통 백엔드 API (Spring Boot)** — DSA와 앱이 공유하는 서버

---

## 2. 기술 스택

### 2.1 프론트엔드 (DSA 웹)

```
Framework  : React 18 (또는 Vue 3 — 최종 확정 필요)
Language   : TypeScript
UI Library : Ant Design 또는 Shadcn/ui
State      : Zustand (전역) + React Query (서버 상태)
Build      : Vite
HTTP       : Axios (인터셉터로 JWT 자동 첨부)
Excel      : SheetJS (xlsx)
Chart      : Recharts
```

### 2.2 백엔드

```
Framework  : Spring Boot 3.x
Language   : Java 21
ORM        : Spring Data JPA + QueryDSL
DB         : Supabase (PostgreSQL 15)
Auth       : Supabase Auth + Spring Security (JWT 검증)
Excel      : Apache POI
HTTP Client: WebClient (Reactive) + Resilience4j (Circuit Breaker)
Messaging  : (필요 시 Redis Pub/Sub 또는 SSE for 실시간)
```

### 2.3 인프라

```
Hosting    : AWS (EC2 또는 ECS) — 확정 필요
DB         : Supabase Cloud (PostgreSQL)
Storage    : Supabase Storage (PDF, 이미지)
CI/CD      : GitHub Actions
Container  : Docker + Docker Compose
Reverse Proxy: Nginx
```

### 2.4 외부 연동

| 시스템 | 연동 방식 | 용도 |
|--------|----------|------|
| 대성전산 API | REST API | 디멤버·수납 프로그램 데이터 연동 |
| 키오스크 | Webhook 수신 | 출결 데이터 실시간 수신 |
| Zyxel Nebula | REST API | 방화벽 해제 승인 |
| 더프리미엄모의고사 | REST API | 전화번호 기반 성적 조회 |
| 카카오 알림톡 | KakaoTalk API | 학부모·학생 알림 발송 |
| PG사 (등록비) | REST API + Webhook | 카드·가상계좌 결제 (학원 명의) |
| PG사 (급식) | REST API + Webhook | 카드·가상계좌 결제 (벤더 명의) |

---

## 3. 아키텍처

### 3.1 전체 시스템 구조

```
┌─────────────────────────────────────────────────────┐
│                   외부 클라이언트                        │
│  [DSA 웹 (React)]     [재원생 앱 (Flutter) — 별도]       │
└──────────────┬──────────────────────┬───────────────┘
               │ HTTPS / REST API     │
               ▼                      ▼
┌─────────────────────────────────────────────────────┐
│            Spring Boot API Server                    │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐ │
│  │Auth/RBAC │ │Business  │ │External Gateway Layer │ │
│  │(JWT+Role)│ │Logic     │ │(Circuit Breaker)      │ │
│  └──────────┘ └──────────┘ └──────────────────────┘ │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
           ▼                          ▼
┌─────────────────┐        ┌─────────────────────────┐
│  Supabase DB    │        │    외부 시스템 연동          │
│  (PostgreSQL)   │        │  - 대성전산 API (디멤버·수납)│
│  Supabase Auth  │        │  - 키오스크 Webhook        │
│  Supabase       │        │  - Zyxel Nebula API       │
│  Storage        │        │  - 더프리미엄 API (성적)    │
└─────────────────┘        │  - 카카오 알림톡            │
                           │  - PG사 (등록비·급식 분리)  │
                           └─────────────────────────┘
```

### 3.2 DSA 프론트엔드 구조 (React)

```
src/
├── api/            # Axios 인스턴스, API 호출 함수
├── components/
│   ├── common/     # 공통 컴포넌트 (SearchForm, DataTable, ExcelButton 등)
│   └── features/   # 기능별 컴포넌트
├── pages/          # 라우트별 페이지
├── store/          # Zustand 전역 상태
├── hooks/          # React Query 커스텀 훅
├── types/          # TypeScript 타입 정의
└── utils/          # 공통 유틸 (날짜, 엑셀, 포맷팅)
```

### 3.3 백엔드 패키지 구조 (Spring Boot)

```
com.dlab.dsa/
├── auth/           # JWT 발급·검증, RBAC
├── common/
│   ├── excel/      # Apache POI 공통 Import/Export
│   ├── notification/ # 알림톡·SMS 추상화
│   ├── search/     # QueryDSL 동적 검색 빌더
│   └── webhook/    # Webhook 수신·서명 검증 공통
├── external/
│   ├── daesungdb/  # 대성전산 API 클라이언트
│   ├── kiosk/      # 키오스크 Webhook 핸들러
│   ├── nebula/     # Zyxel Nebula API 클라이언트
│   ├── pg/         # PG사 어댑터 (Tuition / Lunch 분리)
│   ├── premium/    # 더프리미엄 API 클라이언트
│   └── kakao/      # 카카오 알림톡 클라이언트
├── domain/
│   ├── student/    # 학생 관리
│   ├── attendance/ # 출결 관리
│   ├── score/      # 성적 관리
│   ├── meal/       # 급식 관리
│   ├── payment/    # 결제·수납
│   ├── survey/     # 설문
│   ├── notice/     # 대기자·공지
│   └── admin/      # 관리자·기초설정
└── config/         # Spring Security, JPA, WebClient 설정
```

---

## 4. DB 설계 원칙

### 4.1 공통 컬럼 규칙

모든 엔티티 테이블에 아래 컬럼을 표준으로 포함한다.

```sql
id            UUID PRIMARY KEY DEFAULT gen_random_uuid()
year          INTEGER NOT NULL          -- 학년도 (매년 초기화 기준)
branch_id     UUID NOT NULL             -- 지점 ID
created_at    TIMESTAMPTZ DEFAULT now()
updated_at    TIMESTAMPTZ DEFAULT now()
created_by    UUID REFERENCES users(id)
is_deleted    BOOLEAN DEFAULT FALSE     -- Soft delete
```

### 4.2 연도/지점 버전 관리

"전년도 복사" 기능이 학과·반·커리큘럼·상벌점·교습비·시간표 등 모든 기초 데이터에 필요하다.

```sql
-- 복사 출처 추적 (전년도 → 신규 연도 복사 시)
copied_from_id  UUID REFERENCES self(id) NULL
```

**복사 순서 (의존관계 순):**
```
department (학과) → course_type (전형) → class_group (반)
  → curriculum → penalty_item (상벌점 항목) → tuition (교습비)
```

단순 `INSERT SELECT` 금지 — `YearlySnapshotService`로 그래프 순회 후 새 PK 매핑하여 복사.

### 4.3 학번 초기화 규칙

학번은 매년 초기화. `year` + `sequence`로 생성.

```sql
students (
  id           UUID PK,
  year         INTEGER,          -- 2026, 2027 ...
  student_no   VARCHAR(10),      -- 학번 (해당 연도 내 유니크)
  UNIQUE(year, student_no)
)
```

### 4.4 주요 테이블 목록

```sql
-- 기초 데이터
departments         -- 학과 (year, branch_id)
course_types        -- 전형
class_groups        -- 반 (담임, 강의실, 정원)
curriculums         -- 커리큘럼
penalty_items       -- 상벌점 항목

-- 학생
students            -- 재원생
student_waitlist    -- 대기자 (D.Lab 사이트 연동)
admission_consults  -- 입학예약 상담 기록
class_assignments   -- 반 배정 이력

-- 출결
attendance_logs     -- 출결 기록 (키오스크 Webhook 수신)
absence_requests    -- 사유 신청 (앱 → DSA 동기화)
study_sessions      -- 자습 시간 기록

-- 결제·수납
payment_transactions -- 결제 내역 (TUITION / MEAL 구분)
meal_orders         -- 급식 신청
meal_order_items    -- 급식 신청 상세 (일별)

-- 성적·설문
score_records       -- 성적 기록 (더프리미엄 API 동기화)
surveys             -- 설문 정의
survey_responses    -- 설문 응답

-- 알림·메시지
notification_logs   -- 발송 이력
notification_templates -- 알림톡 템플릿

-- 관리자
users               -- DSA 사용자 (직원·강사)
roles               -- 권한 역할
branch_configs      -- 지점별 외부 연동 설정 (API 키 등)
```

---

## 5. API 설계 원칙

### 5.1 URL 규칙

```
GET    /api/v1/{resource}           -- 목록 조회
GET    /api/v1/{resource}/{id}      -- 단건 조회
POST   /api/v1/{resource}           -- 생성
PUT    /api/v1/{resource}/{id}      -- 전체 수정
PATCH  /api/v1/{resource}/{id}      -- 부분 수정
DELETE /api/v1/{resource}/{id}      -- 삭제 (soft delete)

-- 연도 복사
POST   /api/v1/{resource}/copy-from-year

-- 엑셀
GET    /api/v1/{resource}/export    -- 엑셀 다운로드
POST   /api/v1/{resource}/import    -- 엑셀 업로드
POST   /api/v1/{resource}/import/preview -- 업로드 전 미리보기
```

### 5.2 공통 응답 형식

```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "page": 1,
    "size": 20,
    "total": 150
  },
  "error": null
}
```

에러 응답:
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "STUDENT_NOT_FOUND",
    "message": "학생을 찾을 수 없습니다.",
    "details": []
  }
}
```

### 5.3 공통 검색 파라미터

```
GET /api/v1/students?year=2026&branchId=xxx&keyword=홍길동
  &departmentId=xxx&classGroupId=xxx&status=ENROLLED
  &page=0&size=20&sort=name,asc
```

모든 목록 API는 `year`, `branchId` 필수. 나머지는 선택.

---

## 6. 외부 연동 명세

### 6.1 대성전산 API 연동

- **용도**: 디멤버 사이트 데이터 조회, 수납 프로그램 데이터 연동
- **방식**: REST API (스펙 협의 필요 — Phase 0 선결 조건)
- **주의**: 대성전산 DB 직접 접근 금지. 반드시 API 레이어 경유
- **Circuit Breaker**: 대성전산 API 장애 시 DSA 전체 장애 방지 필수

```java
@Component
public class DaesungApiClient {
    // Resilience4j CircuitBreaker 적용
    @CircuitBreaker(name = "daesungApi", fallbackMethod = "fallback")
    public SunapData fetchSunapData(String studentId) { ... }
}
```

**연동 데이터 흐름:**
```
급식 결제 완료 (PG Webhook)
  → Spring Boot 수신
  → DSA DB 저장
  → 대성전산 수납 API 호출 (자동 반영)
```

### 6.2 키오스크 Webhook

- **방향**: 키오스크 → Spring Boot (수신)
- **엔드포인트**: `POST /webhooks/kiosk/attendance`
- **인증**: HMAC-SHA256 서명 검증 (시크릿 키는 `branch_configs` 테이블에서 지점별 관리)
- **처리 흐름**:

```
키오스크 태그
  → POST /webhooks/kiosk/attendance
  → 서명 검증
  → attendance_logs 저장
  → 출결 상태 계산 (등원/지각/결석)
  → 카카오 알림톡 발송 (학부모)
  → 앱 실시간 반영 (SSE 또는 FCM — 앱 문서 참고)
```

**키오스크 Webhook 페이로드 (추정 — 업체 확인 필요):**
```json
{
  "branchId": "branch-uuid",
  "studentNo": "2026001",
  "taggedAt": "2026-06-23T08:30:00+09:00",
  "deviceType": "ENTRANCE",
  "nfcCardId": "CARD-xxx"
}
```

### 6.3 Zyxel Nebula API (방화벽)

- **용도**: 학생 방화벽 해제 요청 → DSA 관리자 승인 → Nebula API로 해제 명령
- **담당**: 자이엘코리아 (010-6412-8131) — API 접근 권한 협의 필요
- **장비**: USG Flex 700H + WBE510D + WBE630S + GS1920-24HP

```
앱에서 방화벽 해제 신청
  → DSA 관리자 승인
  → POST Nebula API /firewall/whitelist
  → 학생 MAC 주소 일시 허용
  → 앱 해제 완료 알림
```

### 6.4 더프리미엄모의고사 API (성적)

- **용도**: 학생 전화번호 기반 모의고사 성적 조회
- **방식**: `GET /scores?phone=010xxxx`
- **스펙**: Phase 0에서 확인 필요

```java
@Component
public class PremiumExamApiClient {
    public ScoreResult fetchByPhone(String phone) {
        // 전화번호로 성적 조회
        // 결과를 score_records 테이블에 저장·동기화
    }
}
```

### 6.5 PG사 결제 연동

**두 결제 흐름 분리 (가맹점 명의가 다름)**

```java
interface PaymentGatewayClient {
    VirtualAccountResult issueVirtualAccount(PaymentRequest req);
    CardPaymentResult requestCardPayment(PaymentRequest req);
    void verifyWebhookSignature(String payload, String signature);
    RefundResult requestRefund(String transactionId, int amount); // 등록비만
}

// 등록비 (학원 명의 가맹점)
@Component
class TuitionPgClient implements PaymentGatewayClient { ... }

// 급식 (급식 업체 벤더 명의 가맹점)
@Component
class LunchVendorPgClient implements PaymentGatewayClient {
    // 환불은 벤더 직접 처리 → DSA는 취소 요청·기록만
    // requestRefund() 미구현 (UnsupportedOperationException)
}
```

**Webhook 엔드포인트 분리:**
```
POST /webhooks/pg/tuition    -- 등록비 결제 콜백
POST /webhooks/pg/lunch      -- 급식 결제 콜백
```

**급식 결제 흐름:**
```
결제 요청 → 급식 업체 PG (벤더 명의)
  → 결제 완료 Webhook → /webhooks/pg/lunch
  → payment_transactions 저장 (payment_type = MEAL)
  → 대성전산 수납 API 자동 반영
```

**가상계좌 상태 관리:**
```
PENDING(발급대기) → ISSUED(발급완료) → PAID(입금확인) → EXPIRED(만료)
```
입금 대기 중 만료 처리 스케줄러 필요 (`@Scheduled`).

**DB 스키마:**
```sql
payment_transactions (
  id                UUID PK,
  student_id        UUID REFERENCES students(id),
  payment_type      VARCHAR(10) NOT NULL,  -- TUITION | MEAL
  pg_provider       VARCHAR(30),           -- PG사명
  pg_transaction_id VARCHAR(100),
  amount            INTEGER NOT NULL,
  status            VARCHAR(20) NOT NULL,  -- PENDING|ISSUED|PAID|CANCELLED|EXPIRED
  virtual_account_no VARCHAR(30),          -- 가상계좌번호
  virtual_account_bank VARCHAR(10),
  expires_at        TIMESTAMPTZ,
  paid_at           TIMESTAMPTZ,
  settled_at        TIMESTAMPTZ,
  year              INTEGER,
  branch_id         UUID,
  created_at        TIMESTAMPTZ DEFAULT now()
)
```

### 6.6 카카오 알림톡

- **채널 개설 및 템플릿 심사 선행 필요** (Phase 0 착수)
- 심사 소요: 통상 영업일 1~3일 / 템플릿이 많으므로 일괄 신청

**DSA에서 발송하는 알림톡 템플릿 목록:**

| # | 템플릿명 | 발송 트리거 |
|---|---------|-----------|
| 1 | 출결_등원 | 키오스크 등원 태그 |
| 2 | 출결_지각 | 기준 시간 초과 등원 |
| 3 | 출결_결석 | 미등원 처리 |
| 4 | 상벌점_부여 | 상벌점 등록 |
| 5 | 대기자_순번처리 | 대기순번 → 등록 가능 |
| 6 | 수납_미납안내 | 미납자 명단 조회 시 |
| 7 | 급식_결제완료 | 급식 PG 결제 완료 |
| 8 | 성적_업로드 | PDF 성적표 업로드 시 |

```java
@Component
public class KakaoNotificationClient {
    public void send(String phone, String templateCode, Map<String, String> variables) {
        // 실패 시 SMS fallback 처리
    }
}

// NotificationService — 채널 추상화
@Service
public class NotificationService {
    public void sendAttendanceAlert(Student student, AttendanceType type) {
        // type에 따라 템플릿 선택 → KakaoNotificationClient.send()
    }
}
```

---

## 7. 공통 모듈 설계

### 7.1 동적 검색 빌더

거의 모든 조회 화면이 동일한 필터(학과·전형·계열·반·성별·상태 등)를 사용하므로 공통 빌더로 통합한다.

```java
// QueryDSL Specification 패턴
@Component
public class StudentSearchBuilder {
    public BooleanExpression build(StudentSearchCondition cond) {
        return Expressions.allOf(
            yearEq(cond.getYear()),
            branchEq(cond.getBranchId()),
            departmentEq(cond.getDepartmentId()),
            classGroupEq(cond.getClassGroupId()),
            statusEq(cond.getStatus()),
            keywordLike(cond.getKeyword())  // 이름·학번·전화번호 OR 검색
        );
    }
}
```

### 7.2 엑셀 Import/Export 프레임워크

```java
// Export
public interface ExcelExporter<T> {
    List<String> getHeaders();
    List<Object> toRow(T entity);
    String getSheetName();
}

// Import
public interface ExcelImporter<T> {
    T fromRow(Row row);
    List<ValidationError> validate(T entity); // 업로드 전 검증
}

// 공통 Import 응답
public record ImportPreviewResult(
    int totalRows,
    int validRows,
    int errorRows,
    List<ValidationError> errors,   // 행·열·메시지
    List<Object> preview            // 유효 데이터 미리보기
) {}
```

**사용 예:**
```java
@PostMapping("/students/import/preview")
public ImportPreviewResult preview(@RequestParam MultipartFile file) {
    return excelService.preview(file, new StudentImporter());
}
```

### 7.3 연도 초기화 서비스 (YearlySnapshotService)

```java
@Service
public class YearlySnapshotService {

    /**
     * 전년도 기초 데이터 → 신규 연도 전체 복사
     * 의존 그래프 순서: department → course_type → class_group
     *                  → curriculum → penalty_item → tuition
     */
    @Transactional
    public void copyFromYear(int sourceYear, int targetYear, UUID branchId) {
        // 1. 학과 복사
        List<Department> newDepts = copyDepartments(sourceYear, targetYear, branchId);
        // 2. 전형 복사 (학과 FK 새 PK로 매핑)
        // 3. 반 복사
        // 4. 이하 순서대로 ...
        // 각 복사 레코드에 copied_from_id 기록
    }
}
```

### 7.4 Webhook 서명 검증 공통 레이어

```java
@Component
public class WebhookSignatureVerifier {
    public void verify(String payload, String receivedSignature, String secret) {
        String expected = HmacUtils.hmacSha256Hex(secret, payload);
        if (!MessageDigest.isEqual(expected.getBytes(), receivedSignature.getBytes())) {
            throw new WebhookSignatureException("서명 불일치");
        }
    }
}

// 키오스크, PG 콜백 모두 이 검증기 사용
@PostMapping("/webhooks/kiosk/attendance")
public void handleKioskWebhook(
    @RequestBody String payload,
    @RequestHeader("X-Kiosk-Signature") String sig
) {
    signatureVerifier.verify(payload, sig, kioskSecret);
    // 처리 로직
}
```

---

## 8. 기능별 구현 명세

### 8.1 학생 관리

**주요 API:**
```
GET    /api/v1/students                     -- 목록 (동적 검색)
GET    /api/v1/students/{id}               -- 상세
POST   /api/v1/students                    -- 신규 등록
PUT    /api/v1/students/{id}               -- 수정
GET    /api/v1/students/export             -- 엑셀 다운로드
POST   /api/v1/students/import/preview     -- 엑셀 업로드 미리보기
POST   /api/v1/students/import             -- 엑셀 업로드 확정
POST   /api/v1/students/{id}/graduate      -- 원생 전환 (대기자 → 원생)
```

**학번 자동 생성:**
```java
// 연도별 시퀀스 채번
String studentNo = year + String.format("%04d", nextSeq(year, branchId));
// e.g. 2026-0001
```

### 8.2 대기자 관리 (신규)

D.Lab 사이트에서 입학 예약한 학생이 대기자로 등록됨.

```
GET    /api/v1/waitlist                    -- 대기자 목록 (순번 포함)
POST   /api/v1/waitlist/{id}/notify        -- 순번 처리 알림 발송
POST   /api/v1/waitlist/{id}/message       -- 관리자 직접 메시지 발송
POST   /api/v1/waitlist/{id}/convert       -- 원생 전환
DELETE /api/v1/waitlist/{id}               -- 대기 취소
```

**알림 발송 시 포함 내용:**
- 처리 완료 메시지
- 등록 가능 기한 (날짜·시간)
- 확인 응답 링크 (올 수 있는지 여부 답변)

```java
// 대기자 → 원생 전환 로직
@Transactional
public Student convertToStudent(UUID waitlistId, ConvertRequest req) {
    Waitlist waitlist = waitlistRepo.findById(waitlistId);
    Student student = Student.from(waitlist, req.getYear(), req.getClassGroupId());
    student.setStudentNo(studentNoGenerator.generate(req.getYear(), req.getBranchId()));
    studentRepo.save(student);
    waitlist.markConverted();
    // 앱 초대 알림톡 발송
    notificationService.sendAdmissionInvite(student);
    return student;
}
```

### 8.3 출결 관리

**키오스크 Webhook 처리:**
```java
@PostMapping("/webhooks/kiosk/attendance")
@Transactional
public void handleAttendance(@RequestBody KioskPayload payload) {
    // 1. 서명 검증
    signatureVerifier.verify(payload);
    // 2. 학생 조회 (NFC 카드 ID or 학번)
    Student student = studentRepo.findByNfcCard(payload.getNfcCardId());
    // 3. 출결 상태 계산
    AttendanceType type = attendanceCalculator.calculate(student, payload.getTaggedAt());
    // 4. DB 저장
    AttendanceLog log = attendanceLogRepo.save(AttendanceLog.of(student, type, payload));
    // 5. 알림톡 발송 (비동기)
    notificationService.sendAttendanceAlert(student, type);
}
```

**출결 상태 계산 기준:**
```java
public enum AttendanceType {
    ON_TIME,    // 기준 시간 이전 등원
    LATE,       // 기준 시간 초과 등원
    ABSENT,     // 미등원 (스케줄러로 배치 처리)
    OUT,        // 외출
    EXCUSED     // 사유 승인된 경우
}
```

### 8.4 급식 관리

**신청 로직:**
```java
@PostMapping("/api/v1/meals/orders")
@Transactional
public MealOrder createOrder(MealOrderRequest req) {
    // 1. 신청 가능 기간 검증 (월말 신청 기간)
    mealPolicy.validateOrderPeriod(req.getTargetMonth());
    // 2. 주말 필터링 (주말 제외)
    List<LocalDate> validDates = mealPolicy.filterWeekdays(req.getRequestedDates());
    // 3. 결제 요청 (LunchVendorPgClient)
    PaymentResult payment = lunchPgClient.requestCardPayment(toPaymentRequest(req));
    // 4. meal_orders 저장
    return mealOrderRepo.save(MealOrder.of(student, validDates, payment));
}
```

**취소 로직:**
```java
@DeleteMapping("/api/v1/meals/orders/{orderId}/items/{itemId}")
public void cancelItem(@PathVariable UUID itemId,
                       @RequestAttribute("currentUser") User user) {
    MealOrderItem item = mealItemRepo.findById(itemId);
    // 3일 전 취소 정책 (관리자는 예외)
    if (!user.isAdmin()) {
        mealPolicy.validateCancelDeadline(item.getMealDate()); // 3일 전 검증
    }
    // 업체 API 취소 요청 기록
    item.markCancelRequested(user);
    // 수납 프로그램 반영 (대성전산 API)
    daesungApiClient.syncMealCancel(item);
}
```

### 8.5 성적 관리

```java
// 더프리미엄 API 성적 동기화
@Service
public class ScoreService {
    public ScoreResult fetchAndSync(UUID studentId) {
        Student student = studentRepo.findById(studentId);
        // 전화번호로 성적 조회
        ExamScore score = premiumExamClient.fetchByPhone(student.getPhone());
        // score_records 저장
        ScoreRecord record = scoreRecordRepo.save(ScoreRecord.of(student, score));
        return ScoreResult.from(record);
    }

    // PDF 업로드 → Supabase Storage → D.Lab 사이트 연동
    public String uploadScorePdf(UUID studentId, MultipartFile pdf) {
        String url = supabaseStorage.upload("scores/" + studentId, pdf);
        // D.Lab 사이트 연동 API 호출 (대성전산 API)
        daesungApiClient.notifyScorePdfUploaded(studentId, url);
        return url;
    }
}
```

### 8.6 설문 관리

```
POST   /api/v1/surveys                     -- 설문 생성
GET    /api/v1/surveys/{id}               -- 설문 상세
POST   /api/v1/surveys/{id}/publish       -- 설문 배포
GET    /api/v1/surveys/{id}/responses     -- 응답 목록
GET    /api/v1/surveys/{id}/results       -- 집계 결과
```

```sql
surveys (
  id, title, type,          -- type: MOCK_EXAM_SCORE | GENERAL
  target_year, branch_id,
  starts_at, ends_at,
  is_published
)
survey_questions (
  id, survey_id, question_text, question_type, -- TEXT | NUMBER | SELECT
  options JSONB, is_required, order_seq
)
survey_responses (
  id, survey_id, student_id, submitted_at
)
survey_answers (
  id, response_id, question_id, answer_text, answer_number
)
```

---

## 9. 보안·인증·권한

### 9.1 인증 흐름

```
1. DSA 로그인 (직원 ID + PW)
   → Supabase Auth 인증
   → JWT 발급 (Access Token 1시간 / Refresh Token 7일)
2. 이후 모든 API 요청 Header: Authorization: Bearer {accessToken}
3. Spring Security → JWT 검증 → SecurityContext 설정
```

### 9.2 권한 체계 (RBAC)

```java
public enum Role {
    SUPER_ADMIN,    // 전체 지점 관리
    BRANCH_ADMIN,   // 지점 관리자
    TEACHER,        // 담임 (담당 반만 접근)
    STAFF,          // 일반 직원
    READONLY        // 조회 전용
}
```

**메뉴별 권한 매트릭스 (주요 항목):**

| 메뉴 | SUPER_ADMIN | BRANCH_ADMIN | TEACHER | STAFF |
|------|:-----------:|:------------:|:-------:|:-----:|
| 학생 관리 전체 | ✅ | ✅ | 담당 반만 | ✅ |
| 수납현황 조회 | ✅ | ✅ | ❌ | 조회만 |
| 관리자 기초설정 | ✅ | ✅ | ❌ | ❌ |
| 사용자 관리 | ✅ | 지점 내만 | ❌ | ❌ |
| 성적 관리 | ✅ | ✅ | 담당 반만 | ✅ |

> ⚠️ RBAC 골격은 Phase 0에서 반드시 먼저 구현. 개인정보 노출 메뉴(수강생 대장·강사 주소록 등)가 Phase 1부터 개발됨.

### 9.3 개인정보 보호

- 전화번호·주소·생년월일 포함 응답: `BRANCH_ADMIN` 이상만 접근
- 엑셀 다운로드: 전화번호 마스킹 옵션 기본 ON (`010-****-1234`)
- API 응답에서 민감 필드는 권한 없으면 마스킹

---

## 10. 배포·인프라

### 10.1 환경 구성

```
개발 (dev)   → 로컬 Docker Compose + Supabase local
스테이징 (stg) → AWS + Supabase Cloud (테스트 데이터)
운영 (prod)  → AWS + Supabase Cloud
```

### 10.2 환경변수 관리

```properties
# application-prod.yml (Secret Manager에서 주입)
spring:
  datasource:
    url: ${SUPABASE_DB_URL}
    username: ${SUPABASE_DB_USER}
    password: ${SUPABASE_DB_PASSWORD}

external:
  daesung:
    api-url: ${DAESUNG_API_URL}
    api-key: ${DAESUNG_API_KEY}
  kiosk:
    webhook-secret: ${KIOSK_WEBHOOK_SECRET}
  pg:
    tuition:
      api-key: ${PG_TUITION_API_KEY}
      webhook-secret: ${PG_TUITION_WEBHOOK_SECRET}
    lunch:
      api-key: ${PG_LUNCH_API_KEY}
      webhook-secret: ${PG_LUNCH_WEBHOOK_SECRET}
  kakao:
    sender-key: ${KAKAO_SENDER_KEY}
  nebula:
    api-url: ${NEBULA_API_URL}
    api-key: ${NEBULA_API_KEY}
  premium-exam:
    api-url: ${PREMIUM_EXAM_API_URL}
    api-key: ${PREMIUM_EXAM_API_KEY}
```

### 10.3 지점별 설정 테이블

지점마다 방화벽·PG 가맹점 코드·키오스크 시크릿이 다를 수 있으므로 DB에서 관리.

```sql
branch_configs (
  branch_id        UUID PK,
  kiosk_secret     TEXT,          -- 암호화 저장
  pg_merchant_code VARCHAR(50),
  nebula_device_id VARCHAR(50),
  config_json      JSONB          -- 추가 지점별 설정
)
```

---

## 11. 개발 착수 선결 조건

Phase 0 시작 전 반드시 확인 필요. 미확인 상태에서 DB 스키마 확정 시 재작업 위험.

| # | 확인 대상 | 담당 | 상태 |
|---|---------|------|------|
| 1 | 대성전산 API 연동 스펙 (디멤버·수납 프로그램) | 개발팀 | 🔴 미확인 |
| 2 | 키오스크 업체 Webhook 스펙·인증 방식 | 개발팀 | 🔴 미확인 |
| 3 | Zyxel Nebula API 접근 권한 | 개발팀 / 자이엘코리아 | 🔴 미확인 |
| 4 | 더프리미엄모의고사 API 스펙 | 개발팀 | 🔴 미확인 |
| 5 | 급식 PG사 선정 및 API 계약 | 기획팀 | 🔴 미확인 |
| 6 | 대성전산 DSA 계약 종료 시점 | 행정팀 | 🔴 미확인 |
| 7 | D.Lab 디자인 가이드 수령 | 기획팀 | 🔴 수령 예정 |
| 8 | 교무업무 명단 조회 구분 항목 정리 | 운영팀 | 🔴 확인 필요 (!!!)|
| 9 | 카카오 알림톡 채널 개설 | 운영팀 | 🔴 미완료 |
| 10 | 구글 시트 2개 데이터 구조·마이그레이션 계획 | 개발팀·운영팀 | 🔴 미확인 |

**확정 완료:**
- ✅ 디멤버·수납 프로그램 개발 주체: 대성전산 (API 연동 방식 A 선택)
- ✅ 급식 결제 가맹점: 급식 업체(벤더) 명의 / PG 분리 설계
- ✅ 방화벽 장비: Zyxel USG Flex 700H (현재 운영 중)

---

## 12. Phase별 개발 범위

### Phase 0 — 사전 준비 (선결 조건 확인 병행)

**백엔드:**
- [ ] Supabase 프로젝트 생성·DB 스키마 초안
- [ ] Spring Boot 프로젝트 초기 세팅 (Security, JPA, QueryDSL)
- [ ] RBAC 공통 레이어 구현
- [ ] 공통 모듈: 검색 빌더, 엑셀 Import/Export, 알림 서비스, Webhook 서명 검증
- [ ] 외부 연동 게이트웨이 (Circuit Breaker 설정)
- [ ] PG 어댑터 인터페이스 설계
- [ ] 카카오 알림톡 템플릿 일괄 등록 신청

**프론트엔드:**
- [ ] React 프로젝트 세팅 (TypeScript, Vite, Axios, React Query)
- [ ] 공통 컴포넌트: 레이아웃, SearchForm, DataTable, ExcelButton
- [ ] 로그인·권한 분기 처리
- [ ] D.Lab 디자인 가이드 적용 (수령 후)

### Phase 1 — MVP

| 기능 | 백엔드 | 프론트엔드 |
|------|--------|-----------|
| 학생 관리 (검색·등록·반배정) | students, class_assignments API | 학원생 목록·상세·등록 화면 |
| 대기자 관리 | waitlist API, 알림 발송 | 대기자 목록·전환 화면 |
| 입학예약 상담 기록 | admission_consults API | 상담 기록 입력·조회 화면 |
| 출결 관리 | Webhook 수신, attendance API | 출결현황 화면 |
| 상벌점 관리 | penalty API, 전년도 복사 | 상벌점 항목·내역 화면 |
| 문자 발송 | notification API | 개별·단체 발송 화면 |
| 관리자 기초설정 | 기초 데이터 API, YearlySnapshotService | 기초설정 화면 |

### Phase 2 — 핵심 기능

| 기능 | 백엔드 | 프론트엔드 |
|------|--------|-----------|
| 급식 관리 | meal API, LunchVendorPgClient, 수납 연동 | 급식 신청·결제·현황 화면 |
| 수납현황 | 대성전산 API 연동, payment API | 매출장·미납자 화면 |
| 교무업무 (명단 출력) | 명단 조회 API, 엑셀 Export | 수강생 대장·장학생 명단 화면 |
| 특강 관리 | 특강 API (설명회 포함) | 특강 신청·출석 화면 |
| 관리자 수납관리 | 대성전산 수납 API 연동 | 수납 설정 화면 |

### Phase 3 — 교육·성적

| 기능 | 백엔드 | 프론트엔드 |
|------|--------|-----------|
| 성적 관리 | 더프리미엄 API, score API, PDF 업로드 | 성적 조회·PDF 업로드 화면 |
| 상담 리포트 | 리포트 생성·발송 API | 상담 리포트 화면 |
| 설문 관리 | survey API | 설문 생성·배포·결과 화면 |
| D.Lab 전체 | D.Lab 메뉴 API 통합 | D.Lab 화면 |

### Phase 4 — 부가 서비스

- 통계 대시보드 (지점별 비교, 등록 추이)
- D.Lab 스토어 연동 (2027년 12월 예정)
- 관리자 대시보드 고도화

---

## 13. 리스크 및 주의사항

| # | 리스크 | 대응 방안 |
|---|--------|----------|
| 1 | 대성전산 API 스펙 미공개 / 연동 거부 | 계약 조건 사전 합의, 최악의 경우 DB 직접 읽기 협의 |
| 2 | 디멤버 연동 범위 불명확 | Phase 0에서 대성전산과 상세 스펙 미팅 선행 |
| 3 | 카카오 알림톡 심사 지연 | Phase 0부터 템플릿 전량 등록 신청, 심사 지연 시 SMS fallback |
| 4 | 수납 프로그램과 급식 결제 데이터 동기화 오류 | Webhook 재시도 로직 + 데이터 정합성 주기적 검증 배치 |
| 5 | RBAC 미구현 상태에서 개인정보 메뉴 오픈 | Phase 0 RBAC 완성 필수, Phase 1 배포 전 보안 검토 |
| 6 | 대성전산 DSA 계약 종료 전 병행 운영 | 양방향 동기화 어댑터 설계 (단방향 마이그레이션 가정 금지) |
| 7 | 2개월 납기 — 인력 집중 투입 필요 | Phase 0·1 우선 완성 후 점진적 배포 |

---

*본 문서는 개발팀 내부 전용입니다. AI 코드 어시스턴트에게 컨텍스트로 제공 시 이 문서 전체를 참조하세요.*
