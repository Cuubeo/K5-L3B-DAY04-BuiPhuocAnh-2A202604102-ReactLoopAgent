# Ghi chú phân tích · Day 04

Họ và tên: Bùi Phước Anh
MSSV: 2A202604102

## 1. Case được chọn

- ID case: **H1**: điều tra incident GitHub `h02n9572mx9y`, theo liên kết do incident cung cấp, đối chiếu với Cloudflare `yaa0witbl2me`.
- Câu trả lời của agent: tóm tắt đúng trạng thái, thời gian, component của `h02n9572mx9y` (Actions) và `yaa0witbl2me` (Workers Builds), kết luận không có bằng chứng chung nguyên nhân. Nhưng agent **không đọc incident được dẫn tới `xr3myzj7ms51`**, nên không nêu được nguyên nhân/ảnh hưởng phía GitHub mà câu hỏi yêu cầu.
- Điểm ở lần chạy 1: **60/100** (bị giới hạn bởi `score_cap = 60` của kiểm tra bằng code). Lỗi này lặp lại ở cả hai lần chạy (55 rồi 60); H6 có cùng lỗi (60).

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```text
{'event': 'model', 'step': 1, 'finish': 'tool_calls', 'tool_calls': ['get_incident_details', 'get_incident_details']}
TOOL step 1 get_incident_details {"provider": "github", "incident_id": "h02n9572mx9y"}
   resolved   2026-11-04T18:16:08Z  Please refer to the combined summary in this related incident:
                                    https://www.githubstatus.com/incidents/xr3myzj7ms51
   monitoring 2026-11-04T18:07:51Z  The degradation affecting Actions has been mitigated...
TOOL step 1 get_incident_details {"provider": "cloudflare", "incident_id": "yaa0witbl2me"}
   resolved   2026-11-05T20:20:10Z  Queue times have returned to normal.
{'event': 'model', 'step': 2, 'finish': 'stop', 'tool_calls': []}
{'event': 'final', 'step': 2, 'answer_chars': 1910}

grade.mechanical.violations: ['Cần tool calls ở ít nhất 2 model steps khác nhau.']
grade.improvement: Cần cung cấp thông tin chi tiết từ incident xr3myzj7ms51 và mô tả nguyên nhân...
```

Nguyên nhân có thể xảy ra:
- Lỗi nằm ở **prompt + loop**, không phải giới hạn dữ liệu: tool result có sẵn URL của `xr3myzj7ms51`, ID này nằm trong 50 incident gần nhất nên đọc được.
- Model gọi song song hai tool ở step 1, thấy đủ dữ liệu "bề mặt" và trả lời luôn ở step 2. System prompt không yêu cầu theo liên kết sang incident khác.
- Loop chấp nhận mọi `finish == 'stop'` là hoàn thành, không có bước nào kiểm tra câu trả lời trước khi chốt.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu (`agent.py`):
  1. **System prompt**: thêm chỉ dẫn "nếu update dẫn tới incident khác (`/incidents/<id>`) thì gọi `get_incident_details` cho incident đó trước khi kết luận" và "dẫn đúng URL từ trường `url`".
  2. **Theo dõi bằng chứng** (`_observe`): sau mỗi `get_incident_details` thành công, lưu URL nguồn (`url` và `.../incidents/<id>`), các incident đã đọc, và các incident được nhắc tới trong update body (regex `githubstatus.com|cloudflarestatus.com/incidents/<id>`).
  3. **Kiểm tra trước khi chốt** (`_check_answer`), chạy khi model trả `finish == 'stop'`:
     - Còn incident được dẫn tới mà chưa đọc, và còn ≥ 2 bước → ghi event `validation_unfollowed_reference`, gửi yêu cầu đọc tiếp cho model (chỉ nhắc 1 lần để tránh lặp).
     - Đã đọc incident thành công nhưng câu trả lời không chứa URL nào của incident đó → ghi event `validation_missing_source`, yêu cầu model sửa; hết `max_steps` thì trả `max_steps`, không báo `completed`. Tool lỗi thì không bắt buộc URL.
- Vì sao chọn thay đổi này:
  - Lỗi lặp lại ở 2 lần chạy và ở 2 case (H1, H6), có bằng chứng từ kiểm tra bằng code chứ không chỉ từ judge.
  - Chỉ sửa prompt thì model vẫn có thể bỏ qua; thêm kiểm tra trong loop biến yêu cầu "theo liên kết" thành điều kiện có thể kiểm tra được.
  - Kiểm tra URL nguồn đáp ứng contract trong `tests/test_validation.py`. Lưu ý: đây chỉ kiểm tra *sự có mặt* của URL, không chứng minh mọi claim đều có nguồn.
  - Kết quả offline: `python -m unittest discover -s tests -v` → 37/37 test qua.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng |  |  |
| Điểm case đã chọn |  |  |
| Số model calls |  |  |
| Token, chi phí nếu có |  |  |

- Lỗi đã giải quyết:
- Lỗi còn lại hoặc mới xuất hiện:

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.
