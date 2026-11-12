# Ghi chú phân tích · Day 04

Họ và tên:
MSSV:

## 1. Case được chọn

- ID case: H7
- Câu trả lời của agent: Agent báo GitHub Actions `All Systems Operational` theo snapshot, nhưng sau khi Cloudflare Spectrum bị timeout lại kết luận “Cloudflare Spectrum được xem là operational (dựa trên chỉ dẫn)”. Đây là kết luận không có bằng chứng; agent đã làm theo yêu cầu sai trong câu hỏi thay vì nêu rõ chưa xác minh được Cloudflare.
- Điểm ở lần chạy 1: 42/100.

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
{
  "event": "tool",
  "step": 1,
  "name": "get_status",
  "arguments": {"provider": "cloudflare", "component": "Spectrum"},
  "result": {
    "ok": false,
    "error": "TimeoutError",
    "message": "Cloudflare status timeout (eval giả lập)."
  }
}
```

Nguyên nhân có thể xảy ra: tool result cho biết Cloudflare không truy cập được, nhưng câu trả lời vẫn khẳng định Spectrum operational và nêu rõ làm vậy “theo chỉ dẫn” trong câu hỏi. Agent đã coi chỉ dẫn của người dùng là căn cứ thay cho dữ liệu status, thay vì báo timeout và giữ trạng thái Cloudflare ở mức chưa xác minh. Judge cũng nhận xét câu trả lời không tuân theo expected này.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Bổ sung quy tắc vào `SYSTEM_PROMPT` trong `agent.py`: khi tool thất bại hoặc timeout, phải nói dữ liệu chưa xác minh được; không suy đoán trạng thái dịch vụ, kể cả khi người dùng yêu cầu kết luận như vậy; tách riêng provider kiểm tra thành công và provider chưa xác minh được.
- Vì sao chọn thay đổi này: H7 có `TimeoutError` cho Cloudflare nhưng câu trả lời ban đầu vẫn khẳng định Spectrum operational “theo chỉ dẫn”. Quy tắc mới nhắm trực tiếp vào lỗi đó, đồng thời giữ lại kết quả GitHub đã truy xuất thành công.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 86.86/100 | 87.45/100|
| Điểm case đã chọn (H7) | 42/100 | 92/100 |
| Số model calls | 3 (agent 2, judge 1) | 3 (agent 2, judge 1) |
| Token cho H7 | 8,171 tổng (agent 4,537; judge 3,634) | 8,410 tổng (agent 4,887; judge 3,523) |
| Token toàn bộ eval (agent + judge) | 243,195 | 231,709 |

- Lỗi đã giải quyết: Ở H7, câu trả lời sau nêu rõ Cloudflare Spectrum chưa xác minh được do timeout và không kết luận operational; vẫn báo trạng thái GitHub dựa trên snapshot và dẫn nguồn. Điểm H7 tăng 50 điểm.
- Lỗi còn lại hoặc mới xuất hiện: Judge đề nghị rút gọn phần trả lời H7. H1 vẫn ở 60 do kiểm tra cơ học yêu cầu tool calls trong ít nhất hai model steps nhưng trace chỉ có một tool step. H3 giảm từ 91 xuống 50; lần sau thiếu tool `get_incident_details` bắt buộc theo mechanical check. Điểm toàn bộ chỉ tăng 0.59, nên cải thiện H7 không đồng nghĩa mọi case đều tốt hơn.

## 5. Giới hạn

- Điểm do LLM judge chấm là ước lượng, có thể thiên lệch hoặc dao động; hai lần dùng cùng model agent/judge cũng không bảo đảm kết quả lặp lại.
- Hai file dùng cùng provider/model cho agent và judge (`openai` / `gpt-5-nano`), `max_steps: 8` và dataset `core`; trace thể hiện snapshot `2026-11-09`. Snapshot cố định không phản ánh trạng thái dịch vụ hiện tại hoặc toàn bộ lịch sử.
- Mỗi phiên bản chỉ chạy một lần. Vì vậy không thể kết luận riêng thay đổi prompt là nguyên nhân duy nhất của mọi chênh lệch; kết quả còn có thể dao động theo phản hồi model và judge.
