# Ghi chú phân tích · Day 04

Họ và tên: Lê Khánh Cường
MSSV: 2A202603999

## 1. Case được chọn

- ID case: `H1`
- Câu trả lời của agent: Agent đọc incident GitHub `h02n9572mx9y` và Cloudflare
  `yaa0witbl2me`, đối chiếu đúng thời gian, trạng thái và component, đồng thời không
  suy luận hai incident có chung nguyên nhân. Tuy nhiên, khi update GitHub dẫn tới
  incident gộp `xr3myzj7ms51`, agent kết luận rằng không thể đọc incident này và
  dừng điều tra, dù ID đã có thể dùng trực tiếp với `get_incident_details`.
- Điểm ở lần chạy 1: **60/100** (`factuality: 25/40`, `completeness: 13/25`,
  `evidence: 10/20`, `boundaries: 12/15`). Điểm raw cũng là 60 và bị mechanical
  cap 60 vì tool calls chỉ xuất hiện ở một model step.

## 2. Bằng chứng trong trace

Dữ liệu của tool đầu tiên đã cung cấp cả URL và ID của incident liên quan:

```text
step 1 · get_incident_details(github, h02n9572mx9y)
result.updates[0].body:
"Please refer to the combined summary in this related incident:
https://www.githubstatus.com/incidents/xr3myzj7ms51"

step 1 · get_incident_details(cloudflare, yaa0witbl2me)
result.status: resolved

step 2 · model finish=stop, tool_calls=0
```

Trong final answer, agent lại viết: "Tool của tôi chỉ đọc trực tiếp theo ID, và ID
này khác với `h02n9572mx9y`, nên tôi không thể đọc nội dung nguyên nhân trong
incident gộp đó". Lập luận này sai vì `xr3myzj7ms51` chính là một ID hợp lệ và có
thể được truyền vào cùng tool. Do agent trả final answer ngay ở step 2, trace chỉ có
tool calls ở step 1. Mechanical check ghi nhận `tool_calls: 2`, `tool_steps: 1` và
vi phạm "Cần tool calls ở ít nhất 2 model steps khác nhau", nên score bị cap ở 60.

Nguyên nhân có thể nằm ở khả năng lập kế hoạch nhiều bước của model/system prompt:
prompt chưa nhấn mạnh rằng khi một incident trỏ tới incident khác, agent cần tách ID
từ liên kết rồi tiếp tục gọi `get_incident_details` trước khi kết luận thiếu dữ liệu.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Trong `agent.py`, bổ sung chỉ dẫn vào `SYSTEM_PROMPT`: khi update
  của một incident dẫn tới incident khác thuộc provider được hỗ trợ, agent phải tách
  ID từ URL và đọc incident liên quan trước khi kết luận thiếu dữ liệu. Prompt cũng
  yêu cầu báo rõ partial failure và không biến timeout thành trạng thái dịch vụ.
  Ngoài ra, vòng lặp lưu URL từ mọi lần `get_incident_details` thành công. Trước khi
  trả `completed`, nếu final answer thiếu một URL đã quan sát, agent ghi event
  `validation_missing_source`, yêu cầu model sửa câu trả lời trong budget còn lại;
  nếu hết `max_steps` thì trả `max_steps` thay vì chấp nhận answer thiếu nguồn.
- Vì sao chọn thay đổi này: H1 là case thấp nhất (60/100), lỗi có bằng chứng trực
  tiếp trong trace và thay đổi nhắm đúng vào thiếu sót multi-step investigation.
  Evidence gate còn đáp ứng contract kiểm tra nguồn trong `tests/test_validation.py`.
  Sau thay đổi, toàn bộ 40 unit tests pass. Ở eval lần 2, H1 gọi thêm
  `get_incident_details(github, xr3myzj7ms51)` tại step 2, sau đó dẫn đủ URL của ba
  incident và tăng từ 60 lên 95 điểm.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 93.32/100 | 95.05/100 |
| Điểm case đã chọn | 60/100 | 95/100 |
| Số model calls | 55 calls trên 22 case; riêng H1 có 2 calls | 57 calls trên 22 case; riêng H1 có 4 calls |
| Token, chi phí nếu có | Agent: 57,561 input + 13,456 output; judge: 87,021 input + output; provider không trả chi phí | Agent: 84,717 input + 15,449 output; judge: 82,963 input + output; provider không trả chi phí |

- Lỗi đã giải quyết: H1 đã theo liên kết từ `h02n9572mx9y` tới incident gộp
  `xr3myzj7ms51`, lấy được nguyên nhân và phạm vi ảnh hưởng GitHub, đối chiếu với
  Cloudflare mà không suy luận nguyên nhân chung. H1 tăng **35 điểm** (60 → 95).
  H7 cũng tăng 73 → 100 nhờ báo đúng timeout Cloudflare thay vì khẳng định trạng
  thái khi thiếu dữ liệu. Điểm tổng tăng **1.73 điểm** (93.32 → 95.05).
- Lỗi còn lại hoặc mới xuất hiện: A2 còn quy trạng thái degraded của component cho
  riêng incident FedRAMP quá chắc chắn (100 → 91). A5 nói tool chỉ trả 5 incident
  nhưng chưa phân biệt rõ 11 incident khớp với 5 mục được hiển thị (100 → 93).
  H4 giảm 91 → 81 vì chất lượng bài thơ không ổn định; case này không gọi tool và
  mức giảm chủ yếu phản ánh biến thiên output/judge, không liên quan evidence gate.
  Ở H1, judge vẫn lưu ý update resolved của `xr3myzj7ms51` bị cắt và nên nêu giới
  hạn này ngay trong phần nguyên nhân.

## 5. Giới hạn

- Điểm judge là đánh giá xấp xỉ của LLM và có thể thiên lệch vì agent và judge cùng
  dùng `nghi/claude-haiku-5.5`. Cần đối chiếu `expected`, answer, trace và mechanical
  checks; `completed` không đồng nghĩa câu trả lời đúng.
- Snapshot chỉ chứa tối đa 50 incident gần nhất mỗi provider và một số update bị
  lược bớt/cắt ngắn. Kết quả snapshot ngày 09/11/2026 không đại diện cho trạng thái
  dịch vụ hiện tại ngoài thực tế; không tìm thấy cũng không chứng minh incident
  không tồn tại.
- Mỗi phiên bản chỉ chạy eval một lần nên chênh lệch điểm trước/sau chưa chứng minh
  độ ổn định. Kết quả có thể thay đổi do model không deterministic, provider timeout
  hoặc judge trả đánh giá khác giữa các lần chạy.
