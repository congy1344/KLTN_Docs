# Theo dõi token LLM trong môi trường dev

Tính năng này mặc định tắt, không có API và không hiển thị trên frontend. File metric chỉ chứa số token, model, stage và latency; không chứa prompt, source code hoặc API key.

## Bật cho một lần chạy

Đặt các biến sau trong `backend/.env` trước khi khởi động backend:

```properties
LLM_USAGE_ENABLED=true
LLM_USAGE_RUN_ID=piggymetrics-luna-run-01
LLM_USAGE_METHOD=GREYTEST
```

Sau mỗi LLM response, backend cập nhật:

```text
log/llm-usage/<RUN_ID>/calls.jsonl
log/llm-usage/<RUN_ID>/summary.json
```

- `calls.jsonl`: một JSON object trên mỗi API call thành công, phù hợp để tail hoặc xử lý bằng script.
- `summary.json`: tổng token toàn run và breakdown theo Business Rule, Test Plan, Test Case, Unit Test.
- `usageSource=PROVIDER`: số liệu lấy trực tiếp từ response API.
- `usageSource=ESTIMATED`: provider không trả usage; GreyTest ước lượng gần đúng theo độ dài text.

## So sánh model trong cùng luồng

Giữ nguyên project snapshot, prompt, temperature, max token và batch size. Chạy tuần tự, mỗi model dùng một `RUN_ID` riêng:

```text
piggymetrics-luna-run-01
piggymetrics-gemini-run-01
piggymetrics-luna-run-02
piggymetrics-gemini-run-02
```

So sánh các file `summary.json`. Nên chạy ít nhất 5 lần cho mỗi cấu hình và báo cáo mean, median cùng độ lệch chuẩn.

## Xem khi đang chạy

PowerShell:

```powershell
Get-Content ..\log\llm-usage\piggymetrics-luna-run-01\calls.jsonl -Wait
```

`summary.json` được ghi lại sau mỗi response nên có thể mở hoặc refresh trong editor trong lúc pipeline đang chạy.
