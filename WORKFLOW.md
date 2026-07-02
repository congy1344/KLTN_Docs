# WORKFLOW.md — Luồng hoạt động hệ thống

## Luồng tổng quát (End-to-End)

```
0. User đăng nhập
         ↓
1. User upload ZIP / nhập GitHub URL
         ↓
2. Static Analysis (JavaParser trích xuất cấu trúc code)
         ↓
3. User chọn cách tạo Business Rule
         ↓
4. AI sinh hoặc review/gợi ý Business Rule, user duyệt
         ↓
5. AI sinh Test Plan (Happy/Boundary/Exception/Edge)
         ↓
6. ⚙️ HITL: User review Test Plan
         ↓
7. AI sinh Test Case (8 trường)
         ↓
8. ⚙️ HITL: User review Test Case
         ↓
9. AI sinh Unit Test (JUnit 5 + Mockito)
         ↓
10. User download Unit Test mới/cải thiện, chạy JaCoCo ở máy local
         ↓
11. User upload file JaCoCo XML báo cáo
         ↓
12. Hệ thống parse XML, hiển thị coverage
         ↓
13. Coverage Gap Detection → đề xuất bổ sung test case
         ↓
14. User xuất báo cáo JSON/Markdown
```

## Chi tiết từng bước

### Bước 0: Đăng nhập và phân quyền

**Hành động user:**
- Đăng ký hoặc đăng nhập trước khi sử dụng hệ thống.
- Sau khi đăng nhập, frontend gửi token/session cho các API cần bảo vệ.

**Phân quyền:**
- `USER`: thao tác với project do chính mình tạo, bao gồm upload/clone, analysis, review, generate test và export.
- `ADMIN`: xem/quản lý toàn bộ project, quản lý user và hỗ trợ kiểm tra dữ liệu hệ thống.

**Hành động hệ thống:**
- Xác thực request trước khi gọi các endpoint project/test generation.
- Gắn `ownerUserId` cho project khi user upload ZIP hoặc clone GitHub.
- Từ chối truy cập nếu user thường truy cập project không thuộc sở hữu của mình.

### Bước 1: Nhập source code

**Input từ user:**
- File ZIP chứa source code Java Spring Boot
- HOẶC URL GitHub repository public

**Hành động hệ thống:**
- Nếu là ZIP: giải nén vào thư mục tạm
- Nếu là GitHub URL: dùng JGit để clone về thư mục tạm
- Validate đây có phải Spring Boot project (kiểm tra `pom.xml` hoặc `build.gradle`)
- Tạo record `Project` trong database với status = `UPLOADED` và owner là user hiện tại
- Ngay lập tức gọi Static Analysis trước khi trả response cho user
- Nếu analysis thất bại hoàn toàn: rollback record project, xóa source đã lưu và trả lỗi `ANALYSIS_ERROR`
- Nếu chỉ một số file production không parse được: bỏ qua các file đó, lưu `failedParseFilePaths`, vẫn phân tích các file còn lại

**Endpoint:**
- `POST /api/projects/upload` (multipart/form-data) — Upload ZIP
- `POST /api/projects/github` (body: `{ url: string }`) — Clone GitHub

### Bước 2: Static Analysis

**Trigger:** Tự động, đồng bộ trong request upload/clone sau khi bước 1 validate thành công

**Hành động:**
- `AnalysisService.analyze(projectId)` được gọi
- Sử dụng JavaParser quét toàn bộ file `.java`
- Parser cấu hình language level `JAVA_21`, hướng tới source Java 8-21
- Parse production source trong các source root `src/main/java` (hỗ trợ multi-module) để tạo Static Analysis Context chính
- Đọc `src/test/java` trong luồng riêng Existing Test Analysis; không đưa test class/test method vào chỉ số production Classes/Methods/Endpoints
- Bỏ qua `target`, `build`, generated source
- Nếu một số file production không parse được do giới hạn parser, hệ thống skip file đó và thống kê riêng thay vì fail toàn bộ project
- Trích xuất:
  - Danh sách Class (`@Service`, `@Controller`, `@Repository`)
  - Enum và record trong production source
  - Danh sách Method với params, return type, throws
  - Relevant annotation cho component, endpoint, validation, security, transaction và persistence
  - Danh sách REST Endpoint với HTTP method, path
  - Quan hệ Controller Method → Service Method khi resolve chắc chắn
  - Quan hệ Service → Repository
- Lưu kết quả vào database
- Cập nhật status project thành `ANALYZED`
- Trả thêm `existingTestFiles` để UI thông báo số test có sẵn đã được phát hiện và sẽ được dùng làm context cải thiện test

**Endpoint:**
- `POST /api/projects/{id}/analyze` — Chỉ dùng khi muốn re-analyze
- `GET /api/projects/{id}/analysis` — Xem kết quả
- `GET /api/projects/{id}/analysis/manifest` — Export manifest JSON deterministic
- `POST /api/projects/{id}/analysis/manifest/validate` — So sánh ground truth, trả missing/unexpected

### Bước 2.5: Existing Test Analysis

**Trigger:** Tự động sau Static Analysis nếu project có `src/test/java`

**Hành động:**
- Đọc file test có sẵn, ưu tiên test dùng JUnit 5/JUnit 4/Mockito.
- Trích xuất metadata cơ bản: test class, package, test method, class under test nếu suy luận được, imports, assertions, mocks và source code test.
- Liên kết mềm test có sẵn với production class/method theo tên class, package, `@InjectMocks`, constructor usage hoặc direct method call nếu xác định được.
- Không sửa file gốc trong source upload; chỉ lưu context và đề xuất cải thiện trong database/output.

**Mục đích:**
- Tránh sinh test trùng với test đã có.
- Cải thiện test còn yếu: thiếu boundary/exception, thiếu assert, mock chưa đủ hoặc chưa cover Business Rule.
- Bổ sung test mới quanh các gap còn thiếu.

### Bước 3: Chọn cách tạo Business Rule

**Trigger:** User mở màn Business Rule sau khi xem kết quả analysis.

**Hai hướng được hỗ trợ:**

1. **AI auto sinh BR**
   - User bấm "AI sinh Business Rules".
   - GreyTest đọc Static Analysis Context, source method, annotation, relation và Existing Test Context nếu có.
   - AI sinh danh sách Business Rule ở trạng thái `PENDING_REVIEW`.

2. **User nhập BR, AI review và đề xuất thêm**
   - User tự nhập hoặc import một vài Business Rule ban đầu.
   - User bấm "AI review & gợi ý".
   - GreyTest gửi BR của user cùng source context cho AI.
   - AI đánh giá rule hiện có, gợi ý sửa rule mơ hồ/trùng/lệch method và đề xuất rule còn thiếu.

**Hành động:**
- Với hướng AI auto sinh, `AIAgentService.generateBusinessRules(projectId)` được gọi.
- Với hướng user nhập trước, `AIAgentService.reviewBusinessRules(projectId)` được gọi.
- Prompt dùng template `prompts/business-rule.md` hoặc `prompts/business-rule-review.md`.
- Gửi cho LLM kèm:
  - Danh sách method của các Service class
  - Code body của method (nếu ngắn) hoặc signature
  - Business Rule user đã nhập nếu có
  - Existing Test Context nếu có
- Parse response JSON, validate schema
- Lưu rule/gợi ý vào database với status = `PENDING_REVIEW`

**Lưu ý:**
- Mỗi Business Rule phải liên kết với ít nhất 1 method
- Format rule: 1 câu mô tả ngắn gọn yêu cầu nghiệp vụ
- AI chỉ sinh/review/gợi ý; người dùng duyệt mới được dùng để sinh Test Plan

### Bước 4: HITL — Duyệt Business Rule

**Hành động user:**
- Xem danh sách Business Rule do AI sinh hoặc gợi ý review từ BR user nhập
- **Sửa:** Edit text của rule
- **Thêm:** Thêm rule mới (nhập text + chọn method liên kết)
- **Xóa:** Xóa rule không phù hợp
- **Chấp nhận gợi ý:** Áp dụng nội dung AI đề xuất cho rule user nhập
- **Bỏ qua gợi ý:** Giữ rule hiện tại nếu user thấy phù hợp
- Bấm "Approve" để chuyển sang bước tiếp theo

**Endpoint:**
- `GET /api/projects/{id}/business-rules`
- `POST /api/projects/{id}/business-rules/generate` — AI auto sinh BR từ source
- `POST /api/projects/{id}/business-rules/review` — AI review BR user nhập và đề xuất thêm
- `POST /api/projects/{id}/business-rules` — Thêm rule mới
- `PUT /api/business-rules/{id}` — Sửa rule
- `DELETE /api/business-rules/{id}` — Xóa rule
- `POST /api/projects/{id}/business-rules/approve` — Phê duyệt

**Sau khi approve:**
- Status các rule chuyển sang `APPROVED`
- Tính `User Modification Rate` cho metric đánh giá
- Cho phép trigger bước 5

### Bước 5: AI sinh Test Plan

**Hành động:**
- `AIAgentService.generateTestPlan(projectId)` được gọi
- Với mỗi Business Rule, tạo prompt yêu cầu LLM sinh Test Plan
- Nếu có Existing Test Context, prompt nêu rõ test nào đã tồn tại để tránh tạo plan trùng lặp không cần thiết
- Test Plan có 4 loại: Happy Path, Boundary, Exception, Edge
- Không phải mọi rule đều có đủ 4 loại — LLM quyết định
- Lưu Test Plan vào database
- Tạo link Business Rule → Test Plan trong Traceability Matrix

**Endpoint:**
- `POST /api/projects/{id}/test-plans/generate`
- `GET /api/projects/{id}/test-plans`

### Bước 6: HITL — Review Test Plan

Tương tự bước 4 nhưng cho Test Plan.

### Bước 7: AI sinh Test Case

**Hành động:**
- `AIAgentService.generateTestCases(projectId)` được gọi
- Với mỗi Test Plan, sinh các Test Case cụ thể
- Mỗi Test Case có đủ 8 trường (xem `PROJECT_OVERVIEW.md`)
- Tạo link Test Plan → Test Case
- Nếu test sẵn đã cover một phần scenario, test case cần ghi rõ mục tiêu là cải thiện/bổ sung phần còn thiếu

### Bước 8: HITL — Review Test Case

Tương tự, user có thể sửa/thêm/xóa Test Case.

### Bước 9: AI sinh Unit Test

**Hành động:**
- `AIAgentService.generateUnitTests(projectId)` được gọi
- Với mỗi Test Case, sinh code Java sử dụng JUnit 5 + Mockito hoặc cải thiện test class hiện có nếu tìm được file test tương ứng
- Code phải đúng package, đúng class name `*Test.java`
- Lưu code vào database (cột TEXT), kèm loại output: test mới, cải thiện test hiện có hoặc bổ sung method test mới vào class test hiện có
- Tạo link Test Case → Unit Test

**Output cuối cùng:**
- File ZIP chứa unit test mới/cải thiện
- User download và merge/copy vào project gốc của họ

### Bước 10–11: User chạy JaCoCo và upload XML

**User làm:**
- Copy unit test vào project gốc
- Chạy lệnh `mvn test jacoco:report`
- File báo cáo sinh ra ở `target/site/jacoco/jacoco.xml`
- Upload file XML lên web qua giao diện

**Lý do không tự động hóa:**
- Server không biết môi trường build của project gốc (Maven/Gradle, Java version)
- Cần sandbox phức tạp, không hợp với phạm vi 2 sinh viên

### Bước 12: Parse JaCoCo XML

**Hành động:**
- `CoverageService.parseJacocoXml(file)` được gọi
- Parse XML, trích xuất:
  - Line coverage per class/method
  - Branch coverage per method
  - Missed lines, missed branches
- Lưu kết quả vào database
- Tính tổng coverage cho cả project

### Bước 13: Coverage Gap Detection

**Hành động:**
- `CoverageService.detectCoverageGap(projectId)`
- Tìm method có line coverage < 80% hoặc branch coverage < 70%
- Lấy code những đoạn chưa cover
- Gửi cho LLM kèm prompt yêu cầu sinh thêm test case
- Hiển thị đề xuất cho user, user review và approve

### Bước 14: Xuất báo cáo

**Định dạng:**
- **JSON:** Toàn bộ dữ liệu structured (project, rules, plans, cases, tests, traceability, coverage)
- **Markdown:** Báo cáo dễ đọc, có table và section rõ ràng

**Endpoint:**
- `GET /api/projects/{id}/export?format=json`
- `GET /api/projects/{id}/export?format=markdown`

## State Machine

Mỗi project có một state machine rõ ràng:

```
UPLOADED
   ↓ (analyze)
ANALYZED
   ↓ (generate business rules)
BR_PENDING_REVIEW
   ↓ (approve)
BR_APPROVED
   ↓ (generate test plan)
PLAN_PENDING_REVIEW
   ↓ (approve)
PLAN_APPROVED
   ↓ (generate test cases)
CASE_PENDING_REVIEW
   ↓ (approve)
CASE_APPROVED
   ↓ (generate unit tests)
TEST_GENERATED
   ↓ (upload jacoco xml)
COVERAGE_ANALYZED
   ↓ (export)
COMPLETED
```

Có thể quay lùi (ví dụ: muốn sinh lại Test Case thì quay về `PLAN_APPROVED`).

## Xử lý lỗi

### Khi LLM trả về sai format
- Retry tối đa 2 lần
- Nếu vẫn lỗi: chuyển job sang status `FAILED`, hiển thị lỗi cho user

### Khi parse JaCoCo XML lỗi
- Validate XML schema trước khi parse
- Báo lỗi rõ ràng: "File không phải JaCoCo XML hợp lệ"

### Khi upload ZIP không phải Spring Boot
- Báo lỗi: "Không tìm thấy pom.xml hoặc build.gradle"
- Vẫn cho user thử với project bất kỳ (chế độ relaxed) — optional
