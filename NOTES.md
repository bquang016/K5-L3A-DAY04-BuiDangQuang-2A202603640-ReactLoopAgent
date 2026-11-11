# Ghi chú phân tích · Day 04

Họ và tên: Bùi Đăng Quang
MSSV: 2A202603640

## 1. Case được chọn

- ID case: E2
- Câu trả lời của agent:
> Theo dữ liệu từ Cloudflare, có một sự cố liên quan đến Cache Purge gần đây:
> - Tên sự cố: Fedramp Customers have Delayed Purges
> - ID: wp49bp8bqo6c
> - Trạng thái: Đã xác định (identified)
> - Cập nhật lần cuối: 6 tháng 11, 2026
> - Nội dung: Cloudflare đã xác định một lỗi trong quy trình purge ảnh hưởng đến khách hàng Fedramp. Các khách hàng không thuộc Fedramp không bị ảnh hưởng.
> 
> Đối với website CLB không thuộc FedRAMP, theo thông tin trên, họ không bị ảnh hưởng bởi sự cố này. Tuy nhiên, vẫn nên theo dõi thêm thông tin từ [Cloudflare Status](https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c) để có cập nhật mới nhất.
- Điểm ở lần chạy 1: 75 / 100

## 2. Bằng chứng trong trace

Dán đoạn trace chứng minh lỗi và nêu nguyên nhân có thể xảy ra.

```json
[
  {
    "event": "tool",
    "step": 1,
    "name": "get_status",
    "arguments": {
      "component": "cache purge",
      "provider": "cloudflare"
    },
    "result": {
      "provider": "cloudflare",
      "overall": {
        "indicator": "major",
        "description": "Partial System Outage"
      },
      "components": [
        {
          "name": "CDN Cache Purge",
          "status": "degraded_performance"
        }
      ],
      "incidents": [
        {
          "id": "wp49bp8bqo6c",
          "name": "Fedramp Customers have Delayed Purges",
          "status": "identified"
        }
      ]
    }
  },
  {
    "event": "tool",
    "step": 2,
    "name": "get_incident_details",
    "arguments": {
      "incident_id": "wp49bp8bqo6c",
      "provider": "cloudflare"
    },
    "result": {
      "ok": true,
      "id": "wp49bp8bqo6c",
      "name": "Fedramp Customers have Delayed Purges",
      "status": "identified",
      "updates": [
        {
          "status": "investigating",
          "body": "Cloudflare has identified a bug with our purge pipeline impacting fedramp customers. Stale content may be served until expired. Non-fedramp customers are not impacted."
        }
      ],
      "url": "https://www.cloudflarestatus.com/incidents/wp49bp8bqo6c"
    }
  }
]
```

**Nguyên nhân:** Ở lần chạy 1, agent đã tra cứu đúng 2 bước tools (`get_status` rồi `get_incident_details`) và trích dẫn URL nguồn. Tuy nhiên, nhận xét của Judge ở lần 1 (`score: 75`) chỉ ra rằng agent chưa nhấn mạnh rõ ranh giới: mặc dù khách hàng non-FedRAMP không bị ảnh hưởng bởi incident cụ thể này, nhưng không thể khẳng định chắc chắn 100% website CLB có gặp lỗi cache riêng do cấu hình khác hay không. Sang lần 2, câu trả lời phân tách ranh giới rõ ràng hơn giữa sự cố hạ tầng và lỗi riêng của website, đạt điểm tuyệt đối 100/100.

## 3. Thay đổi đã thực hiện

- Sửa gì, ở đâu:
  1. Trong `agent.py`: 
     - Xây dựng Agent Loop ReAct hoàn chỉnh, lưu trữ tin nhắn assistant nguyên vẹn trước khi đẩy tool results, parse an toàn JSON arguments (`INVALID_ARGUMENTS`).
     - Tích hợp **Validation Gate** tự động phát hiện thiếu URL nguồn (`validation_missing_source`) để yêu cầu model bổ sung trong ngân sách bước.
     - Tích hợp **Cơ chế theo vết liên kết đa bước (Multi-step linked incident tracking)**: Tự động trích xuất các incident ID liên kết trong phần updates (như `xr3myzj7ms51` trong case H1, H6) để nhắc model gọi tiếp `get_incident_details` ở bước sau trước khi kết luận.
     - Bổ sung quy tắc ranh giới lịch sử (giới hạn 50 incidents, không bịa đặt năm 2025) giúp case A5 đạt 100 điểm tuyệt đối; và quy tắc từ chối dịch vụ không hỗ trợ (Linear, Slack) giúp case B9 tăng vọt từ 10 lên 80 điểm.
  2. Trong `providers.py`: Tích hợp cơ chế retry tự động với backoff thời gian khi gặp lỗi `HTTP 429` (Rate Limit) giúp bộ eval chạy ổn định xuyên suốt 22 cases mà không bị dừng đột ngột.
- Vì sao chọn thay đổi này: Đảm bảo agent hoạt động trung thực dựa trên bằng chứng (evidence-based), không hallucinate, luôn đính kèm URL xác thực từ nhà cung cấp và có khả năng điều tra đa bước sâu sắc.

## 4. So sánh hai lần chạy

| Chỉ số | `eval_before.json` | `eval_after.json` |
|---|---|---|
| Điểm tổng | 80.91 / 100 | 81.91 / 100 |
| Điểm case đã chọn (E2) | 75 / 100 | 100 / 100 |
| Điểm Case A5 | 40 / 100 | 100 / 100 |
| Điểm Case H1 | 60 / 100 | 75 / 100 |
| Số model calls | 53 | 52 |
| Token, chi phí | 87,873 tokens (~$0.0115) | 117,896 tokens (~$0.0152) |

- Lỗi đã giải quyết: Toàn bộ 22/22 case ở cả 2 lần chạy đều hoàn thành trọn vẹn (`scored_cases: 22`, không có case nào bị `unscored` hay `null`). Khắc phục triệt để lỗi vi phạm `min_tool_steps` ở các case điều tra liên kết, nâng điểm case A5 lên 100 và case E2 lên 100.
- Lỗi còn lại hoặc mới xuất hiện: Một số case đặc thù yêu cầu dữ liệu nhiễu dài (H6) đạt mức 60 điểm do hạn chế về ngữ cảnh tóm tắt dài của model.

## 5. Giới hạn

1. **Giới hạn của điểm judge:** LLM Judge sử dụng prompting để chấm xấp xỉ theo 4 tiêu chí (factuality, completeness, evidence, boundaries). Điểm số có độ biến thiên nhẹ giữa các lần chạy với cùng một câu trả lời do tính ngẫu nhiên (temperature) của judge model.
2. **Giới hạn của snapshot:** Dữ liệu thử nghiệm được lưu cố định tại thời điểm 09/11/2026 và chỉ bao gồm 50 incidents gần nhất cho mỗi nhà cung cấp; do đó không phản ánh trạng thái live thời gian thực và không thể đại diện cho toàn bộ lịch sử dài hạn.
3. **Giới hạn của việc chạy một lần mỗi phiên bản:** Một lượt chạy đơn lẻ chỉ phản ánh hiệu năng tại một thời điểm cụ thể; trong thực tế cần chạy nhiều lượt lặp để đánh giá phương sai và độ ổn định thống kê của hệ thống agent.

## 6. Thử thách Bonus (Tùy chọn)

- **Kết quả file `eval_bonus.json`:** Hoàn thành **10/10 cases (B1–B10)**, 0 case lỗi.
- **Điểm bộ Bonus:** **70.00 / 100**
- **Điểm Bonus quy đổi:** **+10.50 / 15 điểm** (theo công thức: `score_0_100 × 0.15`).
- **Các cải tiến kỹ thuật nổi bật:**
  - Case B9 (từ chối bịa đặt Linear/Slack): Tăng vọt từ 10 lên **80 điểm**.
  - Case B6 (lọc 160 mục nhiễu): Tăng từ 50 lên **85 điểm**.
  - Case B4 (dừng retry 429 sau 3 lần): Tăng từ 50 lên **80 điểm**.
  - Case B7 (chống Prompt Injection): Đạt **90 điểm**.
  - Phân trang timeline Vercel qua `offset` và `limit` được xử lý chuẩn xác.
