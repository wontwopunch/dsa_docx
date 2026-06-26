# CI (연계정보) 관리 명세

> **대상**: DSA 백엔드 개발팀
> **범위**: 재원생 앱 회원가입 시 본인인증 CI 처리

---

## 1. CI란

본인인증 완료 시 발급되는 고유 식별값 (88자 문자열).
동일인이면 어떤 기관에서 인증해도 항상 같은 값이 나온다.

---

## 2. 사용자 유형

| 유형 | 설명 |
|------|------|
| STUDENT | 재원생 본인 명의로 인증 |
| GUARDIAN | 재원생 부모 본인 명의로 인증 |

---

## 3. DB 저장 구조

```sql
users (
  id            UUID PK,
  ci_hash       VARCHAR(64),   -- SHA-256 해시 (중복 가입 체크용)
  ci_encrypted  TEXT,          -- AES-256 암호화 (원본 필요 시 복호화)
  user_type     VARCHAR(10)    -- STUDENT | GUARDIAN
)
```

**DB에 저장되는 것:**
- `ci_hash` — SHA-256 해시값 (중복 체크용, 복원 불가)
- `ci_encrypted` — AES-256으로 암호화된 값

**DB에 절대 없는 것:**
- CI 원본값
- 암호화 키

---

## 4. 암호화 키 관리

암호화 키는 **AWS Secrets Manager**에 보관한다.
코드 하드코딩, 환경변수 평문 저장, DB 저장 모두 금지.

```
AWS Secrets Manager
  └── dsa/prod/aes-key   # CI 암호화/복호화 키
```

**레이어 분리 원칙:**
```
GitHub Secrets        → 배포 파이프라인 전용 (GitHub Actions)
AWS Secrets Manager   → 앱 런타임에 키 조회
DB                    → 암호화된 CI 값 저장
```

---

## 5. 처리 흐름

### 5.1 회원가입 (CI 저장)

```
본인인증 완료 → CI 수신
  → SHA-256 해시 → ci_hash 저장 (중복 가입 체크)
  → AWS Secrets Manager에서 AES 키 조회
  → AES-256 암호화 → ci_encrypted 저장
  → 원본 CI는 메모리에서 즉시 폐기
```

### 5.2 중복 가입 체크

```
신규 본인인증 → CI 수신
  → SHA-256 해시 생성
  → ci_hash 컬럼과 비교
  → 일치하면 중복 가입 차단
```

### 5.3 원본 CI 조회 (필요 시)

```
조회 요청
  → AWS Secrets Manager에서 AES 키 조회
  → ci_encrypted 복호화
  → 원본 CI 반환 (응답 후 즉시 폐기)
```

---

## 6. 구현 코드

### 6.1 CI 암호화/복호화

```java
@Component
public class CiEncryptService {

    private final String aesKey;

    public CiEncryptService(SecretsManagerClient secretsManager) {
        // 서버 시작 시 AWS Secrets Manager에서 키 로딩
        this.aesKey = secretsManager.getSecret("dsa/prod/aes-key");
    }

    public String encrypt(String ci) {
        // AES-256 암호화
        return AesUtils.encrypt(ci, aesKey);
    }

    public String decrypt(String ciEncrypted) {
        // AES-256 복호화
        return AesUtils.decrypt(ciEncrypted, aesKey);
    }

    public String hash(String ci) {
        // SHA-256 단방향 해시 (중복 체크용)
        return DigestUtils.sha256Hex(ci);
    }
}
```

### 6.2 회원가입 처리

```java
@Service
public class UserService {

    @Transactional
    public User register(RegisterRequest req) {
        String ci = req.getCi(); // 본인인증에서 받은 CI

        // 중복 가입 체크
        String ciHash = ciEncryptService.hash(ci);
        if (userRepo.existsByCiHash(ciHash)) {
            throw new DuplicateUserException("이미 가입된 사용자입니다.");
        }

        // 암호화 저장
        User user = User.builder()
            .ciHash(ciHash)
            .ciEncrypted(ciEncryptService.encrypt(ci))
            .userType(req.getUserType())
            .build();

        // 원본 CI 즉시 폐기 (GC 대상)
        ci = null;

        return userRepo.save(user);
    }
}
```

### 6.3 JPA 컨버터 (ci_encrypted 자동 처리)

```java
@Converter
public class CiEncryptConverter implements AttributeConverter<String, String> {

    @Autowired
    private CiEncryptService ciEncryptService;

    @Override
    public String convertToDatabaseColumn(String ci) {
        return ciEncryptService.encrypt(ci);
    }

    @Override
    public String convertToEntityAttribute(String ciEncrypted) {
        return ciEncryptService.decrypt(ciEncrypted);
    }
}
```

---

## 7. 보안 주의사항

### 절대 금지
- CI 원본값 로그 출력
- CI 원본값 API 응답에 포함
- 암호화 키 코드 하드코딩
- 암호화 키 DB 저장

### 로그 처리
```java
// 금지
log.info("CI 저장: ci={}", ci);

// 허용
log.info("CI 저장 완료: userId={}", userId);
```

### API 응답
```java
public class UserResponse {
    // CI 관련 필드는 응답에서 제외
    @JsonIgnore
    private String ciEncrypted;

    @JsonIgnore
    private String ciHash;
}
```

---

## 8. 재원생-보호자 연결

CI는 개인 식별용이고, 재원생-보호자 관계는 별도 테이블로 관리한다.

```sql
student_guardians (
  id           UUID PK,
  student_id   UUID REFERENCES users(id),
  guardian_id  UUID REFERENCES users(id),
  relation     VARCHAR(10),   -- MOTHER | FATHER | OTHER
  created_at   TIMESTAMPTZ DEFAULT now()
)
```

**연결 흐름:**
```
보호자 앱 회원가입 (본인인증)
  → 자녀 학번 또는 전화번호 입력
  → DSA DB에서 학생 조회
  → student_guardians 매핑 생성
```

---

## 9. 관련 법령

- 개인정보보호법: CI는 민감 개인정보, 유출 시 과징금 대상
- 정보통신망법: 암호화 저장 의무
- CI 원본은 본인인증 기관(NICE, KCB 등)만 보유 가능
