---
tags:
  - security
  - cryptography
  - hash
  - encryption
  - pki
created: 2026-06-16
---

# 암호화 (Cryptography)

> [!summary] 한 줄 요약
> **단방향 해시**(복호화 불가, 무결성 검증·패스워드 저장), **대칭키**(빠름, 키 공유 문제), **비대칭키**(공개키 배포 가능, 느림) 세 축이 현대 암호의 기반. TLS는 이 세 가지를 모두 조합해 사용한다.

---

## 1. 단방향 해시 (One-Way Hash)

```
평문 ──────────────► Hash 함수 ──► 해시값(고정 길이)
"password123"                       "ef92b778..."
         역산 불가 (단방향)
```

### 특성

| 특성 | 설명 |
|---|---|
| **결정적** | 같은 입력 → 항상 같은 출력 |
| **빠른 계산** | 임의 크기 입력 → 고정 길이 출력 |
| **역산 불가** | 해시값 → 원문 복원 수학적으로 불가능 |
| **충돌 저항** | 다른 입력 → 같은 출력(충돌) 찾기 어려움 |
| **눈사태 효과** | 입력 1비트 변화 → 출력 50% 이상 변화 |

### 해시 알고리즘 비교

| 알고리즘 | 출력 길이 | 상태 | 비고 |
|---|---|---|---|
| **MD5** | 128 bit | ⛔ 취약 | 충돌 공격 가능, 사용 금지 |
| **SHA-1** | 160 bit | ⛔ 취약 | 2017년 구글 SHAttered 충돌 시연 |
| **SHA-256** | 256 bit | ✅ 안전 | SHA-2 계열, 현재 표준 |
| **SHA-512** | 512 bit | ✅ 안전 | SHA-2 계열, 더 큰 출력 |
| **SHA-3 (Keccak)** | 가변 | ✅ 안전 | SHA-2와 다른 설계(스펀지 구조) |
| **bcrypt** | 60 char | ✅ 패스워드 | 의도적으로 느림, salt 내장 |
| **Argon2** | 가변 | ✅ 패스워드 최강 | PHC 수상, GPU 저항 |
| **scrypt** | 가변 | ✅ 패스워드 | 메모리 집약적 |

### 패스워드 저장 — 왜 bcrypt/Argon2인가

```
일반 SHA-256만 쓰면:
  1. 공격자가 DB 탈취
  2. Rainbow Table (사전 계산 해시 테이블)로 빠르게 역산
  3. GPU로 초당 수십억 번 SHA-256 계산 가능

bcrypt/Argon2가 방어하는 이유:
  · Salt: 사용자마다 랜덤값 추가 → Rainbow Table 무력화
  · Work Factor: 의도적으로 수백ms 걸리게 설계
    → 공격자가 초당 수십 번만 시도 가능 (GPU 무력화)
```

```java
// Spring Security — BCrypt 사용
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // cost factor 12 = ~300ms/hash
}

// 저장
String hashed = passwordEncoder.encode("password123");
// "$2a$12$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy"
// ↑ 알고리즘$cost$salt(22chars)hash(31chars)

// 검증 (타이밍 공격 방어 포함)
boolean matches = passwordEncoder.matches("password123", hashed);
```

### HMAC (Hash-based Message Authentication Code)

```
HMAC = Hash(key || message)  — 단순 연결은 취약
HMAC = Hash(key XOR opad || Hash(key XOR ipad || message))  — RFC 2104

목적: 무결성 + 인증 (수신자가 올바른 키 보유자임을 증명)
용도: JWT 서명(HS256), API 요청 서명, Cookie 위변조 방지
```

```java
// HMAC-SHA256
Mac mac = Mac.getInstance("HmacSHA256");
SecretKeySpec key = new SecretKeySpec("secret-key".getBytes(), "HmacSHA256");
mac.init(key);
byte[] hmac = mac.doFinal("message".getBytes());
```

---

## 2. 대칭키 암호화 (Symmetric Encryption)

```
평문 ──[공유 비밀키]──► 암호문 ──[공유 비밀키]──► 평문
                  암호화                   복호화
```

**장점**: 빠름 (AES는 하드웨어 가속 지원)  
**단점**: 키 배송 문제 — 상대방에게 어떻게 안전하게 키를 전달할 것인가?

### AES (Advanced Encryption Standard)

현재 대칭키 표준. 블록 암호 (128비트 블록 단위 처리).

| 종류 | 키 길이 | 안전성 |
|---|---|---|
| AES-128 | 128 bit | 충분 (일반용) |
| AES-192 | 192 bit | 충분 |
| **AES-256** | 256 bit | 권장 (민감 데이터) |

### 운용 모드 (Mode of Operation)

```
[ECB — Electronic Codebook] ⛔ 사용 금지
  같은 블록 → 같은 암호문
  패턴이 그대로 드러남 (리눅스 펭귄 암호화 예시 유명)

[CBC — Cipher Block Chaining]
  C₁ = E(P₁ XOR IV)
  C₂ = E(P₂ XOR C₁)   ← 이전 암호문을 XOR
  IV(초기화 벡터)가 달라야 같은 평문도 다른 암호문 생성
  ⚠ PKCS7 패딩 오라클 공격에 취약할 수 있음

[GCM — Galois/Counter Mode] ✅ 권장
  CTR 모드(병렬 가능) + GHASH(인증 태그)
  AES-256-GCM = 기밀성 + 무결성(AEAD) 동시 제공
  TLS 1.3에서 필수 사용
```

```java
// AES-256-GCM 암호화
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
byte[] iv = new byte[12];  // 96비트 IV (GCM 표준)
new SecureRandom().nextBytes(iv);

GCMParameterSpec spec = new GCMParameterSpec(128, iv);  // 128비트 인증 태그
SecretKey key = new SecretKeySpec(keyBytes, "AES");  // 32바이트 = AES-256

cipher.init(Cipher.ENCRYPT_MODE, key, spec);
byte[] ciphertext = cipher.doFinal(plaintext);
// 저장: IV + ciphertext (IV는 비밀 아님, 재사용만 금지)
```

---

## 3. 비대칭키 암호화 (Asymmetric / Public Key Encryption)

```
[키 쌍 생성]
  개인키(Private Key) — 절대 공개 금지, 본인만 보관
  공개키(Public Key)  — 누구에게나 배포 가능

[암호화]
  발신자: 수신자의 공개키로 암호화
  수신자: 자신의 개인키로 복호화
  → 공개키로 잠근 것은 개인키만 열 수 있음

[서명]
  서명자: 자신의 개인키로 서명
  검증자: 서명자의 공개키로 검증
  → 개인키 보유자만 서명 가능, 위조 불가
```

### RSA

```
키 생성 원리:
  1. 두 소수 p, q 선택 (각 2048+ 비트)
  2. n = p × q  (공개)
  3. φ(n) = (p-1)(q-1)
  4. e: φ(n)과 서로소인 공개 지수 (보통 65537)
  5. d: e의 모듈러 역원 (개인키)

암호화: C = M^e mod n
복호화: M = C^d mod n

보안 근거: n을 소인수분해해서 p,q를 구하는 것이 수학적으로 어려움
```

| RSA 키 길이 | 상태 | 권장 용도 |
|---|---|---|
| 1024 bit | ⛔ 취약 (2010년 이후 금지) | — |
| 2048 bit | ✅ 최소 권장 | TLS 인증서 일반 |
| **4096 bit** | ✅ 권장 | 장기 보안, CA 루트 |

**단점**: 느림 (ECC 대비 100배 이상 느린 연산)

### ECC (Elliptic Curve Cryptography)

```
타원곡선 방정식: y² = x³ + ax + b (mod p)

보안 근거: 이산 로그 문제 (ECDLP)
  → 점 P에서 k번 덧셈한 점 Q=kP를 알아도 k를 구하기 어려움

장점: RSA 2048비트 ≈ ECC 256비트 보안 수준
      → 키 크기 작음, 연산 빠름
```

| 곡선 | 키 길이 | RSA 동등 | 용도 |
|---|---|---|---|
| P-256 (secp256r1) | 256 bit | RSA-3072 | TLS 1.3, JWT ES256 |
| P-384 (secp384r1) | 384 bit | RSA-7680 | 고보안 |
| **X25519** | 255 bit | RSA-3072 | ECDH 키 교환 (TLS 1.3 기본) |
| **Ed25519** | 255 bit | RSA-3072 | 서명, SSH 키 |

```java
// ECDSA 서명 (Java)
KeyPairGenerator kpg = KeyPairGenerator.getInstance("EC");
kpg.initialize(new ECGenParameterSpec("secp256r1"));
KeyPair kp = kpg.generateKeyPair();

Signature sig = Signature.getInstance("SHA256withECDSA");
sig.initSign(kp.getPrivate());
sig.update(data);
byte[] signature = sig.sign();

sig.initVerify(kp.getPublic());
sig.update(data);
boolean valid = sig.verify(signature);
```

### Diffie-Hellman 키 교환

```
목적: 비밀 채널 없이 두 당사자가 공유 비밀(세션키)을 협의

[ECDH 프로세스]
  Alice: a (개인), A = a×G (공개)
  Bob:   b (개인), B = b×G (공개)

  A, B를 교환 (공개 채널 OK)

  Alice: a×B = a×b×G
  Bob:   b×A = b×a×G
  → 두 결과 같음 = 공유 비밀

  도청자가 A, B를 알아도 a, b를 구하는 것 = ECDLP (어려움)
```

---

## 4. TLS 핸드셰이크 — 암호화 기법의 통합

```
Client                              Server
  │── ClientHello ──────────────────►│
  │   (지원 cipher suites, TLS 버전)  │
  │                                  │
  │◄── ServerHello ──────────────────│
  │    (선택한 cipher suite)          │
  │◄── Certificate ──────────────────│
  │    (서버 X.509 인증서)             │
  │                                  │
  │  [인증서 검증: CA 서명 확인]         │
  │                                  │
  │── ClientKeyExchange ────────────►│  (TLS 1.2 RSA)
  │   또는 ECDH Key Agreement         │  (TLS 1.3)
  │                                  │
  │  [양쪽: 세션키 도출 (PRF/HKDF)]    │
  │                                  │
  │── Finished (HMAC) ──────────────►│
  │◄── Finished (HMAC) ──────────────│
  │                                  │
  │════ AES-256-GCM 암호화 통신 ══════│
```

**TLS 1.3 cipher suite 예:**  
`TLS_AES_256_GCM_SHA384`  
= 대칭키(AES-256-GCM) + 해시(SHA-384), 키교환은 ECDH로 별도 협상

---

## 5. PKI (Public Key Infrastructure) — 인증서 체계

```
[신뢰 체계]
  Root CA (최상위, 오프라인 보관)
    └── Intermediate CA (중간 CA)
          └── 서버 인증서 (서비스 도메인)

신뢰 확인:
  브라우저/OS가 Root CA 목록 내장
  → 인증서 체인 서명 검증
  → 최종적으로 Root CA까지 이어지면 신뢰
```

```
X.509 인증서 주요 필드:
  Subject:    CN=example.com, O=Example Corp
  Issuer:     CN=Let's Encrypt Authority X3
  Valid:      2026-01-01 ~ 2026-04-01
  Public Key: EC secp384r1 (공개키)
  SAN:        DNS:example.com, DNS:www.example.com
  Signature:  (Issuer의 개인키로 서명)
```

---

## 6. 실전 선택 기준

| 목적 | 권장 알고리즘 | 비고 |
|---|---|---|
| **패스워드 저장** | Argon2id | bcrypt도 OK. SHA-256 단독 금지 |
| **데이터 암호화 (저장)** | AES-256-GCM | IV 재사용 절대 금지 |
| **데이터 암호화 (전송)** | TLS 1.3 | AES-256-GCM 또는 ChaCha20 |
| **API 무결성 서명** | HMAC-SHA256 | JWT HS256 |
| **공개키 서명 (인증서)** | ECDSA P-256 | JWT ES256 |
| **SSH 키** | Ed25519 | RSA 4096도 OK |
| **TLS 키 교환** | ECDH X25519 | TLS 1.3 기본 |
| **파일 무결성** | SHA-256 | MD5/SHA-1 금지 |

---

## 7. 관련
- [[Web-Attacks]] · [[Network-Attacks]] · [[../spring/security/OAuth2-JWT]] · [[../spring/security/Spring-Security]]
