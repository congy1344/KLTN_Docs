# DATA_MODEL.md — Database Schema

## Tổng quan

Database PostgreSQL với các bảng chính phục vụ Static Analysis, Test Generation và Traceability Matrix.

## ERD (Entity Relationship Diagram)

```
auth_user (1) ──── (n) project (1) ──── (n) java_class
   │                    │
   │                    └── (n) java_method
   │                              │
   │                              ├── (n) endpoint
   │                              │
   │                              ├── (n) relevant_annotation
   │                              │
   │                              └── (n) business_rule
   │                                       │
   │                                       └── (n) test_plan
   │                                                 │
   │                                                 └── (n) test_case
   │                                                          │
   │                                                          └── (1) unit_test
   │
   ├── (n) controller_service_relation
   │
   ├── (n) service_repository_relation
   │
   ├── (n) existing_test
   │
   └── (1) coverage_report
              │
              └── (n) coverage_detail
```

## Chi tiết các bảng

### 1. `project`
Lưu thông tin project được upload.

```sql
CREATE TABLE project (
    id              BIGSERIAL PRIMARY KEY,
    owner_user_id   BIGINT REFERENCES auth_user(id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    source_type     VARCHAR(20) NOT NULL,  -- 'ZIP' | 'GITHUB'
    source_url      TEXT,                  -- URL GitHub nếu có
    storage_path    TEXT,                  -- Path lưu source code
    status          VARCHAR(50) NOT NULL,  -- State machine
    total_production_files INT,            -- Số production .java files cần parse
    parsed_production_files INT,           -- Số production .java files parse thành công
    failed_parse_files INT,                -- Số production .java files bị skip do lỗi parser
    failed_parse_file_paths JSONB,         -- ["module/src/main/java/..."]
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);
```

**Status values:**
- `UPLOADED`
- `ANALYZED`
- `BR_PENDING_REVIEW`
- `BR_APPROVED`
- `PLAN_PENDING_REVIEW`
- `PLAN_APPROVED`
- `CASE_PENDING_REVIEW`
- `CASE_APPROVED`
- `TEST_GENERATED`
- `COVERAGE_ANALYZED`
- `COMPLETED`
- `FAILED`

### 1.1. `auth_user`
Lưu tài khoản đăng nhập và phân quyền cơ bản.

```sql
CREATE TABLE auth_user (
    id              BIGSERIAL PRIMARY KEY,
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255),
    role            VARCHAR(30) NOT NULL, -- 'USER' | 'ADMIN'
    enabled         BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_auth_user_role ON auth_user(role);
```

Quy tắc truy cập:
- `USER` chỉ xem/sửa project có `project.owner_user_id = auth_user.id`.
- `ADMIN` có thể xem/sửa toàn bộ project và quản lý user.

### 2. `java_class`
Lưu thông tin class trích xuất từ JavaParser.

```sql
CREATE TABLE java_class (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    package_name    VARCHAR(500),
    class_name      VARCHAR(255) NOT NULL,
    qualified_name  VARCHAR(1000) NOT NULL, -- Identity ổn định, gồm package + nested type
    file_path       TEXT,                  -- Đường dẫn file gốc
    class_type      VARCHAR(50),           -- SERVICE | CONTROLLER | REPOSITORY | ENTITY | ENUM | RECORD | OTHER
    source_code     TEXT,                  -- Toàn bộ code của class (optional, có thể chỉ giữ signature)
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_java_class_project ON java_class(project_id);
CREATE INDEX idx_java_class_type ON java_class(class_type);
CREATE INDEX idx_java_class_qualified_name ON java_class(qualified_name);
```

### 3. `java_method`
Lưu thông tin method trong class.

```sql
CREATE TABLE java_method (
    id              BIGSERIAL PRIMARY KEY,
    class_id        BIGINT NOT NULL REFERENCES java_class(id) ON DELETE CASCADE,
    method_name     VARCHAR(255) NOT NULL,
    return_type     VARCHAR(255),
    parameters      JSONB,                 -- [{ name, type }, ...]
    throws_list     JSONB,                 -- ['Exception1', 'Exception2']
    visibility      VARCHAR(20),           -- 'PUBLIC' | 'PRIVATE' | 'PROTECTED'
    source_code     TEXT,                  -- Body của method
    line_start      INT,
    line_end        INT
);

CREATE INDEX idx_java_method_class ON java_method(class_id);
```

### 4. `endpoint`
Lưu REST endpoint trích xuất từ Controller.

```sql
CREATE TABLE endpoint (
    id              BIGSERIAL PRIMARY KEY,
    method_id       BIGINT NOT NULL REFERENCES java_method(id) ON DELETE CASCADE,
    http_method     VARCHAR(10) NOT NULL,  -- ANY | GET | HEAD | POST | PUT | DELETE | PATCH | OPTIONS | TRACE
    path            VARCHAR(500) NOT NULL,
    consumes        VARCHAR(255),
    produces        VARCHAR(255)
);

CREATE INDEX idx_endpoint_method ON endpoint(method_id);
```

### 5. `service_repository_relation`
Lưu quan hệ Service → Repository.

```sql
CREATE TABLE service_repository_relation (
    id              BIGSERIAL PRIMARY KEY,
    service_class_id    BIGINT NOT NULL REFERENCES java_class(id) ON DELETE CASCADE,
    repository_class_id BIGINT NOT NULL REFERENCES java_class(id) ON DELETE CASCADE
);

CREATE UNIQUE INDEX uq_service_repository_relation
    ON service_repository_relation(service_class_id, repository_class_id);
```

### 6. `relevant_annotation`
Lưu annotation chọn lọc phục vụ phân loại component, endpoint, validation, security, transaction, persistence và test context.

```sql
CREATE TABLE relevant_annotation (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    class_id        BIGINT REFERENCES java_class(id) ON DELETE CASCADE,
    method_id       BIGINT REFERENCES java_method(id) ON DELETE CASCADE,
    target_type     VARCHAR(50) NOT NULL,  -- CLASS | METHOD
    category        VARCHAR(50) NOT NULL,  -- COMPONENT | ENDPOINT | VALIDATION | ...
    annotation_name VARCHAR(255) NOT NULL,
    attributes      VARCHAR(2000)
);

CREATE INDEX idx_relevant_annotation_class ON relevant_annotation(class_id);
CREATE INDEX idx_relevant_annotation_method ON relevant_annotation(method_id);
```

### 7. `controller_service_relation`
Lưu quan hệ Controller Method → Service Method khi static analysis resolve chắc chắn field dependency và direct method call.

```sql
CREATE TABLE controller_service_relation (
    id                   BIGSERIAL PRIMARY KEY,
    controller_class_id  BIGINT NOT NULL REFERENCES java_class(id) ON DELETE CASCADE,
    controller_method_id BIGINT NOT NULL REFERENCES java_method(id) ON DELETE CASCADE,
    service_class_id     BIGINT NOT NULL REFERENCES java_class(id) ON DELETE CASCADE,
    service_method_id    BIGINT REFERENCES java_method(id) ON DELETE SET NULL,
    service_field_name   VARCHAR(255) NOT NULL,
    service_field_type   VARCHAR(1000) NOT NULL,
    called_method_name   VARCHAR(255) NOT NULL
);

CREATE INDEX idx_csr_controller_class ON controller_service_relation(controller_class_id);
CREATE INDEX idx_csr_controller_method ON controller_service_relation(controller_method_id);
CREATE INDEX idx_csr_service_class ON controller_service_relation(service_class_id);
```

### 8. `business_rule`
Lưu Business Rule (do AI đề xuất hoặc user nhập).

```sql
CREATE TABLE business_rule (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    method_id       BIGINT REFERENCES java_method(id) ON DELETE SET NULL,
    rule_code       VARCHAR(50) NOT NULL,  -- 'BR-001', 'BR-002'
    description     TEXT NOT NULL,
    source          VARCHAR(30) NOT NULL,  -- 'AI_GENERATED' | 'USER_ADDED' | 'USER_MODIFIED' | 'AI_REVIEW_SUGGESTED'
    status          VARCHAR(30) NOT NULL,  -- 'PENDING_REVIEW' | 'APPROVED' | 'REJECTED'
    review_note     TEXT,                  -- Nhận xét/gợi ý của AI khi review rule user nhập
    is_modified     BOOLEAN DEFAULT FALSE, -- Đánh dấu user đã sửa từ bản AI sinh ra
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_business_rule_project ON business_rule(project_id);
CREATE INDEX idx_business_rule_method ON business_rule(method_id);
```

### 9. `test_plan`
Lưu Test Plan được sinh từ Business Rule.

```sql
CREATE TABLE test_plan (
    id              BIGSERIAL PRIMARY KEY,
    project_id      BIGINT NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    business_rule_id BIGINT NOT NULL REFERENCES business_rule(id) ON DELETE CASCADE,
    plan_code       VARCHAR(50) NOT NULL,  -- 'TP-001'
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    test_type       VARCHAR(30) NOT NULL,  -- 'HAPPY_PATH' | 'BOUNDARY' | 'EXCEPTION' | 'EDGE'
    status          VARCHAR(30) NOT NULL,
    is_modified     BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_test_plan_project ON test_plan(project_id);
CREATE INDEX idx_test_plan_rule ON test_plan(business_rule_id);
```

### 10. `test_case`
Lưu Test Case (8 trường).

```sql
CREATE TABLE test_case (
    id              BIGSERIAL PRIMARY KEY,
    test_plan_id    BIGINT NOT NULL REFERENCES test_plan(id) ON DELETE CASCADE,
    case_code       VARCHAR(50) NOT NULL,  -- 'TC-001'
    test_type       VARCHAR(30) NOT NULL,  -- Kế thừa từ test_plan
    description     TEXT NOT NULL,
    preconditions   TEXT,
    test_data       JSONB,                 -- { input: {...}, mocks: {...} }
    expected_result TEXT NOT NULL,
    priority        VARCHAR(10) NOT NULL,  -- 'HIGH' | 'MEDIUM' | 'LOW'
    trace_source    VARCHAR(500),          -- 'BR-001 → TP-001 → TC-001'
    status          VARCHAR(30) NOT NULL,
    is_modified     BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_test_case_plan ON test_case(test_plan_id);
```

### 11. `unit_test`
Lưu code Unit Test sinh ra.

```sql
CREATE TABLE unit_test (
    id              BIGSERIAL PRIMARY KEY,
    test_case_id    BIGINT NOT NULL REFERENCES test_case(id) ON DELETE CASCADE,
    test_class_name VARCHAR(255) NOT NULL, -- Vd: 'UserServiceTest'
    test_method_name VARCHAR(255) NOT NULL,-- Vd: 'testCreateUser_Success'
    package_name    VARCHAR(500),
    generation_type VARCHAR(40),           -- NEW_TEST | IMPROVE_EXISTING_TEST | SUPPLEMENT_EXISTING_TEST
    existing_test_file_path TEXT,          -- Nếu output dựa trên test có sẵn
    source_code     TEXT NOT NULL,         -- Code Java đầy đủ
    file_path       TEXT,                  -- Path tương đối khi export
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_unit_test_case ON unit_test(test_case_id);
```

### 11.1. `existing_test`
Lưu metadata test có sẵn trong project đầu vào.

```sql
CREATE TABLE existing_test (
    id                  BIGSERIAL PRIMARY KEY,
    project_id           BIGINT NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    file_path            TEXT NOT NULL,
    package_name         VARCHAR(500),
    test_class_name      VARCHAR(255),
    related_class_id     BIGINT REFERENCES java_class(id) ON DELETE SET NULL,
    related_method_id    BIGINT REFERENCES java_method(id) ON DELETE SET NULL,
    test_methods         JSONB,             -- [{ name, annotations, assertions, mocks }]
    imports              JSONB,
    source_code          TEXT,
    created_at           TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_existing_test_project ON existing_test(project_id);
CREATE INDEX idx_existing_test_related_class ON existing_test(related_class_id);
```

### 12. `coverage_report`
Lưu kết quả JaCoCo upload.

```sql
CREATE TABLE coverage_report (
    id                  BIGSERIAL PRIMARY KEY,
    project_id          BIGINT NOT NULL REFERENCES project(id) ON DELETE CASCADE,
    line_coverage       DECIMAL(5,2),      -- 85.50%
    branch_coverage     DECIMAL(5,2),
    requirement_coverage DECIMAL(5,2),
    total_lines         INT,
    covered_lines       INT,
    total_branches      INT,
    covered_branches    INT,
    uploaded_at         TIMESTAMP NOT NULL DEFAULT NOW(),
    xml_file_path       TEXT                -- Path file XML đã upload
);

CREATE INDEX idx_coverage_project ON coverage_report(project_id);
```

### 13. `coverage_detail`
Lưu coverage cho từng method (để detect gap).

```sql
CREATE TABLE coverage_detail (
    id              BIGSERIAL PRIMARY KEY,
    report_id       BIGINT NOT NULL REFERENCES coverage_report(id) ON DELETE CASCADE,
    method_id       BIGINT REFERENCES java_method(id) ON DELETE SET NULL,
    line_coverage   DECIMAL(5,2),
    branch_coverage DECIMAL(5,2),
    missed_lines    JSONB,                 -- [10, 15, 23, ...]
    missed_branches JSONB,
    has_gap         BOOLEAN DEFAULT FALSE  -- Coverage < threshold
);

CREATE INDEX idx_coverage_detail_report ON coverage_detail(report_id);
CREATE INDEX idx_coverage_detail_method ON coverage_detail(method_id);
```

## Quan hệ Traceability

Traceability Matrix là **view ảo** dựng từ các bảng trên:

```sql
-- View để truy vấn Traceability Matrix
CREATE VIEW v_traceability AS
SELECT
    br.id              AS rule_id,
    br.rule_code       AS rule_code,
    br.description     AS rule_description,
    tp.id              AS plan_id,
    tp.plan_code       AS plan_code,
    tp.title           AS plan_title,
    tp.test_type       AS test_type,
    tc.id              AS case_id,
    tc.case_code       AS case_code,
    tc.description     AS case_description,
    ut.id              AS unit_test_id,
    ut.test_method_name AS unit_test_name,
    br.project_id      AS project_id
FROM business_rule br
LEFT JOIN test_plan tp ON tp.business_rule_id = br.id
LEFT JOIN test_case tc ON tc.test_plan_id = tp.id
LEFT JOIN unit_test ut ON ut.test_case_id = tc.id;
```

## Metric Tracking

Các bảng để lưu metric đánh giá thực nghiệm:

### 14. `experiment_metric`

```sql
CREATE TABLE experiment_metric (
    id                      BIGSERIAL PRIMARY KEY,
    project_id              BIGINT NOT NULL REFERENCES project(id),
    method_used             VARCHAR(50),   -- 'LLM_ONLY' | 'PROMPT_BASED' | 'GREY_BOX'
    requirement_coverage    DECIMAL(5,2),
    line_coverage           DECIMAL(5,2),
    branch_coverage         DECIMAL(5,2),
    generation_time_seconds INT,
    user_modification_rate  DECIMAL(5,2),
    input_tokens            INT,
    output_tokens           INT,
    stability_score         DECIMAL(5,2),
    traceability_score      DECIMAL(5,2),
    run_at                  TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## Migration với Flyway

Đặt file migration tại `backend/src/main/resources/db/migration/`:

```
V1__create_project_table.sql
V2__create_java_class_table.sql
V3__create_java_method_table.sql
V4__create_endpoint_table.sql
V5__create_service_repository_relation_table.sql
V6__create_business_rule_table.sql
V7__create_test_plan_table.sql
V8__create_test_case_table.sql
V9__create_unit_test_table.sql
V10__create_coverage_tables.sql
V11__create_experiment_metric_table.sql
V12__create_traceability_view.sql
V13__enhance_static_analysis_identity.sql
V14__add_test_generation_analysis_context.sql
V15__add_parse_failure_stats_to_project.sql
V16__add_auth_user_and_project_owner.sql
V17__add_existing_test_context.sql
V18__enforce_artifact_codes.sql
```

> Cập nhật: bổ sung `V5` cho bảng `service_repository_relation` (bản đầu thiếu),
> các migration sau dời số tương ứng; view traceability chuyển sang `V12`.
> `V13` bổ sung fully-qualified identity và chống trùng Service→Repository relation.
> `V14` bổ sung relevant annotation và Controller→Service relation phục vụ test generation context.
> `V15` bổ sung thống kê parse lỗi để analysis có thể chạy best-effort trên project lớn/legacy.
> `V16` bổ sung user đăng nhập, role và owner cho project.
> `V17` bổ sung Existing Test Context và metadata cho unit test sinh mới/cải thiện.

## Lưu ý quan trọng

### Cascade Delete
Tất cả foreign key từ project xuống đều có `ON DELETE CASCADE`. Khi xóa project, toàn bộ dữ liệu liên quan bị xóa.

### Soft Delete (KHÔNG dùng)
Để đơn giản, KHÔNG dùng soft delete. Xóa là xóa thật.

### JSONB Fields
- `parameters`, `throws_list`, `test_data`, `test_methods`, `imports`, `missed_lines`: dùng JSONB cho flexibility
- Không index JSONB trừ khi thực sự cần query trên field con

### Index
Đã có index trên các foreign key và trường thường dùng để filter. Không cần thêm index cho đến khi gặp vấn đề performance thực sự.
