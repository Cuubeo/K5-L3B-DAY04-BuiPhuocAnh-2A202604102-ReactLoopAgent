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
Cùng cấu hình ở cả hai lần: agent và judge `openrouter / openai/gpt-4o-mini`, `max_steps = 8`, dataset `core`, snapshot cố định (khối `config` giống hệt nhau).

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 78.14 (22/22 case đã chấm) | 79.77 (22/22 case đã chấm) |
| Điểm case đã chọn | H1: 60 (H6: 60) | H1: 80 (H6: 75) |
| Số model calls (agent) | 52 | 54 |
| Token, chi phí nếu có | agent: 33.510 prompt + 9.411 completion; ≈ $0.0104 (judge: 42.062 token) | agent: 42.124 prompt + 9.473 completion; ≈ $0.0113 (judge: 39.414 token) |

- Lỗi đã giải quyết:
  - **H1**: trace lần 2 có event `validation_unfollowed_reference` ở step 2 → model gọi tiếp `get_incident_details(github, xr3myzj7ms51)` → trả lời ở step 4. Vi phạm "Cần tool calls ở ít nhất 2 model steps khác nhau" không còn, `score_cap` từ 60 lên 100, điểm 60 → 80.
  - **H6**: cùng cơ chế, đọc thêm `xr3myzj7ms51`, hết vi phạm, điểm 60 → 75.
  - Không case nào bị `max_steps` vì kiểm tra mới (chỉ nhắc khi còn ≥ 2 bước).
- Lỗi còn lại hoặc mới xuất hiện:
  - H1 vẫn chưa 100: judge nhận xét chưa kết luận rõ hai incident có liên quan hay không và chưa nêu dữ liệu có thể bị cắt.
  - **H2: 100 → 50 (mới)**: lần này model chỉ gọi `get_incident_details` mà không gọi `get_status`, nên kiểm tra bằng code báo "Chưa gọi tool cần thiết: get_status". Thay đổi của tôi không chặn điều này; có thể chỉ dẫn mới trong prompt khiến model ưu tiên đọc incident, hoặc chỉ là dao động giữa các lần chạy, cần chạy thêm để kết luận.
  - A5 vẫn thấp (60): vẫn khẳng định "không có incident năm 2025" từ kết quả tìm kiếm giới hạn trong 50 incident.
  - Chi phí: prompt token của agent tăng khoảng 26% (thêm lượt nhắc và tool result của incident được dẫn tới), số model calls +2.

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.

- **Điểm judge**: judge dùng cùng model với agent (gpt-4o-mini), có thể thiên lệch và nhiễu. Ví dụ: lần chạy đầu tiên của eval_before, judge trả JSON lỗi ở E1 (case bị `unscored`, phải chạy lại); ở A4 judge trừ điểm URL `stspg.io/pjgptg3cfc6n` dù `expected` chấp nhận URL này; H4 (bài thơ, không đổi gì liên quan) dao động 100 → 80.
- **Chạy một lần mỗi phiên bản**: chính eval_before chạy hai lần với cùng code và cấu hình đã cho điểm khác nhau nhiều (E4 75 → 95, A2 70 → 100, A5 55 → 35). Vì vậy chênh lệch tổng +1.63 điểm (78.14 → 79.77) **nằm trong mức nhiễu**, không chứng minh agent tốt hơn ổn định. Bằng chứng đáng tin hơn là thay đổi cơ học trong trace của H1/H6 (có thêm tool step, hết vi phạm `min_tool_steps`).
- **Snapshot**: dữ liệu chụp lúc `2026-11-09T03:50Z`, chỉ 50 incident gần nhất mỗi provider; kết quả không phản ánh trạng thái hiện tại, và tìm không thấy không chứng minh không có incident.
- **Giới hạn của thay đổi**: chỉ theo liên kết dạng `githubstatus.com|cloudflarestatus.com/incidents/<id>` (không theo shortlink `stspg.io` hay mô tả bằng chữ), chỉ nhắc một lần; kiểm tra URL chỉ xác nhận có ít nhất một URL incident trong câu trả lời, không chứng minh mọi claim đều có nguồn.
