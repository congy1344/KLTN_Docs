# GreyTest — Tài liệu đồ án

> Hệ thống AI QA Agent theo hướng Grey-box hỗ trợ sinh Test Plan, Test Case và Unit Test cho dự án Java Spring Boot.

## Đây là gì?

Đây là bộ tài liệu thiết kế cho đồ án tốt nghiệp ngành Kỹ thuật phần mềm của 2 sinh viên. Bộ tài liệu này được dùng để:

1. **Tham chiếu khi viết code** — Mở khi đang implement
2. **Đưa vào Claude Code** — Giúp Claude hiểu bối cảnh và viết code đúng ý
3. **Cơ sở để viết báo cáo đồ án** — Toàn bộ thiết kế đã có sẵn

## Cấu trúc tài liệu

| File | Nội dung | Ai cần đọc |
|---|---|---|
| `CLAUDE.md` | Hướng dẫn cho Claude Code | Claude Code (đọc đầu tiên) |
| `PROJECT_OVERVIEW.md` | Mục tiêu, phạm vi đề tài | Tất cả |
| `ARCHITECTURE.md` | Kiến trúc hệ thống | Dev (cả FE và BE) |
| `TECH_STACK.md` | Công nghệ sử dụng | Dev |
| `WORKFLOW.md` | Luồng hoạt động end-to-end | Tất cả |
| `DATA_MODEL.md` | Database schema | Backend dev |
| `AI_AGENT.md` | Chi tiết AI Agent + prompt | Backend dev |
| `EVALUATION.md` | Phương pháp đánh giá | Tất cả (khi làm thực nghiệm) |
| `CODING_RULES.md` | Quy tắc code | Dev |
| `../analysis_ft.md` | Bàn giao Phase 3 Static Analysis: phạm vi, API, golden validation, limitation | Cả hai thành viên |

## Tóm tắt nhanh

### Đề tài
GreyTest là hệ thống web app cho phép user upload project Java Spring Boot, sau đó AI tự động sinh ra Test Plan, Test Case và Unit Test có chất lượng cao nhờ:
- **Static Analysis** đọc cấu trúc code (white-box)
- **Business Rule** mô tả nghiệp vụ (black-box)
- **Existing Test Context** đọc test sẵn để cải thiện/bổ sung thay vì bỏ qua hoàn toàn
- **Kết hợp cả 2** = Grey-box
- **Human-in-the-Loop** cho user review từng bước
- **Đăng nhập và phân quyền** để user chỉ thao tác với project của mình, admin quản lý toàn hệ thống

### Tech Stack
- **Frontend:** React + TypeScript + TailwindCSS + TanStack Query
- **Backend:** Spring Boot 3 + Java 17
- **Database:** PostgreSQL 15
- **AI:** OpenAI GPT-4o hoặc Anthropic Claude
- **Code Analysis:** JavaParser
- **Deploy:** Docker Compose

### Phân công 2 người
- **Người 1:** Backend (Analysis, AI Agent, API)
- **Người 2:** Frontend (toàn bộ UI + tích hợp API)
- **Cả 2:** Database, Traceability, Coverage, thực nghiệm

## Cách dùng với Claude Code

1. Copy toàn bộ folder `docs/` vào root của project
2. Khi mở Claude Code, nó sẽ tự đọc `CLAUDE.md` đầu tiên
3. Yêu cầu Claude đọc thêm các file cần thiết khi bắt đầu task mới
4. Ví dụ prompt:
   ```
   Đọc docs/ARCHITECTURE.md và docs/DATA_MODEL.md.
   Sau đó implement entity Project và ProjectRepository theo specification.
   ```

## Cập nhật tài liệu

- Khi có thay đổi lớn về thiết kế → cập nhật file tương ứng
- Khi có quyết định kiến trúc mới → thêm vào `ARCHITECTURE.md`
- Khi có quy ước code mới → thêm vào `CODING_RULES.md`

Tài liệu là **single source of truth** cho dự án.
