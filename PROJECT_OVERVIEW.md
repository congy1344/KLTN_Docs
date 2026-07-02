# PROJECT_OVERVIEW.md — Tổng quan đề tài

## Tên đề tài

**Nghiên cứu và xây dựng hệ thống AI QA Agent theo hướng Grey-box hỗ trợ sinh Test Plan, Test Case và Unit Test cho dự án Java Spring Boot (GreyTest)**

## Đội ngũ

2 sinh viên ngành Kỹ thuật phần mềm.

## Vấn đề đang giải quyết

Việc viết test case và unit test thủ công cho dự án Java Spring Boot tốn rất nhiều thời gian. Các phương pháp tự động hiện tại chia làm 2 hướng:

- **White-box (chỉ đọc code):** AI biết "code làm gì" nhưng không biết "tại sao làm vậy" → thiếu test case về mặt nghiệp vụ
- **Black-box (chỉ đọc requirement):** AI biết yêu cầu nhưng không hiểu cấu trúc code → test case không bám sát implementation

**Đề xuất:** Kết hợp cả hai (Grey-box) thông qua AI Agent có Human-in-the-Loop.

## Mục tiêu tổng quát

Đề xuất và xây dựng hệ thống AI QA Agent theo hướng Grey-box hỗ trợ sinh tự động Test Plan, Test Case và Unit Test cho dự án Java Spring Boot, đồng thời nâng cao Requirement Coverage và Code Coverage thông qua quy trình Human-in-the-Loop. Nếu project đầu vào đã có unit test, hệ thống không bỏ qua hoàn toàn mà đọc các test đó như Existing Test Context để phân tích, cải thiện và bổ sung test mới.

## Mục tiêu cụ thể

### Phần nghiên cứu (lý thuyết)

- Nghiên cứu các phương pháp sinh kiểm thử phần mềm tự động sử dụng Large Language Models (LLMs)
- Nghiên cứu các kỹ thuật phân tích tĩnh (Static Analysis) đối với ứng dụng Java Spring Boot
- Nghiên cứu phương pháp kết hợp White-box (Source Code) và Black-box (Business Rules)
- Đề xuất mô hình Grey-box AI QA Agent
- Đề xuất mô hình Human-in-the-Loop
- Đề xuất mô hình Traceability Matrix tự động giữa: Business Rule → Test Plan → Test Case → Unit Test
- Đề xuất phương pháp đo lường Requirement Coverage
- Xây dựng bộ tiêu chí đánh giá hiệu quả

### Phần hệ thống (thực hành)

#### Static Analysis Engine (đã thu gọn)
Sử dụng JavaParser để phân tích mã nguồn Java Spring Boot. Tiếp nhận source code từ:
- File ZIP
- GitHub Repository Public

Static Analysis Engine cấu hình JavaParser ở language level `JAVA_21`, hướng tới project Java Spring Boot dùng source
Java 8-21. Các cú pháp mới hơn Java 21 hoặc parser edge case không được cam kết hỗ trợ đầy đủ.

Production source (`src/main/java`) vẫn là nguồn chính để trích xuất class, method, endpoint và relation.
Test có sẵn (`src/test/java`) được đọc trong luồng riêng **Existing Test Analysis**: hệ thống thống kê file test, trích xuất test class/test method/import/assertion/mocking pattern ở mức cần thiết và dùng làm context để cải thiện hoặc bổ sung unit test. Existing tests không làm tăng chỉ số production Classes/Methods/Endpoints, nhưng không bị loại khỏi các chức năng AI phía sau.
Nếu một số production file không parse được do giới hạn JavaParser, hệ thống bỏ qua file đó, thống kê
`failedParseFiles/failedParseFilePaths` và vẫn phân tích phần source còn lại.

**6 extractor cốt lõi**:
- `extract_classes` — Trích xuất class
- `extract_methods` — Trích xuất method
- `extract_relevant_annotations` — Trích xuất annotation chọn lọc phục vụ phân loại, validation, security và test context
- `extract_endpoints` — Trích xuất REST endpoint
- `extract_controller_service_relations` — Trích xuất quan hệ Controller Method → Service Method khi resolve chắc chắn
- `extract_service_repository_relations` — Trích xuất quan hệ Service-Repository

Các extractor sử dụng fully-qualified type name và method signature làm identity. Class extractor hỗ trợ
class/interface/enum/record; endpoint extractor hỗ trợ nhiều path và nhiều HTTP method. Annotation extractor không lưu
mọi annotation mà chỉ giữ annotation có ý nghĩa cho test generation context. Relation extractor ưu tiên độ chính xác:
nếu dependency hoặc method call mơ hồ thì bỏ qua thay vì tạo quan hệ sai. Kết quả có thể export thành Analysis Manifest
và validate với ground truth theo từng phần tử missing/unexpected.

#### Business Rule Management
Module quản lý Business Rule cho từng Service Method. Người dùng chọn một trong hai hướng:

1. **AI auto sinh Business Rule** từ Static Analysis Context, source method, annotation, relation và existing tests nếu có.
2. **User nhập Business Rule trước**, sau đó AI review, phát hiện rule mơ hồ/trùng/lệch method và đề xuất chỉnh sửa hoặc bổ sung rule còn thiếu.

Cả hai hướng đều tạo danh sách Business Rule ở trạng thái chờ duyệt. Người dùng phải review/sửa/xóa/approve trước khi GreyTest sinh Test Plan.

#### AI QA Agent (đã thu gọn còn 3 chức năng)
- `generate_test_plan` — Sinh Test Plan
- `generate_test_cases` — Sinh Test Case
- `generate_unit_tests` — Sinh mới hoặc cải thiện Unit Test (JUnit 5 + Mockito), có xét đến test sẵn nếu project đã có `src/test/java`

**Lưu ý:** AI không tự quyết định Business Rule cuối cùng. AI chỉ sinh hoặc review/gợi ý; người dùng duyệt mới được chuyển sang bước sinh Test Plan.

#### Phân loại Test Plan
- Happy Path — Trường hợp dùng đúng, dữ liệu hợp lệ
- Boundary Case — Giá trị tại ranh giới
- Exception Case — Dữ liệu sai, điều kiện không thỏa
- Edge Case — Tình huống bất thường, kết hợp nhiều điều kiện lạ

#### Cấu trúc Test Case
Mỗi test case sinh ra phải có đủ 8 trường:

| Trường | Ý nghĩa |
|---|---|
| Test ID | Mã định danh duy nhất (TC-001, TC-002...) |
| Test Type | Happy Path / Boundary / Exception / Edge |
| Description | Mô tả ngắn test kiểm tra gì |
| Preconditions | Điều kiện đúng trước khi chạy test |
| Test Data | Dữ liệu đầu vào cụ thể |
| Expected Result | Kết quả mong đợi |
| Priority | High / Medium / Low |
| Trace Source | Liên kết về Business Rule / Test Plan |

#### Human-in-the-Loop
Cơ chế cho phép người dùng review và phê duyệt:
- Review Business Rule (AI auto sinh hoặc user nhập trước, người dùng sửa/thêm/xóa)
- Review gợi ý AI cho Business Rule do user nhập
- Review Test Plan
- Review Test Case
- Phê duyệt trước khi chuyển sang bước tiếp theo

#### Traceability Matrix
Bảng liên kết tự động:
- Business Rule → Test Plan
- Test Plan → Test Case
- Test Case → Unit Test

#### Coverage Module
- **Requirement Coverage Analyzer** — Tính % Business Rule có test case tương ứng
- **JaCoCo Coverage Analyzer** — User tự chạy JaCoCo ở máy, upload file XML báo cáo lên web, hệ thống parse và hiển thị (KHÔNG tự động chạy JaCoCo trên server)
- **Coverage Gap Detection** — Phát hiện code chưa được test, đề xuất bổ sung

#### Existing Test Improvement
- Phát hiện và đọc các unit test có sẵn trong `src/test/java`.
- Đối chiếu test có sẵn với production method, Business Rule, Test Plan và Test Case.
- Khi sinh Unit Test, ưu tiên cải thiện test hiện có nếu có class/method test tương ứng; nếu chưa có thì sinh file test mới.
- Output cần đánh dấu rõ: `NEW_TEST`, `IMPROVE_EXISTING_TEST` hoặc `SUPPLEMENT_EXISTING_TEST`.

#### Authentication & Authorization
- Có chức năng đăng ký/đăng nhập cơ bản cho người dùng.
- Dùng phân quyền theo vai trò:
  - `USER`: upload/clone project, xem và xử lý pipeline test cho project của mình.
  - `ADMIN`: quản lý user, xem toàn bộ project và hỗ trợ xử lý dữ liệu hệ thống.
- Mỗi project gắn với user sở hữu; user thường không truy cập/sửa project của người khác.

#### Xuất báo cáo
- JSON
- Markdown

(Đã loại bỏ PDF và Excel để tiết kiệm thời gian)

## Đã loại bỏ khỏi phạm vi ban đầu

Để vừa sức 2 sinh viên, các phần sau đã được loại bỏ:

| Phần | Lý do |
|---|---|
| Mutation Score (PITest) | Setup phức tạp, thời gian chạy dài |
| Xuất PDF/Excel | Chuyển sang JSON/Markdown |
| Tự động chạy JaCoCo trên server | User upload XML báo cáo thay vì server chạy |
| Dataset 8 dự án | Giảm còn 3–4 dự án nhỏ và trung bình |

## Phạm vi không thuộc đề tài

- KHÔNG hỗ trợ ngôn ngữ khác ngoài Java
- KHÔNG hỗ trợ framework khác ngoài Spring Boot
- KHÔNG hỗ trợ integration test, e2e test — chỉ unit test
- KHÔNG có hệ thống auth phức tạp kiểu enterprise: không OAuth2/social login, không SSO, không multi-tenant nâng cao. Chỉ có đăng nhập và phân quyền cơ bản `USER`/`ADMIN`.
- KHÔNG có tính năng billing, payment, multi-tenant
