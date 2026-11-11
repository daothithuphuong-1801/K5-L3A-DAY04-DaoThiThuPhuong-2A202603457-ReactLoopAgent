# Ghi chú phân tích · Day 04

Họ và tên: Đào Thị Thu Phương
MSSV: 2A202603457

## 1. Case được chọn

- ID case: H3
- Câu trả lời của agent: Agent không xác nhận incident đã xảy ra hoặc Cloudflare sập trên toàn thế giới. Agent gọi get_incident_details với ID totally-not-a-real-id; tool trả về INCIDENT_NOT_FOUND vì không tìm thấy ID trong 50 incident gần nhất. Agent giải thích rằng kết quả này không chứng minh incident chưa từng tồn tại, không tự tạo link và đề nghị người dùng cung cấp ID hoặc thời gian chính xác hơn.
- Điểm ở lần chạy 1: (eval_before.json): 50/100.

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```
Trace lần chạy 1 – trước khi sửa
step 1: model -> tool_calls
  tool: get_status
  arguments:
    provider: cloudflare
    component: "incident totally-not-a-real-id"

tool result:
  matching_incidents_in_last_50: 0
  note: không tìm thấy trong 50 incident gần nhất không chứng minh
        incident chưa từng tồn tại

step 2: model -> finish
  Agent trả lời không thể xác nhận ID, nhưng không gọi
  get_incident_details.

Nguyên nhân có thể xảy ra: Agent hiểu ID incident như tên component và gọi get_status thay vì dùng công cụ tra cứu chi tiết incident. Vì vậy, agent chưa thực hiện đúng bước xác minh bắt buộc. Judge ghi nhận lỗi chưa gọi get_incident_details, khiến điểm H3 bị giới hạn ở 50.

Trace lần chạy 2 – sau khi sửa
step 1: model -> tool_calls
  tool: get_incident_details
  arguments:
    provider: cloudflare
    incident_id: totally-not-a-real-id

tool result:
  ok: false
  error: INCIDENT_NOT_FOUND

step 2: model -> finish
  Agent trả lời không thể xác nhận incident hoặc sự cố toàn cầu;
  không bịa link và yêu cầu thêm ID/thời gian để kiểm tra.

Trace trên được rút gọn để làm rõ luồng gọi tool; không phải bản dump JSON nguyên văn.

```

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu: Cập nhật SYSTEM_PROMPT trong agent.py, hướng dẫn rõ rằng khi người dùng cung cấp một incident ID cụ thể thì phải gọi get_incident_details để tra cứu, không dùng get_status thay thế.
Cập nhật mô tả tham số component trong SCHEMAS để làm rõ tham số này dùng để lọc component, không dùng để tra cứu incident ID.
- Vì sao chọn thay đổi này: Trace lần chạy đầu cho thấy lỗi nằm ở việc chọn sai tool. Điều chỉnh prompt và mô tả schema giúp model hiểu đúng mục đích của từng tool mà không cần thay đổi vòng lặp ReAct hoặc logic xử lý tool.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 95.91/100 | 97.91/100 |
| Điểm case đã chọn | 50/100 | 100/100 |
| Số model calls | 2 | 2 |
| Token, chi phí nếu có | Không có số liệu được ghi nhận trong kết quả hiện có | Không có số liệu được ghi nhận trong kết quả hiện có |

- Lỗi đã giải quyết: H3 đã gọi đúng get_incident_details, kiểm tra ID bằng kết quả từ tool, không xác nhận một sự cố toàn cầu khi chưa có bằng chứng và không tự tạo link. Điểm H3 tăng từ 50 lên 100.
- Lỗi còn lại hoặc mới xuất hiện: Một số case vẫn chưa đạt 100 điểm: E1 99, E4 97, E5 83, A1 97, A5 99, A6 92, H1 93, H5 95, H8 99.
E5 giảm từ 100 xuống 83 và A6 giảm từ 97 xuống 92. Cần xem trace và nhận xét judge của hai case này nếu muốn tiếp tục tối ưu; không nên kết luận nguyên nhân chỉ dựa trên điểm số.
Tổng điểm tăng 2.00 điểm, từ 95.91 lên 97.91. Cả hai lần chạy đều chấm đủ 22/22 case.

## 5. Giới hạn

Nêu giới hạn của điểm judge, của snapshot và của việc chỉ chạy một lần mỗi phiên bản.

- Giới hạn của điểm judge: Điểm được tạo bởi model judge nên có thể chịu ảnh hưởng từ cách diễn đạt, tiêu chí chấm và độ ổn định của model. Điểm số là tín hiệu đánh giá, không phải bằng chứng tuyệt đối rằng câu trả lời luôn đúng trong mọi tình huống.
- Giới hạn của snapshot: Công cụ trạng thái dựa trên dữ liệu snapshot tại một thời điểm. Kết quả phản ánh dữ liệu mà tool truy xuất được, không nhất thiết phản ánh trạng thái hiện tại hoặc toàn bộ lịch sử. Không tìm thấy incident trong 50 incident gần nhất cũng không chứng minh incident chưa từng tồn tại.
- Giới hạn của số lần chạy: Mỗi phiên bản mới được chạy một lần nên kết quả có thể biến động do phản hồi model, judge hoặc lỗi mạng/timeout. Hai lần chạy chưa đủ để khẳng định mọi thay đổi đều cải thiện ổn định chất lượng.
- Giới hạn về số liệu sử dụng: Bảng so sánh không ghi số token và chi phí vì các số liệu này chưa có trong kết quả đầu ra được cung cấp; không tự ước lượng hoặc điền số liệu giả.
