# Tài liệu phân tích yêu cầu GreyTest

**1\. Tổng quan tài liệu**

**1.1. Mục đích**

Tài liệu này mô tả các yêu cầu của hệ thống GreyTest ở mức phân tích yêu cầu, làm căn cứ cho thiết kế, cài đặt, kiểm thử và đánh giá thực nghiệm trong đồ án tốt nghiệp.

**1.2. Bối cảnh bài toán**

GreyTest là hệ thống AI QA Agent theo hướng grey-box, hỗ trợ phân tích mã nguồn Java Spring Boot để sinh Business Rule, Test Plan, Test Case và Unit Test. Hệ thống không thay thế hoàn toàn kỹ sư QA hoặc lập trình viên, mà đóng vai trò trợ lý sinh/gợi ý tạo tác kiểm thử có sự kiểm duyệt của con người.

**1.3. Phạm vi hệ thống**

- Cho phép người dùng đưa mã nguồn Java Spring Boot vào hệ thống bằng file ZIP hoặc GitHub URL public.
- Phân tích tĩnh mã nguồn production và test hiện có để tạo ngữ cảnh cho AI.
- Hỗ trợ quy trình Human-in-the-Loop: Business Rule -> Test Plan -> Test Case -> Unit Test.
- Cho phép truy vết tạo tác kiểm thử, upload báo cáo JaCoCo XML và xuất báo cáo JSON/Markdown.
- Cung cấp xác thực, phân quyền cơ bản cho User và Admin.

**1.4. Ngoài phạm vi**

- Không hỗ trợ ngôn ngữ/framework ngoài Java Spring Boot.
- Không tự động chạy build, test hoặc coverage trên server.
- Không tự động commit, merge hoặc tạo pull request vào project gốc.
- Không sinh integration test, end-to-end test, performance test hoặc security test.
- Không hướng tới sản phẩm thương mại nhiều tenant, billing, SSO hoặc phân quyền team phức tạp.

**1.5. Giả định và phụ thuộc**

- Project đầu vào có cấu trúc phổ biến của Java Spring Boot, ưu tiên `src/main/java` và `src/test/java`.
- Người dùng chịu trách nhiệm kiểm chứng lại Business Rule, Test Case và Unit Test do AI đề xuất.
- Người dùng tự chạy test/coverage ở môi trường local rồi upload file `jacoco.xml`.
- Chất lượng kết quả phụ thuộc vào chất lượng source code, context phân tích tĩnh, prompt và model LLM được cấu hình.

**1.6. Thuật ngữ**

| **Thuật ngữ** | **Ý nghĩa** |
| ------------- | ----------- |
| Grey-box | Cách tiếp cận kết hợp thông tin nội bộ từ source code với mục tiêu kiểm thử ở mức hành vi. |
| HITL | Human-in-the-Loop, nghĩa là người dùng review/chỉnh sửa/phê duyệt kết quả AI trước khi đi tiếp. |
| Business Rule | Luật nghiệp vụ hoặc điều kiện xử lý quan trọng được rút ra từ code hoặc do người dùng nhập. |
| Test Plan | Ý tưởng/nhóm kiểm thử ở mức kế hoạch, được sinh từ Business Rule. |
| Test Case | Kịch bản kiểm thử cụ thể có dữ liệu đầu vào, điều kiện và kết quả mong đợi. |
| Unit Test | Mã kiểm thử tự động ở mức đơn vị, dùng JUnit 5 và Mockito. |
| Traceability Matrix | Ma trận truy vết liên kết Business Rule -> Test Plan -> Test Case -> Unit Test. |
| Manifest | Báo cáo JSON mô tả cấu trúc source code được trích xuất bởi static analysis. |

**2\. Phân định vai trò người dùng (Actors)**

Hệ thống GreyTest phục vụ 3 nhóm tác nhân chính:

- **Người dùng (User / Lập trình viên / Kỹ sư QA):** Tải lên mã nguồn, trực tiếp tham gia vào quy trình Human-in-the-Loop (HITL) để thêm/sửa/xóa/duyệt các Luật nghiệp vụ, Kế hoạch kiểm thử, Kịch bản kiểm thử, xuất mã nguồn Unit Test và tải lên báo cáo độ bao phủ.
- **Quản trị viên (Admin):** Quản lý người dùng, có quyền xem toàn bộ dự án trên hệ thống để phục vụ việc bảo trì và kiểm tra dữ liệu thực nghiệm.
- **Hệ thống tự động (System / AI QA Agent):** Đóng vai trò là tác nhân chạy ngầm, thực hiện phân tích tĩnh và giao tiếp với mô hình ngôn ngữ lớn (LLM) để sinh/gợi ý/phân tích các tạo tác kiểm thử (Test Artifacts).

**3\. Yêu cầu chức năng (Functional Requirements)**

Quy ước mức ưu tiên: **Must** là bắt buộc trong phạm vi đồ án; **Should** là nên có để hoàn thiện quy trình; **Could** là có thể bổ sung nếu còn thời gian.

**3.1. Phân hệ Quản lý Dự án & Phân tích Tĩnh**

| **Mã YC** | **Tên chức năng** | **Mô tả yêu cầu** | **Ưu tiên** | **Tiêu chí chấp nhận** |
| --------- | ----------------- | ----------------- | ----------- | ---------------------- |
| **FR1.1** | **Nhập mã nguồn dự án** | Hỗ trợ upload mã nguồn qua file ZIP hoặc clone từ GitHub URL public. Hệ thống kiểm tra project có dấu hiệu là Java Spring Boot và gán quyền sở hữu cho User tải lên. | Must | Với ZIP/GitHub URL hợp lệ, hệ thống tạo project mới và gán owner. Với project không hợp lệ, hệ thống trả lỗi rõ ràng và không tạo dữ liệu rác. |
| **FR1.2** | **Phân tích tĩnh source code** | Hệ thống phân tích source production để trích xuất type, method, annotation, REST endpoint và quan hệ Controller-Service/Service-Repository. | Must | Sau khi phân tích thành công, project có dữ liệu class/method/endpoint/relation để dùng làm context cho các bước AI. |
| **FR1.3** | **Phân tích best-effort** | Nếu một phần source code bị lỗi cú pháp, hệ thống ghi nhận file lỗi và tiếp tục phân tích phần còn lại khi còn source hợp lệ. | Must | File lỗi được thống kê; pipeline không dừng toàn bộ nếu vẫn còn file hợp lệ để phân tích. |
| **FR1.4** | **Phân tích Unit Test có sẵn** | Hệ thống quét test hiện có để trích xuất metadata cơ bản và dùng làm ngữ cảnh tránh sinh trùng hoặc đề xuất cải thiện test cũ. | Should | Nếu có `src/test/java`, hệ thống lưu được test class, test method, import/assertion/mock ở mức metadata. Nếu không có test cũ, pipeline vẫn tiếp tục. |
| **FR1.5** | **Xuất và đối chiếu Manifest** | Cho phép xuất báo cáo cấu trúc trích xuất tĩnh dạng JSON và đối chiếu với Ground Truth phục vụ đánh giá thực nghiệm. | Should | Người dùng xuất được Manifest JSON; API validate trả thống kê exact match, missing và unexpected khi có Ground Truth. |

**3.2. Phân hệ Quản lý Luật nghiệp vụ (Business Rules - HITL)**

| **Mã YC** | **Tên chức năng** | **Mô tả yêu cầu** | **Ưu tiên** | **Tiêu chí chấp nhận** |
| --------- | ----------------- | ----------------- | ----------- | ---------------------- |
| **FR2.1** | **AI tự động sinh Rule** | Dựa trên context của method/service, AI đề xuất Business Rule liên quan đến validation, điều kiện xử lý và side effect. | Must | Người dùng chọn method/phạm vi phù hợp và nhận được danh sách rule ở trạng thái chờ review. |
| **FR2.2** | **AI review Rule người dùng nhập** | Người dùng nhập rule thủ công; AI nhận xét rule và đề xuất chỉnh sửa hoặc bổ sung rule còn thiếu nếu có tín hiệu từ source code. | Should | Hệ thống trả về nhận xét hoặc gợi ý bổ sung; rule gốc của người dùng không bị tự động ghi đè khi chưa xác nhận. |
| **FR2.3** | **Tương tác HITL và phê duyệt** | Người dùng có thể xem, thêm, sửa, xóa và phê duyệt danh sách Business Rule. | Must | Chỉ Business Rule đã được phê duyệt mới được dùng để sinh Test Plan. |

**3.3. Phân hệ Test Plan & Test Case - HITL**

| **Mã YC** | **Tên chức năng** | **Mô tả yêu cầu** | **Ưu tiên** | **Tiêu chí chấp nhận** |
| --------- | ----------------- | ----------------- | ----------- | ---------------------- |
| **FR3.1** | **AI sinh Test Plan** | Từ Business Rule đã duyệt, AI sinh Test Plan theo các nhóm phù hợp như happy path, boundary, exception và edge. | Must | Mỗi Test Plan được liên kết với Business Rule nguồn và có trạng thái chờ duyệt trước khi dùng để sinh Test Case. |
| **FR3.2** | **AI sinh Test Case chi tiết** | Từ Test Plan đã duyệt, AI sinh Test Case có đủ thông tin cần thiết để người dùng review và làm đầu vào sinh Unit Test. | Must | Test Case có Test ID, Test Type, Description, Preconditions, Test Data, Expected Result, Priority và Trace Source. |
| **FR3.3** | **Tái tạo dữ liệu** | Khi người dùng regenerate ở một pha, hệ thống yêu cầu xác nhận và xử lý dữ liệu các pha sau để tránh trạng thái không nhất quán. | Should | Regenerate rule/plan/case không để lại dữ liệu con mồ côi trái với project state hiện tại. |

**3.4. Phân hệ Sinh Unit Test**

| **Mã YC** | **Tên chức năng** | **Mô tả yêu cầu** | **Ưu tiên** | **Tiêu chí chấp nhận** |
| --------- | ----------------- | ----------------- | ----------- | ---------------------- |
| **FR4.1** | **AI sinh mã Unit Test** | Dựa trên Test Case đã duyệt, AI sinh mã Unit Test Java và phân loại kết quả là test mới, cải thiện test cũ hoặc bổ sung method vào test cũ. | Must | Unit Test được lưu kèm nội dung code, package/file path và loại sinh tương ứng. |
| **FR4.2** | **Xuất mã nguồn Unit Test** | Hệ thống đóng gói các file test sinh ra theo cấu trúc thư mục phù hợp để người dùng tải về. | Must | Người dùng tải được ZIP chứa file test theo cấu trúc `src/test/java/.../*Test.java`. |

**3.5. Phân hệ Traceability, Coverage & Báo cáo**

| **Mã YC** | **Tên chức năng** | **Mô tả yêu cầu** | **Ưu tiên** | **Tiêu chí chấp nhận** |
| --------- | ----------------- | ----------------- | ----------- | ---------------------- |
| **FR5.1** | **Ma trận truy vết** | Hiển thị chuỗi liên kết Business Rule -> Test Plan -> Test Case -> Unit Test, kèm method liên quan khi có. | Must | Người dùng xem được liên kết hiện có và nhận biết phần còn thiếu trong chuỗi truy vết. |
| **FR5.2** | **Upload coverage và phát hiện gap** | Người dùng upload file JaCoCo XML; hệ thống tính coverage và đề xuất bổ sung test cho vùng có độ phủ thấp khi có đủ dữ liệu. | Should | File JaCoCo hợp lệ được parse và lưu; hệ thống hiển thị line/branch coverage và gap nếu phát hiện được. |
| **FR5.3** | **Xuất báo cáo JSON/Markdown** | Xuất dữ liệu dự án ở định dạng JSON cho máy đọc hoặc Markdown cho người đọc. | Should | Người dùng tải được báo cáo chứa analysis, rule, plan, case, unit test, traceability và coverage nếu có. |

**3.6. Phân hệ Xác thực & Phân quyền**

| **Mã YC** | **Tên chức năng** | **Mô tả yêu cầu** | **Ưu tiên** | **Tiêu chí chấp nhận** |
| --------- | ----------------- | ----------------- | ----------- | ---------------------- |
| **FR6.1** | **Đăng ký và đăng nhập** | Cho phép người dùng đăng ký tài khoản, đăng nhập và lấy thông tin người dùng hiện tại. | Must | Người dùng đăng nhập thành công với tài khoản hợp lệ; mật khẩu không lưu dạng plaintext. |
| **FR6.2** | **Phân quyền theo vai trò** | Hỗ trợ hai vai trò cơ bản: USER và ADMIN. USER thao tác với project của mình; ADMIN có quyền xem toàn bộ project. | Must | USER không xem/sửa project của người khác; ADMIN truy cập được dữ liệu cần kiểm tra. |
| **FR6.3** | **Kiểm soát truy cập project/API** | Các API liên quan đến project phải kiểm tra quyền truy cập trước khi xử lý yêu cầu. | Must | Request không có quyền bị từ chối; request hợp lệ tiếp tục xử lý đúng chức năng. |

**4\. Yêu cầu phi chức năng (Non-Functional Requirements)**

| **Mã NFR** | **Nhóm yêu cầu** | **Mô tả** | **Tiêu chí kiểm chứng** |
| ---------- | ---------------- | --------- | ----------------------- |
| **NFR1** | Hiệu năng & bất đồng bộ | Các tác vụ gọi LLM hoặc phân tích source có thể chạy bất đồng bộ để không khóa giao diện người dùng. | Người dùng thấy trạng thái xử lý như `PENDING`, `RUNNING`, `DONE`, `FAILED`; tác vụ lỗi không làm treo toàn bộ hệ thống. |
| **NFR2** | Độ tin cậy khi dùng LLM | Hệ thống cần xử lý trường hợp LLM trả sai định dạng, không có text output hoặc sinh dữ liệu thiếu trường bắt buộc. | Lỗi LLM được ghi nhận và trả thông báo rõ; hệ thống có retry có giới hạn, không lưu kết quả sai schema làm dữ liệu chính thức. |
| **NFR3** | Kiểm soát chi phí thực nghiệm | Mỗi lần gọi LLM cần lưu được thông tin model/provider và token/cost nếu provider trả về dữ liệu này. | Báo cáo thực nghiệm có thể thống kê số lần gọi LLM, token đầu vào/đầu ra và chi phí ước tính khi có dữ liệu. |
| **NFR4** | Bảo mật & phân quyền | Dữ liệu project phải gắn với owner; mật khẩu không lưu plaintext; API project phải kiểm tra quyền trước khi xử lý. | User không truy cập được project của user khác; password trong database là hash; request thiếu/sai quyền trả lỗi phù hợp. |
| **NFR5** | Tính toàn vẹn dữ liệu | Khi upload, parse, regenerate hoặc rollback lỗi, hệ thống không để lại trạng thái project mâu thuẫn. | Project state phản ánh đúng bước hiện tại; dữ liệu pha sau bị xóa/khóa khi pha trước được regenerate. |
| **NFR6** | Khả dụng giao diện | Giao diện cần thể hiện rõ trạng thái dữ liệu, trạng thái review và lỗi để người dùng tiếp tục workflow. | Người dùng biết artifact nào đang chờ duyệt, đã duyệt, lỗi hoặc chưa được sinh. |

**5\. Ràng buộc kỹ thuật**

- Backend sử dụng Spring Boot; frontend sử dụng React/TypeScript theo kiến trúc hiện tại của dự án.
- Static Analysis Engine sử dụng JavaParser và ưu tiên phân tích source Java 8-21.
- Unit Test sinh ra hướng tới JUnit 5 và Mockito.
- Coverage sử dụng file JaCoCo XML do người dùng upload.
- Frontend có thể dùng TanStack Query để đồng bộ dữ liệu theo trạng thái project.
- Provider LLM phải có khả năng cấu hình để chuyển đổi giữa mock, Google Gemini và OpenAI khi cần.

**6\. Giới hạn của hệ thống**

- **Giới hạn về phạm vi dự án đầu vào:** Hệ thống chỉ hướng tới dự án **Java Spring Boot** có cấu trúc phổ biến (`src/main/java`, `src/test/java`) và sử dụng Maven/Gradle. Các ngôn ngữ hoặc framework khác như Kotlin, Groovy, Quarkus, Micronaut, Node.js, .NET không thuộc phạm vi đề tài. Với dự án multi-module, hệ thống xử lý theo hướng best-effort và chỉ đảm bảo phân tích các module có cấu trúc source chuẩn.
- **Giới hạn về phiên bản Java và khả năng parse:** Static Analysis Engine cấu hình JavaParser ở language level `JAVA_21`, phù hợp với source Java 8-21. Các cú pháp mới hơn Java 21, mã nguồn lỗi cú pháp, generated source hoặc cách viết quá đặc thù có thể không parse được. Khi gặp lỗi ở một số file production, hệ thống bỏ qua file đó, ghi nhận `failedParseFiles/failedParseFilePaths` và tiếp tục phân tích phần còn lại thay vì dừng toàn bộ pipeline.
- **Không hỗ trợ phân tích hành vi runtime:** Hệ thống phân tích mã nguồn tĩnh nên không thể phát hiện đầy đủ các hành vi chỉ sinh ra lúc chạy như endpoint đăng ký động qua `RequestMappingHandlerMapping`, reflection, dynamic proxy, AOP advice, cấu hình conditional bean, runtime configuration hoặc code được sinh bởi annotation processor/thư viện build.
- **Giới hạn đồ thị quan hệ và luồng nghiệp vụ:** Extractor chỉ nhận diện quan hệ chắc chắn ở mức gần: Controller Method → Service Method và Service → Repository. Hệ thống chưa xây dựng method-level call graph nhiều tầng, chưa phân tích luồng dữ liệu sâu, transaction boundary phức tạp, event/listener async, message queue hoặc nghiệp vụ trải qua nhiều service gọi chéo.
- **Hạn chế về annotation và mapping tùy chỉnh:** Hệ thống ưu tiên các Spring annotation phổ biến như `@RestController`, `@RequestMapping`, `@GetMapping`, `@PostMapping`, validation annotation và một số annotation bảo mật/giao dịch liên quan. Các custom/composed meta-annotation, route được cấu hình ngoài source code, hoặc convention riêng của từng dự án có thể không được đưa vào Analysis Context.
- **Giới hạn khi suy luận Business Rule:** AI có thể đề xuất rule dựa trên code, annotation, relation và existing tests, nhưng không thể tự biết các yêu cầu nghiệp vụ không xuất hiện trong source hoặc tài liệu đầu vào. Vì vậy Business Rule do AI sinh/review chỉ là gợi ý; người dùng phải kiểm tra, chỉnh sửa và phê duyệt trước khi dùng cho Test Plan.
- **Giới hạn của Unit Test được sinh:** Hệ thống tập trung sinh/cải thiện **unit test** bằng JUnit 5 và Mockito. Không sinh integration test, end-to-end test, performance test, security test hoặc contract test. Code test sinh ra được thiết kế để đúng package/import và có cấu trúc AAA, nhưng không cam kết luôn compile/pass trên mọi dự án vì còn phụ thuộc dependency, cấu hình build, dữ liệu test, Spring context và cách dự án gốc tổ chức mã nguồn.
- **Không tự động sửa hoặc merge vào project gốc:** GreyTest chỉ xuất file test mới/cải thiện để người dùng tải xuống. Hệ thống không tự commit, không tự tạo pull request, không ghi đè trực tiếp vào repository gốc và không chịu trách nhiệm xử lý conflict khi người dùng merge test vào project thật.
- **Không tự động chạy build/test/coverage trên server:** Hệ thống không chạy `mvn test`, `gradle test` hoặc JaCoCo trong backend vì cần sandbox phức tạp và môi trường build của từng dự án có thể khác nhau. Người dùng phải copy test về máy local, chạy test/JaCoCo và upload file `jacoco.xml` lên hệ thống.
- **Giới hạn của Coverage Gap Detection:** Coverage module chỉ đọc báo cáo JaCoCo XML do người dùng upload và tập trung vào line coverage, branch coverage, requirement coverage. Hệ thống không đo mutation score, path coverage, condition coverage chi tiết hoặc chất lượng assertion. Gap detection dựa trên ngưỡng coverage và traceability nên chỉ mang tính hỗ trợ đề xuất thêm test, không thay thế đánh giá thủ công của QA.
- **Giới hạn về Existing Test Analysis:** Hệ thống chỉ trích xuất metadata cơ bản từ `src/test/java` như test class, test method, import, assertion và mocking pattern. Việc liên kết existing test với production method là suy luận mềm, có thể bỏ sót nếu dự án dùng naming convention lạ, test parameterized phức tạp, custom test framework hoặc helper nhiều tầng.
- **Giới hạn vận hành LLM:** Kết quả sinh bởi LLM phụ thuộc model, prompt, token limit và chất lượng context đầu vào. Với class/method quá dài hoặc project lớn, hệ thống có thể phải rút gọn context, làm giảm độ chính xác. Hệ thống có retry khi LLM trả sai JSON, nhưng không loại bỏ hoàn toàn rủi ro hallucination, thiếu case hoặc sinh test chưa phù hợp nghiệp vụ.
- **Giới hạn về bảo mật và phân quyền:** Hệ thống chỉ có đăng ký/đăng nhập và phân quyền cơ bản `USER`/`ADMIN`. Không hỗ trợ OAuth2, SSO, social login, phân quyền chi tiết theo team/project, audit log đầy đủ, billing/payment hoặc multi-tenant nâng cao.
- **Giới hạn về xuất báo cáo:** Hệ thống chỉ xuất báo cáo ở định dạng JSON và Markdown. Không xuất PDF, Excel hoặc dashboard BI nâng cao để giữ phạm vi phù hợp với đồ án 2 sinh viên.
- **Giới hạn thực nghiệm:** Việc đánh giá chỉ dự kiến thực hiện trên 3-4 dự án Java Spring Boot nhỏ và trung bình. Kết quả thực nghiệm phản ánh dataset, model LLM và cấu hình prompt tại thời điểm chạy, không khẳng định hệ thống hoạt động tốt tương đương trên mọi dự án enterprise lớn hoặc domain nghiệp vụ phức tạp.

**7\. Bảng Use Case**

**7.1. Danh sách Use Case tổng quan**

| **Mã UC** | **Tên Use Case** | **Actor chính** | **Tiền điều kiện** | **Kết quả sau khi hoàn tất** | **FR liên quan** |
| --------- | ---------------- | --------------- | ------------------ | ----------------------------- | ---------------- |
| **UC-01** | Đăng ký và đăng nhập | User, Admin | Người dùng chưa hoặc đã có tài khoản | Người dùng được xác thực và truy cập hệ thống theo đúng vai trò | FR6.1, FR6.2 |
| **UC-02** | Nhập mã nguồn dự án | User | User đã đăng nhập | Project được tạo, gán owner và sẵn sàng phân tích | FR1.1 |
| **UC-03** | Phân tích tĩnh source code | System | Project đã upload hoặc clone thành công | Hệ thống lưu class, method, annotation, endpoint và relation trích xuất được | FR1.2, FR1.3 |
| **UC-04** | Phân tích unit test có sẵn | System | Project có thư mục `src/test/java` | Existing Test Context được lưu để hỗ trợ sinh/cải thiện test | FR1.4 |
| **UC-05** | Tạo và duyệt Business Rule | User, AI QA Agent | Project đã phân tích xong | Danh sách Business Rule được user phê duyệt | FR2.1, FR2.2, FR2.3 |
| **UC-06** | Sinh và duyệt Test Plan | User, AI QA Agent | Business Rule đã được phê duyệt | Test Plan được sinh theo nhóm Happy/Boundary/Exception/Edge và được duyệt | FR3.1 |
| **UC-07** | Sinh và duyệt Test Case | User, AI QA Agent | Test Plan đã được phê duyệt | Test Case đủ 8 trường được sinh và được duyệt | FR3.2 |
| **UC-08** | Sinh và xuất Unit Test | User, AI QA Agent | Test Case đã được phê duyệt | Unit Test JUnit 5 + Mockito được sinh và đóng gói để tải xuống | FR4.1, FR4.2 |
| **UC-09** | Xem Traceability Matrix | User, Admin | Dự án đã có rule, plan, case hoặc unit test | Người dùng xem được chuỗi liên kết giữa các tạo tác kiểm thử | FR5.1 |
| **UC-10** | Upload JaCoCo và phát hiện gap | User, AI QA Agent | User đã chạy test và có file `jacoco.xml` | Hệ thống hiển thị coverage và đề xuất bổ sung test cho vùng chưa đủ độ phủ | FR5.2 |
| **UC-11** | Xuất báo cáo dự án | User, Admin | Dự án đã có dữ liệu cần báo cáo | Báo cáo JSON hoặc Markdown được xuất để lưu trữ/trình bày | FR5.3 |
| **UC-12** | Quản lý người dùng và dữ liệu hệ thống | Admin | Admin đã đăng nhập | Admin xem được toàn bộ project và hỗ trợ quản lý dữ liệu thực nghiệm | FR6.2, FR6.3 |

**7.2. Luồng chính và ngoại lệ rút gọn**

| **Mã UC** | **Luồng chính** | **Ngoại lệ/Ghi chú** |
| --------- | --------------- | -------------------- |
| **UC-01** | Người dùng nhập thông tin đăng ký hoặc đăng nhập. Hệ thống xác thực, tạo phiên/token và trả thông tin vai trò. | Sai email/mật khẩu hoặc tài khoản bị vô hiệu hóa thì hệ thống từ chối truy cập. |
| **UC-02** | User upload ZIP hoặc nhập GitHub URL. Hệ thống validate project Spring Boot, lưu source và tạo project. | File không hợp lệ, GitHub URL không public hoặc không tìm thấy `pom.xml`/`build.gradle` thì trả lỗi. |
| **UC-03** | System quét `src/main/java`, parse bằng JavaParser và lưu kết quả phân tích. | File parse lỗi được ghi nhận vào `failedParseFiles`; hệ thống tiếp tục nếu vẫn còn source hợp lệ. |
| **UC-04** | System quét `src/test/java`, đọc metadata test cũ và liên kết mềm với production code nếu đủ tín hiệu. | Nếu không có test cũ, hệ thống bỏ qua bước này và tiếp tục pipeline. |
| **UC-05** | User chọn AI sinh rule hoặc nhập rule trước để AI review. User sửa/thêm/xóa và bấm phê duyệt. | AI chỉ đưa gợi ý; rule chưa được duyệt không dùng để sinh Test Plan. |
| **UC-06** | AI sinh Test Plan từ Business Rule đã duyệt. User review, chỉnh sửa và phê duyệt. | Không bắt buộc mỗi rule có đủ cả 4 loại test. |
| **UC-07** | AI sinh Test Case từ Test Plan đã duyệt. User review, chỉnh sửa và phê duyệt. | Test Case thiếu trường bắt buộc thì không được chuyển sang bước sinh Unit Test. |
| **UC-08** | AI sinh Unit Test mới hoặc cải thiện test cũ, sau đó hệ thống export file theo cấu trúc package. | Code sinh ra cần user chạy lại trong project gốc để kiểm chứng compile/pass. |
| **UC-09** | User mở Traceability Matrix. Hệ thống hiển thị liên kết Business Rule → Test Plan → Test Case → Unit Test. | Các tạo tác chưa sinh xong sẽ hiển thị thiếu liên kết tương ứng. |
| **UC-10** | User upload `jacoco.xml`. Hệ thống parse coverage, phát hiện gap và gọi AI đề xuất thêm test case. | XML không đúng định dạng JaCoCo thì hệ thống báo lỗi và không lưu report. |
| **UC-11** | User chọn định dạng JSON hoặc Markdown. Hệ thống xuất toàn bộ dữ liệu dự án theo định dạng đã chọn. | Chỉ hỗ trợ JSON/Markdown, không hỗ trợ PDF/Excel. |
| **UC-12** | Admin đăng nhập, xem danh sách user/project và kiểm tra dữ liệu phục vụ bảo trì hoặc thực nghiệm. | Admin không thay thế bước review nghiệp vụ của owner project. |

**7.3. Đặc tả Use Case chi tiết theo SRS**

**UC-01. Đăng ký và đăng nhập**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Cho phép người dùng xác thực danh tính trước khi sử dụng GreyTest. |
| Actor chính | User, Admin |
| Actor phụ | System |
| Tiền điều kiện | Người dùng có email hợp lệ. Nếu đăng nhập thì tài khoản đã tồn tại và đang được kích hoạt. |
| Kích hoạt | Người dùng mở màn hình đăng ký hoặc đăng nhập. |
| Hậu điều kiện thành công | Người dùng có phiên đăng nhập hợp lệ và được gán vai trò `USER` hoặc `ADMIN`. |
| Hậu điều kiện thất bại | Người dùng không được cấp quyền truy cập vào các chức năng yêu cầu xác thực. |
| Dữ liệu vào | Email, mật khẩu, họ tên khi đăng ký. |
| Dữ liệu ra | Thông tin người dùng hiện tại, vai trò, token/session. |
| Yêu cầu liên quan | FR6.1, FR6.2 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | Người dùng nhập thông tin đăng ký hoặc đăng nhập. |
| 2 | Hệ thống kiểm tra định dạng dữ liệu đầu vào. |
| 3 | Với đăng ký, hệ thống băm mật khẩu bằng BCrypt và tạo tài khoản `USER`. |
| 4 | Với đăng nhập, hệ thống xác thực email và mật khẩu. |
| 5 | Hệ thống trả thông tin người dùng hiện tại và phiên/token đăng nhập. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | Email đã tồn tại khi đăng ký: hệ thống báo lỗi và không tạo tài khoản mới. |
| E2 | Sai email hoặc mật khẩu: hệ thống từ chối đăng nhập. |
| E3 | Tài khoản bị vô hiệu hóa: hệ thống từ chối truy cập. |

**UC-02. Nhập mã nguồn dự án**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Cho phép User đưa source code Java Spring Boot vào hệ thống để phân tích. |
| Actor chính | User |
| Actor phụ | System |
| Tiền điều kiện | User đã đăng nhập. |
| Kích hoạt | User chọn upload ZIP hoặc nhập GitHub URL public. |
| Hậu điều kiện thành công | Project được tạo, gán owner và có trạng thái ban đầu để phân tích. |
| Hậu điều kiện thất bại | Không tạo project hoặc rollback project đã tạo lỗi. |
| Dữ liệu vào | File ZIP hoặc GitHub URL. |
| Dữ liệu ra | Thông tin project, trạng thái xử lý. |
| Yêu cầu liên quan | FR1.1 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | User chọn phương thức nhập source code. |
| 2 | Hệ thống nhận ZIP hoặc clone repository GitHub public. |
| 3 | Hệ thống kiểm tra cấu trúc dự án Spring Boot thông qua `pom.xml` hoặc `build.gradle`. |
| 4 | Hệ thống lưu source vào thư mục lưu trữ/tạm có kiểm soát. |
| 5 | Hệ thống tạo project và gán `ownerUserId` là User hiện tại. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | File ZIP lỗi hoặc không đọc được: hệ thống báo lỗi upload. |
| E2 | GitHub URL không public hoặc clone thất bại: hệ thống báo lỗi nhập source. |
| E3 | Không tìm thấy `pom.xml` hoặc `build.gradle`: hệ thống báo project không hợp lệ. |

**UC-03. Phân tích tĩnh source code**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Trích xuất cấu trúc production code để tạo Static Analysis Context cho AI. |
| Actor chính | System |
| Actor phụ | User |
| Tiền điều kiện | Project đã được upload hoặc clone thành công. |
| Kích hoạt | Hệ thống tự động phân tích sau khi nhập source hoặc User yêu cầu phân tích lại. |
| Hậu điều kiện thành công | Class, method, annotation, endpoint và relation được lưu vào database. |
| Hậu điều kiện thất bại | Project chuyển trạng thái lỗi nếu không phân tích được source hợp lệ nào. |
| Dữ liệu vào | Source code trong `src/main/java`. |
| Dữ liệu ra | Static Analysis Context, manifest, thống kê file parse thành công/thất bại. |
| Yêu cầu liên quan | FR1.2, FR1.3, FR1.5 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | Hệ thống quét các source root `src/main/java`. |
| 2 | Hệ thống bỏ qua thư mục build/generated như `target`, `build`, `.gradle`. |
| 3 | JavaParser parse từng file production theo language level `JAVA_21`. |
| 4 | Hệ thống trích xuất type, method, annotation, endpoint và relation chắc chắn. |
| 5 | Hệ thống lưu kết quả và cập nhật trạng thái project thành `ANALYZED`. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | Một số file parse lỗi: hệ thống ghi `failedParseFiles` và tiếp tục. |
| E2 | Toàn bộ source không parse được: hệ thống rollback hoặc chuyển project sang `FAILED`. |
| E3 | User không có quyền truy cập project: hệ thống từ chối yêu cầu phân tích lại. |

**UC-04. Phân tích unit test có sẵn**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Đọc test hiện có để AI tránh sinh trùng và ưu tiên cải thiện/bổ sung test. |
| Actor chính | System |
| Actor phụ | AI QA Agent |
| Tiền điều kiện | Project có source hợp lệ; có thể có hoặc không có `src/test/java`. |
| Kích hoạt | Hệ thống chạy sau Static Analysis. |
| Hậu điều kiện thành công | Existing Test Context được lưu vào database. |
| Hậu điều kiện thất bại | Pipeline vẫn tiếp tục nhưng không có context test cũ. |
| Dữ liệu vào | Source test trong `src/test/java`. |
| Dữ liệu ra | Test class, test method, import, assertion, mock và liên kết mềm nếu có. |
| Yêu cầu liên quan | FR1.4 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | Hệ thống kiểm tra project có thư mục `src/test/java` hay không. |
| 2 | Hệ thống quét các file test Java. |
| 3 | Hệ thống trích xuất metadata test class, test method, import, assertion và mock. |
| 4 | Hệ thống suy luận class/method production liên quan nếu đủ tín hiệu. |
| 5 | Hệ thống lưu Existing Test Context để dùng trong các prompt AI. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | Không có `src/test/java`: hệ thống bỏ qua use case này. |
| E2 | Test dùng convention lạ: hệ thống lưu metadata cơ bản nhưng có thể không liên kết được method production. |

**UC-05. Tạo và duyệt Business Rule**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Tạo danh sách Business Rule đã được con người phê duyệt làm nền cho Test Plan. |
| Actor chính | User |
| Actor phụ | AI QA Agent, System |
| Tiền điều kiện | Project đã có Static Analysis Context. |
| Kích hoạt | User chọn AI sinh rule hoặc nhập rule để AI review. |
| Hậu điều kiện thành công | Business Rule có trạng thái `APPROVED`. |
| Hậu điều kiện thất bại | Rule chưa duyệt không được dùng cho bước sinh Test Plan. |
| Dữ liệu vào | Method context, source code liên quan, annotation/relation, existing tests, rule user nhập nếu có. |
| Dữ liệu ra | Danh sách Business Rule và review note/gợi ý nếu có. |
| Yêu cầu liên quan | FR2.1, FR2.2, FR2.3 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | User chọn chế độ AI auto sinh rule hoặc user nhập rule trước. |
| 2 | Hệ thống xây dựng context từ kết quả phân tích tĩnh và existing tests. |
| 3 | AI sinh rule hoặc review rule user nhập. |
| 4 | User xem, thêm, sửa, xóa hoặc chấp nhận gợi ý. |
| 5 | User phê duyệt danh sách rule cuối cùng. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | LLM trả sai JSON: hệ thống retry tối đa 2 lần. |
| E2 | User chưa phê duyệt rule: hệ thống không cho sinh Test Plan. |
| E3 | User regenerate rule: hệ thống yêu cầu xác nhận và xóa dữ liệu pha sau. |

**UC-06. Sinh và duyệt Test Plan**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Sinh Test Plan từ Business Rule đã duyệt theo các nhóm test phù hợp. |
| Actor chính | User |
| Actor phụ | AI QA Agent, System |
| Tiền điều kiện | Có ít nhất một Business Rule `APPROVED`. |
| Kích hoạt | User yêu cầu sinh Test Plan. |
| Hậu điều kiện thành công | Test Plan được User duyệt và liên kết với Business Rule. |
| Hậu điều kiện thất bại | Không tạo Test Plan hoặc Test Plan vẫn ở trạng thái chờ duyệt. |
| Dữ liệu vào | Business Rule, method context, existing tests liên quan. |
| Dữ liệu ra | Test Plan thuộc nhóm `HAPPY_PATH`, `BOUNDARY`, `EXCEPTION`, `EDGE`. |
| Yêu cầu liên quan | FR3.1, FR5.1 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | User bấm sinh Test Plan. |
| 2 | Hệ thống gửi Business Rule và context liên quan cho AI. |
| 3 | AI trả danh sách Test Plan. |
| 4 | Hệ thống lưu Test Plan ở trạng thái chờ duyệt. |
| 5 | User review, chỉnh sửa và phê duyệt Test Plan. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | Chưa có Business Rule đã duyệt: hệ thống không cho sinh Test Plan. |
| E2 | AI không sinh đủ 4 loại test: hệ thống vẫn chấp nhận nếu các loại sinh ra phù hợp rule. |
| E3 | User regenerate Test Plan: hệ thống xóa Test Case/Unit Test phía sau sau khi xác nhận. |

**UC-07. Sinh và duyệt Test Case**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Chuyển Test Plan thành Test Case cụ thể, đủ dữ liệu để sinh Unit Test. |
| Actor chính | User |
| Actor phụ | AI QA Agent, System |
| Tiền điều kiện | Test Plan đã được phê duyệt. |
| Kích hoạt | User yêu cầu sinh Test Case. |
| Hậu điều kiện thành công | Test Case đủ 8 trường được phê duyệt. |
| Hậu điều kiện thất bại | Test Case thiếu thông tin không được dùng để sinh Unit Test. |
| Dữ liệu vào | Test Plan, Business Rule, method context, existing tests. |
| Dữ liệu ra | Test Case gồm Test ID, Test Type, Description, Preconditions, Test Data, Expected Result, Priority, Trace Source. |
| Yêu cầu liên quan | FR3.2, FR5.1 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | User bấm sinh Test Case. |
| 2 | Hệ thống gửi Test Plan và context liên quan cho AI. |
| 3 | AI sinh Test Case theo cấu trúc 8 trường. |
| 4 | Hệ thống kiểm tra các trường bắt buộc và lưu Test Case chờ duyệt. |
| 5 | User review, chỉnh sửa và phê duyệt Test Case. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | Test Plan chưa duyệt: hệ thống không cho sinh Test Case. |
| E2 | Test Case thiếu trường bắt buộc: hệ thống báo lỗi hoặc yêu cầu sinh lại. |
| E3 | User regenerate Test Case: hệ thống xóa Unit Test phía sau sau khi xác nhận. |

**UC-08. Sinh và xuất Unit Test**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Sinh hoặc cải thiện Unit Test JUnit 5 + Mockito từ Test Case đã duyệt. |
| Actor chính | User |
| Actor phụ | AI QA Agent, System |
| Tiền điều kiện | Test Case đã được phê duyệt. |
| Kích hoạt | User yêu cầu sinh Unit Test. |
| Hậu điều kiện thành công | Unit Test được lưu và có thể export ZIP. |
| Hậu điều kiện thất bại | Không có file test hợp lệ để tải xuống. |
| Dữ liệu vào | Test Case, class under test, dependencies, existing test source nếu có. |
| Dữ liệu ra | Source code test, `generation_type`, package/file path. |
| Yêu cầu liên quan | FR4.1, FR4.2 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | User bấm sinh Unit Test. |
| 2 | Hệ thống xây dựng context từ Test Case và source code liên quan. |
| 3 | AI sinh test mới hoặc đề xuất cải thiện/bổ sung test cũ. |
| 4 | Hệ thống lưu code test với `generation_type` tương ứng. |
| 5 | User tải ZIP chứa các file test theo cấu trúc `src/test/java`. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | Test Case chưa duyệt: hệ thống không cho sinh Unit Test. |
| E2 | LLM trả code không đúng format JSON: hệ thống retry tối đa 2 lần. |
| E3 | Code sinh ra chưa compile/pass trong project gốc: User tự kiểm tra và điều chỉnh sau khi tải về. |

**UC-09. Xem Traceability Matrix**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Cho phép truy vết từ Business Rule đến Test Plan, Test Case và Unit Test. |
| Actor chính | User, Admin |
| Actor phụ | System |
| Tiền điều kiện | User có quyền truy cập project. |
| Kích hoạt | User mở màn hình Traceability Matrix. |
| Hậu điều kiện thành công | Hệ thống hiển thị được chuỗi liên kết hiện có. |
| Hậu điều kiện thất bại | Hệ thống báo không có dữ liệu hoặc từ chối truy cập. |
| Dữ liệu vào | Project ID. |
| Dữ liệu ra | Matrix Business Rule → Test Plan → Test Case → Unit Test. |
| Yêu cầu liên quan | FR5.1 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | User chọn project cần xem traceability. |
| 2 | Hệ thống kiểm tra quyền truy cập project. |
| 3 | Hệ thống truy vấn view/bảng liên quan để dựng matrix. |
| 4 | Hệ thống hiển thị các liên kết hiện có và phần còn thiếu nếu có. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | User không phải owner và không phải Admin: hệ thống từ chối truy cập. |
| E2 | Project chưa có dữ liệu test: hệ thống hiển thị matrix rỗng hoặc thiếu liên kết. |

**UC-10. Upload JaCoCo và phát hiện gap**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Phân tích coverage từ JaCoCo XML và đề xuất bổ sung test cho vùng chưa đủ độ phủ. |
| Actor chính | User |
| Actor phụ | AI QA Agent, System |
| Tiền điều kiện | User đã tải Unit Test, chạy test ở máy local và có file `jacoco.xml`. |
| Kích hoạt | User upload file JaCoCo XML. |
| Hậu điều kiện thành công | Coverage report được lưu, gap được phát hiện và có thể đề xuất thêm test. |
| Hậu điều kiện thất bại | File coverage không được lưu. |
| Dữ liệu vào | File `jacoco.xml`. |
| Dữ liệu ra | Line coverage, branch coverage, requirement coverage, danh sách gap. |
| Yêu cầu liên quan | FR5.2 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | User upload file `jacoco.xml`. |
| 2 | Hệ thống kiểm tra file có đúng định dạng JaCoCo XML hay không. |
| 3 | Hệ thống parse line coverage và branch coverage. |
| 4 | Hệ thống liên kết coverage với method trong project nếu có thể. |
| 5 | Hệ thống phát hiện gap và gọi AI đề xuất thêm Test Case khi cần. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | File không phải JaCoCo XML hợp lệ: hệ thống báo lỗi và không lưu report. |
| E2 | Không map được method coverage với source đã phân tích: hệ thống vẫn hiển thị coverage tổng quan. |
| E3 | AI đề xuất thêm test thất bại: coverage report vẫn được lưu, gap hiển thị không có gợi ý AI. |

**UC-11. Xuất báo cáo dự án**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Xuất dữ liệu dự án để lưu trữ, trình bày hoặc phục vụ đánh giá thực nghiệm. |
| Actor chính | User, Admin |
| Actor phụ | System |
| Tiền điều kiện | User có quyền truy cập project. |
| Kích hoạt | User chọn định dạng xuất báo cáo. |
| Hậu điều kiện thành công | Hệ thống trả file JSON hoặc Markdown. |
| Hậu điều kiện thất bại | Không xuất file nếu project không tồn tại hoặc user không có quyền. |
| Dữ liệu vào | Project ID, định dạng `json` hoặc `markdown`. |
| Dữ liệu ra | Báo cáo project gồm analysis, rule, plan, case, unit test, traceability và coverage nếu có. |
| Yêu cầu liên quan | FR5.3 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | User chọn project và định dạng báo cáo. |
| 2 | Hệ thống kiểm tra quyền truy cập. |
| 3 | Hệ thống tổng hợp dữ liệu liên quan đến project. |
| 4 | Hệ thống tạo file JSON hoặc Markdown. |
| 5 | User tải file báo cáo. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | Định dạng không hỗ trợ: hệ thống báo lỗi, chỉ nhận JSON/Markdown. |
| E2 | Project không tồn tại hoặc không thuộc quyền truy cập: hệ thống từ chối export. |

**UC-12. Quản lý người dùng và dữ liệu hệ thống**

| **Thuộc tính** | **Nội dung** |
| -------------- | ------------ |
| Mục tiêu | Cho phép Admin kiểm tra người dùng và dữ liệu project để phục vụ bảo trì/thực nghiệm. |
| Actor chính | Admin |
| Actor phụ | System |
| Tiền điều kiện | Admin đã đăng nhập. |
| Kích hoạt | Admin mở màn hình quản trị. |
| Hậu điều kiện thành công | Admin xem được danh sách user/project và dữ liệu cần kiểm tra. |
| Hậu điều kiện thất bại | Người dùng không phải Admin không truy cập được chức năng quản trị. |
| Dữ liệu vào | Bộ lọc user/project nếu có. |
| Dữ liệu ra | Danh sách user, project và trạng thái xử lý. |
| Yêu cầu liên quan | FR6.2, FR6.3 |

| **Bước** | **Luồng chính** |
| -------- | --------------- |
| 1 | Admin đăng nhập vào hệ thống. |
| 2 | Admin mở khu vực quản trị. |
| 3 | Hệ thống kiểm tra vai trò `ADMIN`. |
| 4 | Hệ thống hiển thị danh sách user/project và trạng thái dữ liệu. |
| 5 | Admin kiểm tra dữ liệu phục vụ bảo trì hoặc đánh giá thực nghiệm. |

| **Mã ngoại lệ** | **Luồng ngoại lệ** |
| --------------- | ------------------ |
| E1 | User không có vai trò `ADMIN`: hệ thống từ chối truy cập. |
| E2 | Dữ liệu project bị lỗi hoặc thiếu: hệ thống hiển thị trạng thái lỗi để Admin kiểm tra. |

**8\. Ma trận truy vết yêu cầu**

| **Nhóm yêu cầu** | **FR liên quan** | **Use Case liên quan** | **Tạo tác/Kết quả chính** |
| ---------------- | ---------------- | ---------------------- | ------------------------- |
| Nhập và phân tích project | FR1.1, FR1.2, FR1.3, FR1.4, FR1.5 | UC-02, UC-03, UC-04 | Project, Static Analysis Context, Existing Test Context, Manifest |
| Business Rule HITL | FR2.1, FR2.2, FR2.3 | UC-05 | Business Rule đã review/phê duyệt |
| Test Plan và Test Case | FR3.1, FR3.2, FR3.3 | UC-06, UC-07 | Test Plan, Test Case, trạng thái regenerate |
| Sinh và xuất Unit Test | FR4.1, FR4.2 | UC-08 | Source code Unit Test, file ZIP xuất ra |
| Traceability, coverage và báo cáo | FR5.1, FR5.2, FR5.3 | UC-09, UC-10, UC-11 | Traceability Matrix, Coverage Report, báo cáo JSON/Markdown |
| Xác thực và phân quyền | FR6.1, FR6.2, FR6.3 | UC-01, UC-12 | Tài khoản, vai trò, kiểm soát truy cập project/API |
