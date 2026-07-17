# TECH_STACK.md — Stack công nghệ

## Tổng quan

| Tầng | Công nghệ | Lý do chọn |
|---|---|---|
| Frontend | React 18 + TypeScript | UI phức tạp, ecosystem mạnh |
| State Management | TanStack Query (React Query) | Đơn giản, đủ cho server state |
| UI Library | TailwindCSS + shadcn/ui | Build UI nhanh, không cần thiết kế từ đầu |
| Backend | Spring Boot 3 + Java 17 | JavaParser chạy tự nhiên, phù hợp ngành KTPM |
| Auth | Spring Security + BCrypt + JWT/session token | Đăng nhập và phân quyền USER/ADMIN ở mức vừa đủ cho đồ án |
| ORM | Spring Data JPA + Hibernate | Chuẩn Spring, ít boilerplate |
| Database | PostgreSQL 15+ | Quan hệ rõ ràng, ổn định, miễn phí |
| Code Analysis | JavaParser 3.x | Thư viện chuẩn cho phân tích Java AST |
| LLM | Mock local, OpenAI GPT-4o hoặc Google Gemini | Có mock để demo/test nhanh; provider thật cấu hình qua env |
| API Doc | SpringDoc OpenAPI (Swagger) | Tự sinh doc từ code |
| Build (BE) | Maven | Chuẩn Spring Boot |
| Build (FE) | Vite | Nhanh, hiện đại |
| Container | Docker + Docker Compose | Deploy đồng nhất |
| Version Control | Git + GitHub | Tiêu chuẩn |

## Chi tiết Frontend

### Dependencies chính

```json
{
  "react": "^18.3.0",
  "react-dom": "^18.3.0",
  "react-router-dom": "^6.x",
  "@tanstack/react-query": "^5.x",
  "axios": "^1.x",
  "tailwindcss": "^3.x",
  "lucide-react": "latest"
}
```

### Cấu trúc thư mục

```
frontend/
├── public/
├── src/
│   ├── features/
│   │   ├── auth/
│   │   ├── projects/
│   │   ├── business-rules/
│   │   ├── test-plans/
│   │   ├── test-cases/
│   │   ├── unit-tests/
│   │   ├── traceability/
│   │   └── coverage/
│   ├── shared/
│   │   ├── api/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── utils/
│   ├── App.tsx
│   └── main.tsx
├── package.json
├── vite.config.ts
├── tailwind.config.js
└── tsconfig.json
```

## Chi tiết Backend

### Dependencies chính (pom.xml)

```xml
<dependencies>
    <!-- Spring Boot Starters -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- Database -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>

    <!-- Flyway - migration schema (version do Spring Boot quản lý) -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-database-postgresql</artifactId>
    </dependency>

    <!-- JavaParser - Phân tích AST Java -->
    <dependency>
        <groupId>com.github.javaparser</groupId>
        <artifactId>javaparser-symbol-solver-core</artifactId>
        <version>3.26.0</version>
    </dependency>

    <!-- LLM Client: dùng java.net.http.HttpClient có sẵn trong JDK, không thêm SDK ngoài. -->

    <!-- API Doc -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.5.0</version>
    </dependency>

    <!-- Utility -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- ZIP handling: dùng java.util.zip (stdlib) cho ZIP chuẩn, không cần thêm
         commons-compress. Chỉ thêm commons-compress nếu sau này cần định dạng khác
         (tar, 7z). -->

    <!-- GitHub clone -->
    <dependency>
        <groupId>org.eclipse.jgit</groupId>
        <artifactId>org.eclipse.jgit</artifactId>
        <version>6.9.0.202403050737-r</version>
    </dependency>

    <!-- Testing -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Cấu trúc thư mục

```
backend/
├── src/
│   ├── main/
│   │   ├── java/com/greytest/
│   │   │   ├── GreytestApplication.java
│   │   │   ├── config/             # Config classes
│   │   │   ├── controller/         # REST controllers
│   │   │   ├── service/
│   │   │   │   ├── auth/           # AuthService, token/session handling
│   │   │   │   ├── analysis/       # AnalysisService
│   │   │   │   ├── agent/          # AIAgentService
│   │   │   │   ├── traceability/   # TraceabilityService
│   │   │   │   └── coverage/       # CoverageService
│   │   │   ├── repository/         # JPA repositories
│   │   │   ├── entity/             # JPA entities
│   │   │   ├── dto/                # DTO classes
│   │   │   ├── mapper/             # Entity ↔ DTO mappers
│   │   │   ├── exception/          # Custom exceptions
│   │   │   └── util/               # Utility classes
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── prompts/            # Prompt templates
│   │       │   ├── business-rule.md
│   │       │   ├── test-plan.md
│   │       │   ├── test-case.md
│   │       │   └── unit-test.md
│   │       └── db/migration/       # Flyway migrations
│   └── test/
├── pom.xml
└── Dockerfile
```

## Cấu trúc dự án tổng thể

```
greytest/
├── frontend/              # React app
├── backend/               # Spring Boot app
├── docs/                  # Tài liệu đồ án
│   ├── CLAUDE.md
│   ├── PROJECT_OVERVIEW.md
│   ├── ARCHITECTURE.md
│   ├── TECH_STACK.md
│   ├── WORKFLOW.md
│   ├── DATA_MODEL.md
│   ├── AI_AGENT.md
│   ├── EVALUATION.md
│   └── CODING_RULES.md
├── docker-compose.yml     # Chạy cả hệ thống
├── .gitignore
└── README.md
```

## Docker Compose

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: greytest
      POSTGRES_USER: greytest
      POSTGRES_PASSWORD: greytest_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  backend:
    build: ./backend
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/greytest
      SPRING_DATASOURCE_USERNAME: greytest
      SPRING_DATASOURCE_PASSWORD: greytest_password
      LLM_PROVIDER: ${LLM_PROVIDER:-mock}
      LLM_API_KEY: ${LLM_API_KEY}
      LLM_MODEL: ${LLM_MODEL:-gpt-4o-mini}
      GREYTEST_AI_CONTEXT_LOG_ENABLED: ${GREYTEST_AI_CONTEXT_LOG_ENABLED:-false}
      GREYTEST_AI_CONTEXT_LOG_CONSOLE_ENABLED: ${GREYTEST_AI_CONTEXT_LOG_CONSOLE_ENABLED:-false}
      GREYTEST_AI_CONTEXT_LOG_PATH: /app/log
    depends_on:
      - postgres

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend

volumes:
  postgres_data:
```

## Lý do KHÔNG dùng các công nghệ phổ biến khác

| Công nghệ | Lý do không chọn |
|---|---|
| Node.js Backend | JavaParser không chạy native trên Node, phải tách service phụ |
| MongoDB | Dữ liệu có quan hệ phức tạp (BR-Plan-Case-Test), SQL phù hợp hơn |
| Redux | Quá phức tạp cho đề tài 2 người, TanStack Query đủ dùng |
| Microservices | Over-engineering cho đề tài này, monolith đủ |
| Kubernetes | Quá phức tạp, Docker Compose là đủ |
| GraphQL | REST API đơn giản và đủ cho use case này |
