# EVALUATION.md — Phương pháp đánh giá thực nghiệm

## Mục tiêu thực nghiệm

Đánh giá hiệu quả của phương pháp **Grey-box AI QA Agent** trong việc sinh Test Plan, Test Case và Unit Test cho dự án Java Spring Boot, so sánh với các phương pháp baseline.

## Dataset (đã thu gọn)

Lựa chọn **3–4 dự án** Java Spring Boot mã nguồn mở, ưu tiên nhỏ và trung bình:

| STT | Dự án | Số Service | Số Controller | Quy mô |
|---|---|---|---|---|
| 1 | Spring PetClinic | ~15 | ~10 | Nhỏ |
| 2 | Library Management System | ~20 | ~12 | Nhỏ |
| 3 | Student Management System | ~25 | ~15 | Trung bình |
| 4 | (Optional) Self-built sample | ~10 | ~5 | Nhỏ |

**Đã loại bỏ** các dự án Lớn (E-Commerce, Hotel Booking, Order Management) để vừa sức 2 sinh viên.

### Chuẩn bị dataset

Với mỗi dự án:
1. Thu thập source code (clone về máy)
2. Xây dựng tập Business Rule từ:
   - Tài liệu yêu cầu (nếu có)
   - JavaDoc
   - API Specification
   - Chuyên gia (giảng viên/sinh viên) đánh giá thủ công
3. Đây là **Ground Truth** để so sánh
4. Với GreyTest, chạy thử hai mode Business Rule:
   - **AI auto sinh BR:** AI tự đề xuất rule từ source/context, user review và duyệt.
   - **User nhập BR + AI review:** user nhập một phần rule ban đầu, AI review/gợi ý sửa và đề xuất rule còn thiếu, user duyệt.

## Các phương pháp so sánh

### Method 1: LLM-only (Baseline)
- LLM đọc trực tiếp source code, không có Static Analysis
- Không có Business Rule
- Sinh test case dựa hoàn toàn vào code

### Method 2: Prompt-based (Baseline)
- LLM nhận source code + Business Rule
- Chỉ qua một prompt duy nhất
- Không có Human-in-the-Loop, không có pipeline nhiều bước

### Method 3: Grey-box AI QA Agent (Đề xuất)
- Static Analysis bằng JavaParser
- Business Rule có hai mode: AI auto sinh hoặc user nhập rồi AI review/gợi ý thêm
- AI Agent pipeline nhiều bước
- Human-in-the-Loop ở mỗi bước
- Traceability Matrix tự động

## Các chỉ số đánh giá (đã thu gọn)

**Đã loại bỏ:** Mutation Score (PITest) — quá phức tạp setup và chạy.

### 1. Requirement Coverage
**Định nghĩa:** Mức độ bao phủ yêu cầu nghiệp vụ.

$$\text{Req. Cov} = \frac{\text{Số Business Rule có ít nhất 1 Test Case tương ứng}}{\text{Tổng số Business Rule}} \times 100\%$$

**Cách đo:**
- Truy vấn từ Traceability Matrix
- Tính tự động sau khi sinh xong Test Case

### 2. Line Coverage
**Định nghĩa:** % dòng mã được thực thi bởi Unit Test.

$$\text{Line Cov} = \frac{\text{Số dòng mã được thực thi}}{\text{Tổng số dòng mã}} \times 100\%$$

**Cách đo:**
- User chạy `mvn test jacoco:report` ở máy
- Upload file XML lên web
- Hệ thống parse và hiển thị

### 3. Branch Coverage
**Định nghĩa:** % nhánh (if/else, switch, loop) được thực thi.

$$\text{Branch Cov} = \frac{\text{Số nhánh được thực thi}}{\text{Tổng số nhánh}} \times 100\%$$

**Cách đo:** Cùng JaCoCo XML.

### 4. Test Generation Time
**Định nghĩa:** Thời gian sinh test từ lúc nhận input đến hoàn thành.

**Đơn vị:** Giây (s)

**Cách đo:** Đo thời gian thực thi của 3 bước:
- Sinh Test Plan
- Sinh Test Case
- Sinh Unit Test

Tổng cộng = T(Plan) + T(Case) + T(Test)

### 5. User Modification Rate (UMR)
**Định nghĩa:** Mức độ user phải sửa output của AI.

$$\text{UMR} = \frac{\text{Số Test Case bị chỉnh sửa}}{\text{Tổng số Test Case được sinh}} \times 100\%$$

**Cách đo:**
- Đánh dấu `is_modified = true` khi user sửa
- Tính tỷ lệ sau khi user approve

**Giá trị càng thấp càng tốt** (AI sinh càng đúng).

### 6. Token Cost
**Định nghĩa:** Tổng số token tiêu thụ.

**Cách đo:**
- Đếm Input tokens + Output tokens cho mỗi LLM call
- Lưu vào `experiment_metric.input_tokens` và `output_tokens`

**Mục đích:** Đánh giá chi phí vận hành.

### 7. Stability Score
**Định nghĩa:** Độ ổn định giữa các lần chạy với cùng input.

$$\text{Stability} = \frac{\text{Số Test Case giống nhau giữa các lần chạy}}{\text{Tổng số Test Case}} \times 100\%$$

**Cách đo:**
- Chạy 5 lần trên cùng project
- So sánh test case giữa các lần
- "Giống nhau" = description similarity > 0.85 + cùng test_type

**Giá trị càng cao càng tốt** (ít phụ thuộc tính ngẫu nhiên LLM).

### 8. Traceability Score
**Định nghĩa:** Khả năng truy vết.

$$\text{Traceability} = \frac{\text{Số Test Case có liên kết đầy đủ về Business Rule}}{\text{Tổng số Test Case}} \times 100\%$$

**Liên kết hợp lệ:** Business Rule → Test Plan → Test Case → Unit Test (đủ chuỗi)

## Kỳ vọng kết quả

Dự kiến phương pháp Grey-box sẽ vượt trội ở các chỉ số sau:

| Method | Req. Cov | Line Cov | Branch Cov | Time (s) | UMR (%) | Tokens | Stability |
|---|---|---|---|---|---|---|---|
| LLM-only | 71% | 78% | 65% | 45 | 28 | 120k | 72 |
| Prompt-based | 84% | 82% | 73% | 58 | 19 | 165k | 79 |
| **Grey-box (Đề xuất)** | **95%** | **89%** | **84%** | 62 | 8 | 92k | 93 |

*Số liệu trên là kỳ vọng, sẽ được xác nhận qua thực nghiệm thật.*

### Lý giải kỳ vọng

- **Req. Cov cao** vì có Business Rule rõ ràng, mỗi rule đều được sinh test
- **Line/Branch Cov cao** vì có Static Analysis hỗ trợ AI hiểu cấu trúc code
- **Time cao hơn một chút** vì pipeline nhiều bước, có HITL
- **UMR thấp** vì AI dùng cả context code và rule → đề xuất sát thực tế
- **Tokens thấp hơn Prompt-based** vì chia nhỏ bước, mỗi bước context ngắn hơn
- **Stability cao** vì Static Analysis giảm độ mơ hồ cho LLM

## So sánh chi tiết per-project

```
| Project    | Method        | Req | Line | Branch |
| PetClinic  | LLM-only      | 72  | 80   | 66     |
| PetClinic  | Prompt-based  | 86  | 83   | 75     |
| PetClinic  | Grey-box      | 96  | 91   | 85     |
| Library    | LLM-only      | ... | ...  | ...    |
| ...        | ...           | ... | ...  | ...    |
```

## Quy trình thực nghiệm cụ thể

### Bước 1: Setup môi trường
- Cài đặt cả 3 method trong cùng hệ thống (có flag `--method=llm-only|prompt-based|grey-box`)
- Hoặc viết script chạy 3 method riêng biệt

### Bước 2: Chuẩn bị Ground Truth
- Mỗi dự án: review thủ công, xây dựng tập Business Rule chuẩn
- Lưu vào file (JSON/Markdown) để tham chiếu

### Bước 3: Chạy thực nghiệm
- Với mỗi (project, method): chạy 5 lần
- Lưu kết quả vào `experiment_metric`
- Tính trung bình của 5 lần

### Bước 4: Phân tích
- Vẽ biểu đồ so sánh (bar chart cho từng metric)
- Tính độ chênh lệch giữa các method
- Viết phân tích vào báo cáo

### Bước 5: Đánh giá định tính
- Phân tích ưu điểm, hạn chế của mỗi method
- Phân tích các case mà Grey-box thua các baseline (nếu có)
- Phân tích khả năng ứng dụng thực tế

## Báo cáo

Phần thực nghiệm trong báo cáo đồ án cần có:

1. **Mô tả dataset** — Bảng các dự án + đặc điểm
2. **Mô tả các phương pháp so sánh** — Vẽ pipeline từng method
3. **Kết quả tổng thể** — Bảng so sánh + biểu đồ
4. **Kết quả per-project** — Bảng chi tiết
5. **Phân tích định lượng** — Lý giải các con số
6. **Phân tích định tính** — Ưu/nhược điểm
7. **Threats to validity** — Hạn chế của thực nghiệm
