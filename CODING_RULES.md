# CODING_RULES.md — Quy tắc viết code

## Nguyên tắc chung

1. **Đơn giản trước, tối ưu sau** — Code dễ đọc quan trọng hơn code tối ưu
2. **KHÔNG copy-paste** — Nếu thấy đoạn code lặp 2 lần → refactor
3. **KHÔNG comment giải thích "code làm gì"** — Đặt tên biến/hàm cho dễ hiểu thay vì comment
4. **Comment "TẠI SAO" thay vì "LÀM GÌ"** — Khi logic nghiệp vụ phức tạp
5. **Tiếng Việt cho comment nghiệp vụ, tiếng Anh cho mọi thứ khác**

## Quy tắc Authentication & Authorization

- Password luôn hash bằng BCrypt, không lưu plain text.
- Không hard-code token, secret hoặc tài khoản admin trong code; dùng biến môi trường hoặc seed/migration có kiểm soát.
- API project/test generation phải kiểm tra user hiện tại có quyền truy cập project: owner hoặc `ADMIN`.
- Logic kiểm tra quyền đặt ở service/security layer, không chỉ dựa vào ẩn/hiện button ở frontend.
- Controller chỉ nhận request và gọi service; không chứa logic phân quyền phức tạp.

## Quy tắc đặt tên

### Backend (Java)

| Loại | Convention | Ví dụ |
|---|---|---|
| Class | PascalCase | `BusinessRuleService` |
| Interface | PascalCase | `LLMClient` |
| Method | camelCase, verb | `generateTestCases()`, `extractMethods()` |
| Variable | camelCase | `projectId`, `testPlanList` |
| Constant | UPPER_SNAKE | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Package | lowercase | `com.greytest.service.analysis` |
| Entity | PascalCase, singular | `BusinessRule` (không `BusinessRules`) |
| Table | snake_case, singular | `business_rule` (không `business_rules`) |
| Column | snake_case | `created_at`, `is_modified` |

### Frontend (TypeScript/React)

| Loại | Convention | Ví dụ |
|---|---|---|
| Component | PascalCase | `BusinessRuleList.tsx` |
| Hook | camelCase với `use` | `useBusinessRules()` |
| Function | camelCase | `fetchProject()` |
| Variable | camelCase | `selectedRule`, `isLoading` |
| Constant | UPPER_SNAKE | `API_BASE_URL` |
| Type/Interface | PascalCase | `type BusinessRule`, `interface ProjectDto` |
| File component | PascalCase | `ProjectList.tsx` |
| File khác | kebab-case | `api-client.ts`, `format-date.ts` |

## Cấu trúc Class/Component

### Backend Service Class

```java
package com.greytest.service.analysis;

import ...;

/**
 * Service xử lý phân tích tĩnh source code Java Spring Boot.
 * Sử dụng JavaParser để trích xuất class, method, endpoint.
 */
@Service
@Slf4j
public class AnalysisService {

    private final ProjectRepository projectRepository;
    private final JavaClassRepository classRepository;
    private final JavaParserHelper parserHelper;

    // Constructor injection - KHÔNG dùng @Autowired field
    public AnalysisService(
        ProjectRepository projectRepository,
        JavaClassRepository classRepository,
        JavaParserHelper parserHelper
    ) {
        this.projectRepository = projectRepository;
        this.classRepository = classRepository;
        this.parserHelper = parserHelper;
    }

    /**
     * Phân tích toàn bộ source code của project.
     *
     * @param projectId ID project cần phân tích
     * @return AnalysisResult chứa danh sách class, method
     * @throws ProjectNotFoundException nếu project không tồn tại
     */
    @Transactional
    public AnalysisResult analyze(Long projectId) {
        Project project = projectRepository.findById(projectId)
            .orElseThrow(() -> new ProjectNotFoundException(projectId));

        log.info("Bắt đầu phân tích project: {}", project.getName());

        List<JavaClass> classes = extractClasses(project);
        // ...

        return new AnalysisResult(classes);
    }

    // Private helpers ở dưới
    private List<JavaClass> extractClasses(Project project) {
        // ...
    }
}
```

### Frontend Component

```tsx
// features/business-rules/components/BusinessRuleList.tsx
import { useState } from 'react';
import { useBusinessRules } from '../hooks/useBusinessRules';
import { BusinessRuleItem } from './BusinessRuleItem';
import type { BusinessRule } from '../types';

interface BusinessRuleListProps {
  projectId: number;
  onApprove?: () => void;
}

export function BusinessRuleList({ projectId, onApprove }: BusinessRuleListProps) {
  const { data: rules, isLoading } = useBusinessRules(projectId);
  const [selectedId, setSelectedId] = useState<number | null>(null);

  if (isLoading) return <div>Đang tải...</div>;

  return (
    <div className="space-y-2">
      {rules?.map((rule) => (
        <BusinessRuleItem
          key={rule.id}
          rule={rule}
          isSelected={selectedId === rule.id}
          onSelect={() => setSelectedId(rule.id)}
        />
      ))}
    </div>
  );
}
```

## Quy tắc cho từng tầng

### Controller
- KHÔNG chứa business logic
- Chỉ: validate input → gọi service → trả về DTO
- KHÔNG trả Entity trực tiếp
- Mỗi endpoint < 20 dòng code

```java
// ĐÚNG
@PostMapping("/{id}/business-rules/generate")
public ResponseEntity<List<BusinessRuleDto>> generateRules(@PathVariable Long id) {
    List<BusinessRuleDto> rules = aiAgentService.generateBusinessRules(id);
    return ResponseEntity.ok(rules);
}

@PostMapping("/{id}/business-rules/review")
public ResponseEntity<BusinessRuleReviewDto> reviewRules(@PathVariable Long id) {
    BusinessRuleReviewDto review = aiAgentService.reviewBusinessRules(id);
    return ResponseEntity.ok(review);
}

// SAI - business logic trong controller
@PostMapping("/{id}/business-rules/generate")
public ResponseEntity<List<BusinessRuleDto>> generateRules(@PathVariable Long id) {
    Project p = projectRepo.findById(id).orElseThrow();
    String prompt = "Generate rules for " + p.getName();
    String response = llm.call(prompt);
    // ... 50 dòng logic
}
```

### Service
- Chứa business logic
- Mỗi method < 50 dòng (refactor nếu dài hơn)
- Inject dependency qua constructor
- Sử dụng `@Transactional` khi có thao tác DB

### Repository
- Chỉ định nghĩa interface, kế thừa `JpaRepository`
- Dùng method name query khi đơn giản
- Dùng `@Query` cho query phức tạp
- KHÔNG viết business logic ở đây

```java
public interface BusinessRuleRepository extends JpaRepository<BusinessRule, Long> {
    List<BusinessRule> findByProjectId(Long projectId);
    List<BusinessRule> findByProjectIdAndStatus(Long projectId, String status);

    @Query("SELECT br FROM BusinessRule br WHERE br.projectId = :projectId AND br.isModified = true")
    List<BusinessRule> findModifiedRules(@Param("projectId") Long projectId);
}
```

### Entity
- KHÔNG có logic, chỉ là data class
- Dùng Lombok: `@Data`, `@NoArgsConstructor`, `@AllArgsConstructor`
- Quan hệ với entity khác qua `@ManyToOne`, `@OneToMany`
- KHÔNG cascade ALL nếu không thật cần

### DTO
- Đặt tên có hậu tố `Dto` hoặc `Request`/`Response`
- Validate bằng Bean Validation (`@NotNull`, `@Size`)
- Mapping từ Entity sang DTO qua mapper class

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class BusinessRuleDto {
    private Long id;
    private String ruleCode;

    @NotBlank
    @Size(max = 1000)
    private String description;

    private String status;
    private Boolean isModified;
}
```

## Error Handling

### Custom Exception

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ProjectNotFoundException extends RuntimeException {
    public ProjectNotFoundException(Long id) {
        super("Không tìm thấy project với ID: " + id);
    }
}

@ResponseStatus(HttpStatus.BAD_REQUEST)
public class InvalidJacocoXmlException extends RuntimeException {
    public InvalidJacocoXmlException(String reason) {
        super("File JaCoCo XML không hợp lệ: " + reason);
    }
}
```

### Global Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ProjectNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ProjectNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR", "Có lỗi xảy ra"));
    }
}
```

## Logging

### Quy tắc
- Dùng `@Slf4j` của Lombok
- KHÔNG dùng `System.out.println`
- Log level:
  - `log.error` — Exception, lỗi nghiêm trọng
  - `log.warn` — Tình huống bất thường nhưng vẫn xử lý được
  - `log.info` — Sự kiện quan trọng (start/end của process dài)
  - `log.debug` — Chi tiết kỹ thuật, chỉ enable khi debug

### Ví dụ

```java
log.info("Bắt đầu sinh test plan cho project {}", projectId);
log.warn("LLM trả về response không đúng format, retry lần {}", attempt);
log.error("Không thể parse JaCoCo XML", e);
```

## Testing

### Backend
- Unit test cho Service layer (mock Repository)
- Integration test cho Controller (MockMvc)
- Tối thiểu test cho:
  - Happy path của mỗi service method
  - Error case quan trọng

### Frontend
- Test component quan trọng (BusinessRuleList, TestCaseForm...)
- Test hooks (useBusinessRules...)
- Sử dụng Vitest + React Testing Library

## Git Workflow

### Branch
- `main` — Production-ready code
- `dev` — Development branch
- `feature/<name>` — Tính năng mới
- `fix/<name>` — Sửa bug

### Commit message
Theo Conventional Commits:
- `feat: thêm chức năng sinh test plan`
- `fix: sửa lỗi parse JaCoCo XML`
- `refactor: tách AIAgentService thành nhiều file`
- `docs: cập nhật README`
- `chore: cập nhật dependency`

### PR
- Mỗi feature một PR
- Reviewer là người còn lại
- Merge sau khi review

## Code Smell cần tránh

### Backend
- Method dài > 50 dòng → tách
- Class > 300 dòng → cân nhắc tách
- Constructor > 5 tham số → refactor
- `if/else` nested > 3 level → simplify

### Frontend
- Component > 200 dòng → tách
- Props > 7 → cân nhắc dùng object
- `useState` > 5 trong 1 component → cân nhắc `useReducer`
- Inline function trong JSX phức tạp → tách ra ngoài

## DON'T

- ❌ KHÔNG dùng `Object.entries`, `Array.flat` mà không check support
- ❌ KHÔNG dùng `Stream` cho operation đơn giản (`for` loop dễ đọc hơn)
- ❌ KHÔNG abuse Lombok (`@Data` cho mọi class) — Entity nên dùng `@Getter @Setter` rõ ràng
- ❌ KHÔNG dùng `Optional` cho field (chỉ cho return type)
- ❌ KHÔNG dùng `null` check thay vì `Optional` khi có sẵn
- ❌ KHÔNG dùng `e.printStackTrace()` — luôn dùng logger
- ❌ KHÔNG commit secret/API key — dùng env variable
- ❌ KHÔNG dùng `any` type trong TypeScript (trừ trường hợp đặc biệt, có comment lý do)
