# AI_AGENT.md — Chi tiết AI QA Agent

## Vai trò

AI QA Agent là **trái tim** của hệ thống. Nó nhận dữ liệu đã được Static Analysis xử lý, kết hợp với Business Rule và Existing Test Context nếu project đã có test, rồi sinh ra Test Plan / Test Case / Unit Test mới hoặc cải thiện test hiện có.

## Kiến trúc Agent

```
┌─────────────────────────────────────────────────┐
│              AIAgentService                     │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │       PromptManager                     │    │
│  │  Load template, fill variables          │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │       LLMClient                         │    │
│  │  Gọi OpenAI / Google Gemini API         │    │
│  │  Retry, timeout, token counting         │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │       ResponseParser                    │    │
│  │  Parse JSON từ LLM, validate schema     │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

## 5 chức năng chính

### 1. `generateBusinessRules(projectId)`

**Mục đích:** AI đọc source code và tự đề xuất Business Rule cho từng Service method khi user chọn chế độ "AI auto sinh BR".

**Input:**
- Danh sách Service class và method chưa có Business Rule (đã có từ Static Analysis)
- Source code của các method quan trọng
- Existing tests liên quan nếu đã có, dùng để tránh đề xuất rule/test trùng lặp hoặc bỏ sót hành vi đã được test

**Output (JSON từ LLM):**
```json
{
  "rules": [
    {
      "method_id": 123,
      "description": "Số dư tài khoản phải đủ trước khi thực hiện chuyển tiền",
      "category": "VALIDATION"
    },
    {
      "method_id": 123,
      "description": "Tài khoản đích phải tồn tại trong hệ thống",
      "category": "VALIDATION"
    }
  ]
}
```

**Lưu ý:**
- Người dùng SẼ review và chỉnh sửa output này
- AI đề xuất 1-5 rule độc lập cho mỗi method khi phù hợp; một method có thể có validation, business logic và side effect riêng
- Đo `User Modification Rate` để đánh giá độ chính xác của AI
- Rule sinh ra có `source = AI_GENERATED`, `status = PENDING_REVIEW`

### 2. `reviewBusinessRules(projectId)`

**Mục đích:** AI review Business Rule do user nhập trước, sau đó gợi ý sửa rule hiện có và đề xuất rule còn thiếu.

**Input:**
- Danh sách Business Rule user đã nhập (`source = USER_ADDED` hoặc `USER_MODIFIED`)
- Method/source context liên quan
- Annotation, endpoint, Controller-Service relation, Service-Repository relation nếu có
- Existing Test Context nếu có

**Output (JSON từ LLM):**
```json
{
  "reviewed_rules": [
    {
      "rule_id": 10,
      "verdict": "NEEDS_REVISION",
      "suggested_description": "Số dư tài khoản phải lớn hơn hoặc bằng số tiền chuyển trước khi tạo giao dịch",
      "reason": "Rule hiện tại thiếu điều kiện so sánh cụ thể"
    }
  ],
  "suggested_rules": [
    {
      "method_id": 123,
      "description": "Không cho phép chuyển tiền tới chính tài khoản nguồn",
      "category": "VALIDATION"
    }
  ]
}
```

**Lưu ý:**
- AI không tự ghi đè rule user nhập. Gợi ý được hiển thị để user chấp nhận hoặc bỏ qua.
- Rule AI đề xuất thêm có `source = AI_REVIEW_SUGGESTED`, `status = PENDING_REVIEW`.

### 3. `generateTestPlan(projectId)`

**Mục đích:** Từ Business Rule đã được duyệt, sinh Test Plan theo 4 nhóm.

**Input:**
- Danh sách Business Rule đã APPROVED
- Method tương ứng với mỗi rule (source code)
- Existing Test Context: test class/test method/assertion/mocking pattern hiện có nếu liên quan

**Output:**
```json
{
  "plans": [
    {
      "rule_id": 45,
      "title": "Chuyển tiền thành công khi số dư đủ",
      "description": "Kiểm tra luồng chuyển tiền chính khi mọi điều kiện hợp lệ",
      "test_type": "HAPPY_PATH"
    },
    {
      "rule_id": 45,
      "title": "Chuyển tiền với số tiền tối đa cho phép",
      "description": "Kiểm tra ranh giới trên của số tiền chuyển",
      "test_type": "BOUNDARY"
    },
    {
      "rule_id": 45,
      "title": "Chuyển tiền khi số dư không đủ",
      "description": "Kiểm tra exception khi không đủ tiền",
      "test_type": "EXCEPTION"
    }
  ]
}
```

**Quy tắc:**
- Một Business Rule có thể có nhiều Test Plan (nhiều loại test)
- KHÔNG bắt buộc đủ 4 loại — AI quyết định loại nào phù hợp
- Mỗi Test Plan PHẢI gán `test_type` rõ ràng

### 4. `generateTestCases(projectId)`

**Mục đích:** Từ Test Plan, sinh Test Case cụ thể với 8 trường.

**Input:**
- Danh sách Test Plan đã APPROVED
- Business Rule và Method tương ứng
- Existing Test Context để biết scenario nào đã có test, scenario nào cần cải thiện hoặc bổ sung

**Output:**
```json
{
  "cases": [
    {
      "plan_id": 78,
      "test_type": "HAPPY_PATH",
      "description": "Chuyển 500,000đ thành công khi số dư đủ",
      "preconditions": "Tài khoản A có số dư 2,000,000đ. Tài khoản B tồn tại.",
      "test_data": {
        "input": {
          "amount": 500000,
          "sourceAccount": "ACC-A",
          "targetAccount": "ACC-B"
        },
        "mocks": {
          "accountRepository.findByCode(\"ACC-A\")": "Account(balance=2000000)",
          "accountRepository.findByCode(\"ACC-B\")": "Account(balance=0)"
        }
      },
      "expected_result": "Transaction thành công. Số dư A còn 1,500,000đ. Số dư B = 500,000đ.",
      "priority": "HIGH",
      "trace_source": "BR-001 → TP-001"
    }
  ]
}
```

### 5. `generateUnitTests(projectId)`

**Mục đích:** Từ Test Case, sinh code JUnit 5 + Mockito hoặc cải thiện test class hiện có.

**Input:**
- Test Case đã APPROVED
- Class/Method/Imports cần thiết
- Existing test source liên quan nếu có

**Output:**
```json
{
  "unit_tests": [
    {
      "case_id": 156,
      "test_class_name": "TransferServiceTest",
      "test_method_name": "testTransferMoney_Success",
      "package_name": "com.example.service",
      "generation_type": "NEW_TEST",
      "source_code": "package com.example.service;\n\nimport ...\n\n@ExtendWith(MockitoExtension.class)\nclass TransferServiceTest {\n    @Mock\n    private AccountRepository accountRepository;\n    \n    @InjectMocks\n    private TransferService transferService;\n    \n    @Test\n    void testTransferMoney_Success() {\n        // Arrange\n        Account sourceAccount = new Account(\"ACC-A\", 2000000L);\n        Account targetAccount = new Account(\"ACC-B\", 0L);\n        when(accountRepository.findByCode(\"ACC-A\")).thenReturn(sourceAccount);\n        when(accountRepository.findByCode(\"ACC-B\")).thenReturn(targetAccount);\n        \n        // Act\n        transferService.transferMoney(500000L, \"ACC-A\", \"ACC-B\");\n        \n        // Assert\n        assertEquals(1500000L, sourceAccount.getBalance());\n        assertEquals(500000L, targetAccount.getBalance());\n    }\n}"
    }
  ]
}
```

**Yêu cầu code sinh ra:**
- Sử dụng JUnit 5 (`@Test`, `@BeforeEach`)
- Sử dụng Mockito (`@Mock`, `@InjectMocks`, `when().thenReturn()`)
- Theo cấu trúc AAA (Arrange-Act-Assert)
- Đúng package, đúng import
- Compile được (mặc dù chưa chắc test pass)
- Nếu đã có test class phù hợp, ưu tiên bổ sung test method hoặc cải thiện assertion/mocking thay vì tạo class trùng tên.
- Output phải ghi rõ `generation_type`: `NEW_TEST`, `IMPROVE_EXISTING_TEST` hoặc `SUPPLEMENT_EXISTING_TEST`.

## Prompt Templates

Đặt tại `backend/src/main/resources/prompts/`.

### `business-rule.md`

```markdown
# System
Bạn là chuyên gia phân tích nghiệp vụ phần mềm. Nhiệm vụ của bạn là đọc source code Java Spring Boot và tự đề xuất các Business Rule cho từng method.

# Context
Tôi sẽ cung cấp cho bạn:
1. Danh sách Service class với các method
2. Source code của từng method

# Yêu cầu
- Với mỗi method, đề xuất 1-5 Business Rule.
- Business Rule là MỘT CÂU ngắn gọn mô tả yêu cầu nghiệp vụ.
- Tập trung vào: validation, business constraint, side effect.
- KHÔNG mô tả lại code, mà mô tả YÊU CẦU mà code đang implement.

# Output Format
Trả về JSON duy nhất, không markdown, không giải thích:
{
  "rules": [
    {
      "method_id": <id>,
      "description": "<câu mô tả>",
      "category": "VALIDATION" | "BUSINESS_LOGIC" | "SIDE_EFFECT"
    }
  ]
}

# Input
{{methods_json}}

Existing Tests:
{{existing_tests_json}}
```

### `business-rule-review.md`

```markdown
# System
Bạn là chuyên gia phân tích nghiệp vụ phần mềm. Nhiệm vụ của bạn là review Business Rule do user nhập, sau đó đề xuất chỉnh sửa hoặc bổ sung rule còn thiếu.

# Context
Tôi sẽ cung cấp cho bạn:
1. Business Rule hiện có của user
2. Danh sách Service method và source code liên quan
3. Annotation/relation/endpoint context nếu có
4. Existing tests nếu project đã có test

# Yêu cầu
- Không tự ghi đè rule của user.
- Với mỗi rule hiện có, đánh giá: hợp lý, mơ hồ, trùng lặp, thiếu điều kiện, hoặc gắn sai method.
- Nếu rule cần sửa, đề xuất câu mô tả mới rõ ràng hơn.
- Đề xuất thêm Business Rule còn thiếu nếu source code cho thấy constraint quan trọng.
- Business Rule là MỘT CÂU ngắn gọn mô tả yêu cầu nghiệp vụ, không mô tả lại code.

# Output Format
Trả về JSON duy nhất, không markdown, không giải thích:
{
  "reviewed_rules": [
    {
      "rule_id": <id>,
      "verdict": "OK" | "NEEDS_REVISION" | "DUPLICATE" | "WRONG_METHOD" | "TOO_VAGUE",
      "suggested_description": "<câu mô tả mới hoặc null>",
      "reason": "<lý do ngắn>"
    }
  ],
  "suggested_rules": [
    {
      "method_id": <id>,
      "description": "<rule còn thiếu>",
      "category": "VALIDATION" | "BUSINESS_LOGIC" | "SIDE_EFFECT"
    }
  ]
}

# Input
User Business Rules:
{{business_rules_json}}

Methods:
{{methods_json}}

Analysis Context:
{{analysis_context_json}}

Existing Tests:
{{existing_tests_json}}
```

### `test-plan.md`

```markdown
# System
Bạn là chuyên gia kiểm thử phần mềm. Nhiệm vụ là sinh Test Plan từ Business Rule.

# Yêu cầu
Với mỗi Business Rule, sinh các Test Plan thuộc 4 loại:
- HAPPY_PATH: Trường hợp thuận lợi, đúng kỳ vọng
- BOUNDARY: Tại ranh giới giá trị
- EXCEPTION: Dữ liệu sai, điều kiện không thỏa
- EDGE: Tình huống bất thường, hiếm gặp

Không bắt buộc đủ 4 loại cho mọi rule. Chỉ sinh loại nào hợp lý.

# Output Format
JSON duy nhất:
{
  "plans": [
    {
      "rule_id": <id>,
      "title": "<tiêu đề ngắn>",
      "description": "<mô tả chi tiết>",
      "test_type": "HAPPY_PATH" | "BOUNDARY" | "EXCEPTION" | "EDGE"
    }
  ]
}

# Input
Business Rules:
{{rules_json}}

Method source code:
{{methods_source}}

Existing Tests:
{{existing_tests_json}}
```

### `test-case.md`

```markdown
# System
Bạn là chuyên gia thiết kế test case. Nhiệm vụ là sinh Test Case cụ thể từ Test Plan.

# Yêu cầu
Mỗi Test Case PHẢI có đủ:
- description: Mô tả ngắn
- preconditions: Điều kiện trước test
- test_data: Input cụ thể (số, chuỗi, object) + mock cần thiết
- expected_result: Kết quả mong đợi rõ ràng
- priority: HIGH | MEDIUM | LOW

# Output Format
JSON duy nhất:
{
  "cases": [
    {
      "plan_id": <id>,
      "test_type": "<inherit từ plan>",
      "description": "...",
      "preconditions": "...",
      "test_data": { "input": {...}, "mocks": {...} },
      "expected_result": "...",
      "priority": "HIGH" | "MEDIUM" | "LOW",
      "trace_source": "BR-XXX → TP-XXX"
    }
  ]
}

# Input
Test Plans:
{{plans_json}}

Business Rules:
{{rules_json}}

Method context:
{{methods_context}}

Existing Tests:
{{existing_tests_json}}
```

### `unit-test.md`

```markdown
# System
Bạn là chuyên gia viết Unit Test Java. Nhiệm vụ là viết code JUnit 5 + Mockito từ Test Case.

# Yêu cầu kỹ thuật
- Dùng JUnit 5 (jakarta.persistence, không phải javax)
- Dùng Mockito với @ExtendWith(MockitoExtension.class)
- @Mock cho dependencies, @InjectMocks cho class under test
- Cấu trúc AAA: Arrange - Act - Assert
- Tên method: testMethodName_Scenario (vd: testTransferMoney_Success)
- Import đầy đủ, package đúng

# Output Format
JSON duy nhất:
{
  "unit_tests": [
    {
      "case_id": <id>,
      "test_class_name": "...",
      "test_method_name": "...",
      "package_name": "...",
      "generation_type": "NEW_TEST" | "IMPROVE_EXISTING_TEST" | "SUPPLEMENT_EXISTING_TEST",
      "source_code": "// Toàn bộ code Java"
    }
  ]
}

# Input
Test Cases:
{{cases_json}}

Class under test:
{{class_source}}

Dependencies:
{{dependencies}}

Existing test source:
{{existing_test_source}}
```

## Xử lý LLM Response

### Validate JSON Schema
Sử dụng `jackson` + `json-schema-validator` để validate response.

### Retry Strategy
```java
public <T> T callLLMWithRetry(String prompt, Class<T> responseType) {
    for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
        try {
            String response = llmClient.call(prompt);
            return parseAndValidate(response, responseType);
        } catch (LLMResponseException e) {
            log.warn("Attempt {} failed: {}", attempt, e.getMessage());
            if (attempt == MAX_RETRIES) throw e;
        }
    }
    throw new IllegalStateException("Should not reach here");
}
```

### Token Counting
- Đếm input tokens trước khi gửi
- Đếm output tokens từ response
- Lưu vào `experiment_metric` để tính `Token Cost`

## LLM Provider

### Mock (mặc định khi dev)
- Provider `mock` trả JSON deterministic, không cần API key, dùng để demo/test local nhanh.
- Đổi sang `openai` khi cần test thật với API AI.

### OpenAI
- Model: `gpt-4o-mini` cho dev, `gpt-4o` cho production
- Temperature: 0.3 (giảm randomness)
- Max tokens output: 4096

### Google Gemini
- Provider `google` dùng endpoint Gemini và model mặc định `gemini-3.5-flash`.
- Request kèm `response_format` và JSON schema theo từng prompt để giảm lỗi sai format.

### Cấu hình
File `application.yml`:
```yaml
llm:
  provider: ${LLM_PROVIDER:mock}  # mock, openai hoặc google
  api-key: ${LLM_API_KEY:}
  model: ${LLM_MODEL:gpt-4o-mini}
  temperature: ${LLM_TEMPERATURE:0.3}
  max-tokens: ${LLM_MAX_TOKENS:4096}
  timeout-seconds: ${LLM_TIMEOUT_SECONDS:60}
  openai-url: ${LLM_OPENAI_URL:https://api.openai.com/v1/responses}
  google-url: ${LLM_GOOGLE_URL:https://generativelanguage.googleapis.com/v1beta/interactions}
  max-retries: 2

greytest:
  ai-context-log-enabled: ${GREYTEST_AI_CONTEXT_LOG_ENABLED:false}
  ai-context-log-console-enabled: ${GREYTEST_AI_CONTEXT_LOG_CONSOLE_ENABLED:false}
  ai-context-log-path: ${GREYTEST_AI_CONTEXT_LOG_PATH:../log}
```

Context/prompt/response log tắt mặc định vì có thể chứa source code của project upload; chỉ bật khi debug local.

## Stability — Đo độ ổn định

Theo đề tài, hệ thống cần chạy 5 lần với cùng input để tính Stability Score. Implementation:

```java
public StabilityResult measureStability(Long projectId) {
    List<List<TestCase>> runs = new ArrayList<>();
    for (int i = 0; i < 5; i++) {
        List<TestCase> result = generateTestCases(projectId);
        runs.add(result);
    }
    return calculateStability(runs);
}
```

Stability = số test case giống nhau giữa các lần / tổng số test case.

So sánh "giống nhau" dựa trên:
- Cùng test_type
- Description tương tự (cosine similarity > 0.85)
- Cùng preconditions chính
