# ARCHITECTURE.md — Kiến trúc hệ thống

## Kiến trúc tổng thể

**Layered Architecture + Service-Oriented Pattern**

- Backend Spring Boot tổ chức theo Layered Architecture (Controller → Service → Repository)
- Trong tầng Service, tách thành nhiều Service độc lập theo chức năng

## Sơ đồ tổng thể

```
┌─────────────────────────────────────────────────┐
│                  FRONTEND                       │
│             React + TypeScript                  │
│                                                 │
│  ┌──────────┐ ┌──────────┐ ┌─────────────────┐  │
│  │ Project  │ │  Review  │ │   Traceability  │  │
│  │  Upload  │ │   HITL   │ │     Matrix      │  │
│  └──────────┘ └──────────┘ └─────────────────┘  │
└───────────────────┬─────────────────────────────┘
                    │ REST API (JSON)
┌───────────────────▼─────────────────────────────┐
│              BACKEND (Spring Boot)              │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │          Controller Layer               │    │
│  │  /api/auth /api/projects /api/agent     │    │
│  └─────────────────┬───────────────────────┘    │
│                    │                            │
│  ┌─────────────────▼───────────────────────┐    │
│  │           Service Layer                 │    │
│  │                                         │    │
│  │  ┌─────────────┐   ┌─────────────────┐  │    │
│  │  │   Analysis  │   │   AI Agent      │  │    │
│  │  │   Service   │   │   Service       │  │    │
│  │  └─────────────┘   └─────────────────┘  │    │
│  │                                         │    │
│  │  ┌─────────────┐   ┌─────────────────┐  │    │
│  │  │ Traceability│   │   Coverage      │  │    │
│  │  │   Service   │   │   Service       │  │    │
│  │  └─────────────┘   └─────────────────┘  │    │
│  └─────────────────┬───────────────────────┘    │
│                    │                            │
│  ┌─────────────────▼───────────────────────┐    │
│  │         Repository Layer                │    │
│  │         (Spring Data JPA)               │    │
│  └─────────────────┬───────────────────────┘    │
└────────────────────┼────────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │      PostgreSQL         │
        └─────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │      LLM Provider       │
        │   OpenAI / Google       │
        └─────────────────────────┘
```

## Các tầng (Layers)

### 1. Controller Layer
- Nhận HTTP request từ React
- Validate input cơ bản (DTO validation)
- Gọi Service tương ứng
- Trả về JSON response
- **KHÔNG** chứa business logic

**Các Controller chính:**
- `ProjectController` — Quản lý project (upload, list, delete)
- `AnalysisController` — Endpoint phân tích source code
- `AuthController` — Đăng ký, đăng nhập, lấy thông tin user hiện tại
- `UserController` — Quản lý user ở phạm vi ADMIN
- `BusinessRuleController` — CRUD Business Rule
- `AgentController` — Trigger AI sinh test plan/case/unit test
- `ReviewController` — Approve/reject các bước HITL
- `TraceabilityController` — Truy vấn Traceability Matrix
- `CoverageController` — Upload JaCoCo XML, lấy coverage report
- `ExportController` — Xuất JSON/Markdown

### 2. Service Layer
Tầng quan trọng nhất, chứa toàn bộ logic nghiệp vụ. Chia thành các service theo nhóm chức năng:

#### ProjectService
**Trách nhiệm:** Nhập source từ ZIP/GitHub, validate project và điều phối Static Analysis tự động.

Sau khi tạo project ở trạng thái `UPLOADED`, service gắn owner là user hiện tại rồi gọi `AnalysisService.analyze()` ngay trong
cùng transaction. Nếu analysis lỗi, database rollback và source trên filesystem được dọn dẹp.

#### AnalysisService
**Trách nhiệm:** Phân tích source code Java Spring Boot bằng JavaParser

**Chức năng:**
- Parse production file Java theo hướng best-effort; nếu một số file không parse được thì thống kê riêng và tiếp tục với phần còn lại
- Source discovery hỗ trợ multi-module; production analysis chính dùng `src/main/java`
- Existing tests trong `src/test/java` được đọc bởi luồng Existing Test Analysis riêng; không tính vào production counters nhưng được đưa vào AI context để cải thiện/bổ sung test
- Bỏ qua build/generated directories (`target`, `build`, `generated-sources`, `.gradle`...)
- `extractClasses()` — Lấy danh sách class
- Hỗ trợ class/interface/enum/record và lưu fully-qualified name
- `extractMethods()` — Lấy danh sách method, params, return type
- `extractEndpoints()` — Lấy REST endpoint (`@GetMapping`, `@PostMapping`...)
- Expand đầy đủ nhiều path × nhiều HTTP method; `ANY` cho unrestricted `@RequestMapping`
- `extractServiceRepositoryRelations()` — Tìm quan hệ Service gọi Repository
- Dùng method signature và fully-qualified class name để tránh liên kết sai khi overload/trùng tên
- `AnalysisManifestService` — export manifest deterministic và validation missing/unexpected với ground truth

#### AIAgentService
**Trách nhiệm:** Giao tiếp với LLM API và xử lý pipeline AI

**Chức năng:**
- `GenerationContextBuilder` gom Project + Static Analysis manifest + relation + Existing Test + Business Rule thành JSON-ready context theo từng stage trước khi gọi prompt/LLM.
- `PromptManager`, `LlmClient`/`MockLlmClient` và `GenerationResponseParser` tách prompt, gọi LLM và parse JSON output khỏi business service.
- `generateBusinessRules()` — AI auto sinh Business Rule từ source code khi user chọn chế độ AI sinh
- `reviewBusinessRules()` — AI review Business Rule user nhập và đề xuất sửa/bổ sung
- `generateTestPlan()` — Sinh Test Plan từ Business Rule + Source Code
- `generateTestCases()` — Sinh Test Case từ Test Plan
- `generateUnitTests()` — Sinh code Unit Test (JUnit 5 + Mockito)
- Khi có existing tests, AI Agent ưu tiên cải thiện/bổ sung test hiện có thay vì luôn sinh file mới

**Nguyên tắc:**
- Mỗi lần gọi LLM phải có prompt template riêng, lưu trong file `.txt` hoặc `.md`
- Parse output LLM bằng JSON schema, không dùng regex
- Retry tối đa 2 lần nếu LLM trả về sai format

#### TraceabilityService
**Trách nhiệm:** Xây dựng và truy vấn Traceability Matrix

**Chức năng:**
- `linkBusinessRuleToTestPlan()`
- `linkTestPlanToTestCase()`
- `linkTestCaseToUnitTest()`
- `getMatrixForProject(projectId)` — Trả về toàn bộ matrix dưới dạng JSON
- `findUncoveredRules(projectId)` — Tìm Business Rule chưa có test

#### CoverageService
**Trách nhiệm:** Phân tích coverage và phát hiện gap

**Chức năng:**
- `parseJacocoXml(file)` — Parse file XML JaCoCo do user upload
- `calculateRequirementCoverage(projectId)` — Tính % Business Rule có test
- `detectCoverageGap(projectId)` — Tìm method chưa được test cover đầy đủ
- `suggestAdditionalTestCases(gap)` — Gọi AI sinh thêm test case cho phần gap

#### AuthService
**Trách nhiệm:** Đăng ký, đăng nhập và phân quyền cơ bản.

**Chức năng:**
- `register()` — Tạo user mới với role mặc định `USER`
- `login()` — Xác thực email/password, trả token/session cho frontend
- `getCurrentUser()` — Lấy user hiện tại từ security context
- `requireProjectAccess(projectId)` — Cho phép owner hoặc `ADMIN` truy cập project

#### ExistingTestService
**Trách nhiệm:** Đọc và tóm tắt test có sẵn để phục vụ test generation.

**Chức năng:**
- Quét `src/test/java` và lưu danh sách file test có sẵn.
- Trích xuất test class/test method/import/assertion/mocking pattern ở mức cần thiết.
- Liên kết mềm existing test với production class/method khi có đủ tín hiệu.
- Cung cấp Existing Test Context cho `AIAgentService`.

### 3. Repository Layer
- Spring Data JPA interfaces
- Mỗi entity có một Repository tương ứng
- Custom query dùng `@Query` khi cần

## Module riêng biệt

Bên cạnh các service trên, có một số module helper:

- `LLMClient` — Wrapper gọi Mock/OpenAI/Google Gemini API
- `PromptManager` — Quản lý prompt template
- `FileStorageService` — Quản lý lưu trữ file ZIP, JaCoCo XML
- `GithubService` — Clone public GitHub repo

## Nguyên tắc thiết kế

### Single Responsibility
Mỗi class chỉ có một lý do để thay đổi. Service không trộn lẫn trách nhiệm.

### Dependency Injection
Sử dụng constructor injection của Spring, KHÔNG dùng `@Autowired` trên field.

```java
// Đúng
@Service
public class AnalysisService {
    private final JavaParserHelper parser;

    public AnalysisService(JavaParserHelper parser) {
        this.parser = parser;
    }
}

// Sai - không dùng
@Autowired
private JavaParserHelper parser;
```

### DTO vs Entity
- Entity dùng để map với database (JPA annotations)
- DTO dùng để giao tiếp với Frontend
- KHÔNG trả Entity trực tiếp về Frontend
- Dùng MapStruct hoặc tự viết mapper

### Async cho LLM calls
Gọi LLM mất thời gian (vài giây đến vài chục giây). Cần xử lý async:
- Sử dụng `@Async` hoặc `CompletableFuture`
- Frontend polling hoặc WebSocket để nhận kết quả
- Lưu trạng thái job vào database (`PENDING`, `RUNNING`, `DONE`, `FAILED`)

### Error Handling
- Sử dụng `@ControllerAdvice` để xử lý exception tập trung
- Custom exception cho từng loại lỗi nghiệp vụ
- Trả về error response chuẩn: `{ code, message, details }`

### Security
- Dùng Spring Security cho authentication/authorization.
- API upload/analysis/generation/export yêu cầu đăng nhập.
- `USER` chỉ thao tác với project mình sở hữu; `ADMIN` có quyền xem và quản lý toàn hệ thống.
- Password phải hash bằng BCrypt; không lưu password plain text.
- Token/session không được hard-code trong frontend hoặc backend.

## Frontend Architecture

Frontend đơn giản hơn, chia theo feature:

```
src/
├── features/
│   ├── auth/            # Login/register/current user
│   ├── projects/        # Upload, list project
│   ├── business-rules/  # Review/edit business rule
│   ├── test-plans/      # Review test plan
│   ├── test-cases/      # Review test case
│   ├── unit-tests/      # Xem unit test sinh ra
│   ├── traceability/    # Hiển thị matrix
│   └── coverage/        # Upload JaCoCo, hiển thị coverage
├── shared/
│   ├── api/             # API client (axios)
│   ├── components/      # UI components dùng chung
│   └── hooks/           # Custom React hooks
└── App.tsx
```

**State management:** TanStack Query cho server state, không cần Redux/Zustand (đơn giản hơn cho 2 người).
