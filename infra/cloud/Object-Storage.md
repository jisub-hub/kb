---
tags:
  - cloud
  - object-storage
  - s3
  - presigned-url
  - file-upload
created: 2026-06-16
---

# Object Storage & Presigned URL

> [!summary] 한 줄 요약
> Object Storage(S3)는 파일을 URL로 직접 업로드·다운로드할 수 있는 저장소. **Presigned URL**을 쓰면 서버를 거치지 않고 클라이언트가 S3에 직접 올리고 받아 **서버 대역폭·메모리 오버헤드를 제거**한다.

---

## 1. Object Storage 기본 개념

```
[파일 저장 방식 비교]
  Block Storage (EBS, 로컬 디스크)
    → 파일 시스템 위에 마운트, OS가 관리
    → 빠른 I/O, 단일 서버에 종속

  File Storage (NFS, EFS)
    → 네트워크 파일 시스템, 여러 서버가 공유
    → 느림, 용량 제한

  Object Storage (S3, GCS, MinIO)
    → 객체(파일 + 메타데이터)를 flat namespace에 저장
    → 무제한 확장, HTTP API로 접근
    → 파일 시스템 계층 없음 (경로는 key 이름일 뿐)

[S3 구조]
  Bucket (유일한 이름, 리전 귀속)
    └── Object (Key = "path/file.jpg", Value = 바이너리, Metadata)

  예:
  Bucket: my-company-assets
  Key:    uploads/2026/06/users/123/profile.jpg
  URL:    https://my-company-assets.s3.ap-northeast-2.amazonaws.com/uploads/...
```

---

## 2. 서버 경유 vs Presigned URL 비교

```
[서버 경유 방식 (비효율)]

  클라이언트 ──파일 전송──→ Spring 서버 ──파일 전송──→ S3
              10MB           10MB (중복)
  
  문제점:
  ① 서버 대역폭 2× 소비 (업로드 + S3 전송)
  ② 서버 메모리: MultipartFile 전체를 메모리에 올림
  ③ 서버 처리 시간: 클라이언트가 올릴 때까지 스레드 점유
  ④ 서버 타임아웃: 대용량 파일 시 서버 timeout 위험
  ⑤ 보안: 서버가 모든 파일 내용을 처리 (민감 데이터 노출)

[Presigned URL 방식 (권장)]

  클라이언트 ──요청──→ Spring 서버 ──생성──→ S3 (서명된 URL 발급)
             10ms 응답
  클라이언트 ──파일 전송──────────────────→ S3 (직접 업로드)
              10MB (서버 통하지 않음)
  
  장점:
  ① 서버 대역폭 절감 (파일이 서버를 통과하지 않음)
  ② 서버 메모리 절감
  ③ 서버 스레드 절감
  ④ S3가 직접 처리 → 더 빠른 업로드 (엣지 위치)
  ⑤ URL 만료 시간으로 보안 제어 (예: 15분만 유효)
```

---

## 3. Presigned URL 동작 원리

```
[업로드 Presigned URL (PUT)]

  1. 클라이언트: "profile.jpg 올릴게요" → 서버 API
  2. 서버: AWS IAM 자격증명으로 S3에 서명된 PUT URL 생성
           URL 예: https://bucket.s3.amazonaws.com/key?X-Amz-Signature=...&X-Amz-Expires=900
  3. 서버: URL을 클라이언트에 응답
  4. 클라이언트: 해당 URL로 직접 S3에 PUT 요청 (파일 전송)
  5. S3: 서명 검증 → 만료 시간 확인 → 저장 완료

[다운로드 Presigned URL (GET)]

  Private S3 버킷 (퍼블릭 접근 차단):
  1. 클라이언트: "파일 주세요" → 서버
  2. 서버: 권한 확인 후 서명된 GET URL 생성 (유효시간 5분)
  3. 클라이언트: URL로 S3에서 직접 다운로드

  → 서버에서 권한 체크, 실제 파일 전송은 S3가 직접
```

---

## 4. Spring Boot 구현

### 의존성

```xml
<dependency>
  <groupId>software.amazon.awssdk</groupId>
  <artifactId>s3</artifactId>
  <version>2.25.0</version>
</dependency>
<dependency>
  <groupId>software.amazon.awssdk</groupId>
  <artifactId>s3-presigner</artifactId>
  <version>2.25.0</version>
</dependency>
```

```yaml
cloud:
  aws:
    credentials:
      access-key: ${AWS_ACCESS_KEY}
      secret-key: ${AWS_SECRET_KEY}
    region:
      static: ap-northeast-2
    s3:
      bucket: my-company-assets
```

### S3 서비스

```java
@Service
@RequiredArgsConstructor
public class S3PresignedService {

    private final S3Presigner presigner;
    private final S3Client s3Client;

    @Value("${cloud.aws.s3.bucket}")
    private String bucket;

    // ── 업로드용 Presigned PUT URL ───────────────────────────────
    public PresignedUploadResponse generateUploadUrl(
            String directory, String originalFilename, String contentType) {

        String key = generateKey(directory, originalFilename);

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
            .signatureDuration(Duration.ofMinutes(15))        // 15분 유효
            .putObjectRequest(r -> r
                .bucket(bucket)
                .key(key)
                .contentType(contentType)
                // 파일 크기 제한 (서버 측 정책)
                .contentLength(50 * 1024 * 1024L)            // 최대 50MB
            )
            .build();

        PresignedPutObjectRequest presigned = presigner.presignPutObject(presignRequest);

        return PresignedUploadResponse.builder()
            .uploadUrl(presigned.url().toString())
            .key(key)
            .expiresAt(Instant.now().plus(Duration.ofMinutes(15)))
            .build();
    }

    // ── 다운로드용 Presigned GET URL ─────────────────────────────
    public String generateDownloadUrl(String key) {
        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
            .signatureDuration(Duration.ofMinutes(5))         // 5분 유효
            .getObjectRequest(r -> r
                .bucket(bucket)
                .key(key)
            )
            .build();

        return presigner.presignGetObject(presignRequest).url().toString();
    }

    // ── 파일 삭제 ─────────────────────────────────────────────────
    public void deleteObject(String key) {
        s3Client.deleteObject(r -> r.bucket(bucket).key(key));
    }

    private String generateKey(String directory, String originalFilename) {
        String ext = StringUtils.getFilenameExtension(originalFilename);
        String uuid = UUID.randomUUID().toString();
        return String.format("%s/%s/%s.%s",
            directory,
            LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy/MM")),
            uuid, ext);
    }
}
```

### API 컨트롤러

```java
@RestController
@RequestMapping("/api/files")
@RequiredArgsConstructor
public class FileController {

    private final S3PresignedService presignedService;
    private final FileMetadataRepository fileRepository;

    // 1단계: 업로드 URL 요청
    @PostMapping("/upload-url")
    public ResponseEntity<PresignedUploadResponse> getUploadUrl(
            @RequestBody UploadUrlRequest req,
            @AuthenticationPrincipal UserPrincipal user) {

        // 권한 체크, 파일 타입 검증
        validateFileType(req.contentType());

        PresignedUploadResponse response = presignedService
            .generateUploadUrl("uploads/" + user.getId(), req.filename(), req.contentType());

        return ResponseEntity.ok(response);
    }

    // 2단계: 업로드 완료 후 메타데이터 저장
    @PostMapping("/confirm")
    public ResponseEntity<FileDto> confirmUpload(
            @RequestBody ConfirmUploadRequest req,
            @AuthenticationPrincipal UserPrincipal user) {

        // 실제로 S3에 파일이 있는지 확인
        boolean exists = presignedService.objectExists(req.key());
        if (!exists) {
            return ResponseEntity.badRequest().build();
        }

        FileMetadata file = FileMetadata.builder()
            .key(req.key())
            .originalName(req.originalName())
            .uploaderId(user.getId())
            .createdAt(Instant.now())
            .build();
        fileRepository.save(file);

        return ResponseEntity.ok(FileDto.from(file));
    }

    // 다운로드 URL 발급 (권한 체크 후)
    @GetMapping("/{fileId}/download-url")
    public ResponseEntity<String> getDownloadUrl(
            @PathVariable Long fileId,
            @AuthenticationPrincipal UserPrincipal user) {

        FileMetadata file = fileRepository.findById(fileId)
            .orElseThrow(() -> new NotFoundException("파일 없음"));

        // 권한: 업로더 본인 또는 관리자만
        if (!file.getUploaderId().equals(user.getId()) && !user.isAdmin()) {
            return ResponseEntity.status(403).build();
        }

        String url = presignedService.generateDownloadUrl(file.getKey());
        return ResponseEntity.ok(url);
    }
}
```

---

## 5. 클라이언트 업로드 흐름

```javascript
// 프론트엔드 (React/Vue)

async function uploadFile(file) {
  // 1. 서버에서 업로드 URL 발급
  const { uploadUrl, key } = await fetch('/api/files/upload-url', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      filename: file.name,
      contentType: file.type,
    })
  }).then(r => r.json());

  // 2. S3에 직접 PUT 업로드 (서버 통하지 않음)
  await fetch(uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': file.type },
    body: file,                          // 파일 바이너리 직접
  });

  // 3. 서버에 업로드 완료 알림 (메타데이터 저장)
  const fileInfo = await fetch('/api/files/confirm', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ key, originalName: file.name })
  }).then(r => r.json());

  return fileInfo;
}
```

---

## 6. CORS 설정 (S3 버킷)

```json
[
  {
    "AllowedHeaders": ["Content-Type", "Content-Length"],
    "AllowedMethods": ["PUT", "GET"],
    "AllowedOrigins": ["https://app.example.com"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
```

---

## 7. MinIO (Self-hosted Object Storage)

```yaml
# docker-compose.yml — 로컬 개발용 MinIO
services:
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"     # API
      - "9001:9001"     # Web Console
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    volumes:
      - minio_data:/data
```

```java
// application-local.yml — MinIO 로컬 설정
cloud:
  aws:
    s3:
      endpoint: http://localhost:9000   # MinIO 엔드포인트
      bucket: dev-bucket
    credentials:
      access-key: minioadmin
      secret-key: minioadmin123
    region:
      static: us-east-1                # MinIO는 리전 무관

// MinIO를 S3 SDK로 사용
@Bean
public S3Client s3Client() {
    return S3Client.builder()
        .endpointOverride(URI.create("http://localhost:9000"))
        .credentialsProvider(StaticCredentialsProvider.create(
            AwsBasicCredentials.create("minioadmin", "minioadmin123")))
        .region(Region.US_EAST_1)
        .forcePathStyle(true)          // MinIO 필수
        .build();
}
```

---

## 8. 보안 체크리스트

```
✅ 버킷 Public Access 차단 (Block All Public Access)
✅ Presigned URL 만료 시간: 업로드 15분, 다운로드 5~60분
✅ 업로드 전 파일 타입 검증 (Content-Type + 확장자)
✅ 업로드 전 파일 크기 제한 (S3 Policy + 서버 측 검증)
✅ 업로드 후 서버에서 S3 Object 존재 확인 (confirm API)
✅ IAM 최소 권한: Lambda/Spring 서버에 s3:GetObject, s3:PutObject만
✅ S3 버킷 암호화: SSE-S3 또는 SSE-KMS
✅ 민감 파일 (계약서 등): 다운로드 URL에 만료 시간 짧게
```

---

## 9. 관련
- [[Serverless-Lambda]] — Lambda + S3 이벤트 트리거 (업로드 완료 후 처리)
- [[../data-pipeline/Apache-Airflow]] — 대용량 파일 ETL 시 S3 스테이징
