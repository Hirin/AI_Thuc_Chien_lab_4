# Phân tích System Prompt của TravelBuddy

Tài liệu này phân tích chi tiết cấu trúc và các quy tắc của `SYSTEM_PROMPT` hiện tại để hiểu cách Agent tư vấn và xử lý logic.

## 1. Cấu trúc Prompt
Prompt được chia thành các block XML rõ ràng để mô hình dễ dàng phân tách các loại chỉ dẫn:
- `<persona>`: Định nghĩa tính cách và tông giọng (thân thiện, người thực).
- `<rules>`: Các quy tắc vận hành và xử lý tool (08 quy tắc chính).
- `<tools_instruction>`: Hướng dẫn sử dụng các công cụ cụ thể (phối hợp tool).
- `<response_format>`: Quy định cấu trúc câu trả lời đầu ra (format trình bày).
- `<constraints>`: Các giới hạn tuyệt đối và bảo mật (không gọi 5+3).

## Phân tích & So sánh 2 phiên bản Prompt

| Tính năng | Bản Cơ bản (Basic) | Bản Thiết quân luật (Hardened) |
| :--- | :--- | :--- |
| **Vai trò** | Trợ lý hữu ích chung chung. | Chuyên gia du lịch nghiêm ngặt. |
| **Quy tắc Im lặng** | Không có (Dễ gây gãy luồng Tool). | Có (Silence Rule) - Ép gọi chuỗi Tool im lặng. |
| **Xử lý địa danh lạ** | Dễ báo lỗi "Không tìm thấy mã". | Tự động gọi `get_airport_code` để tra IATA. |
| **Bảo mật (Prompt Inj)** | Dễ bị lừa viết code/toán. | Chặn đứng 100% bằng quy tắc OOD. |
| **Memory** | Chỉ là hướng dẫn gợi ý. | Hướng dẫn rõ ràng về tên và ngữ cảnh. |

---

## Các thành phần "Sống còn" vừa bổ sung:

### 1. Quy tắc Im lặng Tuyệt đối (Strict Silence)
Đây là quy tắc quan trọng nhất giải quyết lỗi "Agent lặng im/ngáp ngắn ngáp dài" sau khi tra mã IATA. 
- **Bản Cơ bản**: Sau khi tìm được mã (VD: Pleiku -> PXU), Agent sẽ nói "Mình tìm được mã là PXU rồi, đợi chút mình tìm vé nhé". Câu nói này làm dừng vòng lặp UI, người dùng phải gõ tiếp thì bot mới chạy.
- **Bản Hardened**: Agent bị cấm nói. Nó phải âm thầm dùng kết quả PXU đó để gọi ngay `search_flights`.

### 2. Luồng tự trị qua Few-shot Examples
Bằng cách đưa ra ví dụ mẫu: `get_airport_code` -> `search_flights` -> `Kết quả`, chúng ta đã định hướng cho Agent (đặc biệt là gpt-4o) hiểu rằng đây là một **chuỗi liên hoàn**, không phải các bước rời rạc.

---

## 2. Các quy tắc cốt lõi (Rules)

### Quy tắc #2: Ưu tiên hành động (Proactive Tool Call)
- **Mục đích**: Khắc phục tình trạng Agent hỏi đi hỏi lại hoặc xác nhận dư thừa khi đã có đủ thông tin.
- **Cơ chế**: Yêu cầu gọi tool ngay lập tức nếu có (Thành phố, ngày tháng...).
- **Thông minh**: Tự động ánh xạ các từ viết tắt như "SG", "HN" sang mã IATA chuẩn ("SGN", "HAN") để tool không bị lỗi.

### Quy tắc #6: Xử lý ngoài phạm vi (Out-of-Scope)
- **Mục đích**: Giữ cho Agent tập trung vào du lịch và tiết kiệm token/chi phí.
- **Cơ chế**: Từ chối các câu hỏi về toán học, lập trình, chính trị... bằng một câu trả lời mẫu lịch sự.

### Quy tắc #7: Kiểm tra Logic trùng lặp (Sanity Check)
- **Mục đích**: Tránh các cuộc gọi API vô nghĩa.
- **Cơ chế**: So sánh điểm đi và điểm đến. Nếu trùng nhau, Agent sẽ giải thích lý do không hỗ trợ (chỉ có vé máy bay/khách sạn, không có phương tiện nội đô) thay vì gọi tool.

## 3. Bảo mật và Ràng buộc (Constraints)

### Jailbreak Protection
- Thêm các ràng buộc tuyệt đối để Agent không bị "dụ" bỏ qua system prompt (như kịch bản "5+3=" vừa rồi).
- Agent luôn phải quay lại vai trò trợ lý du lịch sau khi từ chối.

### No Booking
- Quy tắc nghiêm ngặt: Tuyệt đối không dùng từ "đặt" (book) để tránh gây hiểu lầm cho người dùng rằng Agent có thể thanh toán thực tế. Hệ thống chỉ hỗ trợ **Tìm kiếm** và **Tư vấn**.

## 4. Tối ưu hóa Tool Call
- Hướng dẫn Agent cách phối hợp các công cụ: Ví dụ, gọi `get_current_location` trước khi gọi `get_weather_forecast` nếu người dùng hỏi "chỗ tôi".
- Điều này giúp giảm số lượt hội thoại cần thiết để có được thông tin cuối cùng.

## 5. Kết luận
System Prompt hiện tại đã được "hardened" (siết chặt) để:
1. Quyết đoán hơn trong việc gọi tool (đã cải thiện sau khi bổ sung Quy tắc #2).
2. Xử lý lỗi logic ngay từ bước lập luận.
3. Duy trì persona an toàn và chuyên nghiệp, tránh các lỗi phổ biến của AI tổng quát.
