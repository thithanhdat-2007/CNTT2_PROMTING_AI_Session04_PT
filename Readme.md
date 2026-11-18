# BÁO CÁO PHÂN TÍCH & SO SÁNH TRADE-OFF GIẢI PHÁP PROMPTING NÂNG CAO

**Chủ đề:** Tối ưu hóa quy trình Chain-of-Thought, Prompt Chaining và Xử lý đứt gãy dữ liệu (Edge Cases) trong tự động hóa xử lý đơn hàng/khiếu nại.

---

## BƯỚC 1: ĐỀ XUẤT 2 PROMPT MẪU

Để giải quyết bài toán tiếp nhận tin nhắn khách hàng lộn xộn, đối chiếu tồn kho/chính sách và tính toán hoàn tiền/chốt đơn tự động, ta xây dựng 2 phương án tiếp cận khác nhau:

### Giải pháp 1: Phương án Đơn giản, Ngắn gọn (Zero-shot / Minimalist Prompting)
*Đặc điểm: Gộp chung các bước vào một chỉ thị ngắn gọn, dựa vào khả năng suy luận mặc định của LLM mà không thiết lập khung cấu trúc cứng hay bẫy lỗi chi tiết.*

```text
Bạn là nhân viên CSKH của cửa hàng công nghệ TechZone. Hãy đọc tin nhắn của khách hàng dưới đây, suy nghĩ từng bước để kiểm tra thông tin đơn hàng, đối chiếu với dữ liệu kho của shop và viết tin nhắn phản hồi cho khách.

Dữ liệu kho & Chính sách:
- Tai nghe Bluetooth X1: Giá 500.000đ (Còn hàng)
- Bàn phím cơ K2: Giá 1.200.000đ (Hết hàng)
- Phí ship: 30.000đ (Freeship cho đơn từ 1.000.000đ).
- Quy định: Đơn hàng hợp lệ cần có Tên, SĐT và Địa chỉ. Nếu thiếu thì hỏi lại khách.

Tin nhắn khách hàng:
"{{TIN_NHAN_KHACH_HANG}}"
```

### Giải pháp 2: Phương án Chi tiết, Nhiều ràng buộc (Structured CoT + Prompt Chaining + Guardrails)
*Đặc điểm: Tách bạch quy trình thành chuỗi xử lý logic (Chuỗi Bước 1: Trích xuất & Bắt lỗi $\rightarrow$ Chuỗi Bước 2: Đối chiếu & Tính toán CoT $\rightarrow$ Chuỗi Bước 3: Sinh phản hồi), sử dụng thẻ XML để cô lập suy luận và thiết lập các điều kiện chặn lỗi biên (Guardrails) nghiêm ngặt.*

```text
# ROLE
Bạn là Hệ thống Xử lý Đơn hàng Tự động (Order Logic Engine) của TechZone. Nhiệm vụ của bạn là thực thi chuỗi xử lý 3 giai đoạn một cách chính xác tuyệt đối, không được bỏ qua bất kỳ ràng buộc nào.

# KNOWLEDGE BASE (DỮ LIỆU NGUỒN)
- Sản phẩm 1: `Tai nghe Bluetooth X1` | Giá: `500000` | Tồn kho: `15`
- Sản phẩm 2: `Bàn phím cơ K2` | Giá: `1200000` | Tồn kho: `0`
- Chính sách vận chuyển: Phí ship = `30000`. Nếu `Tong_tien_hang >= 1000000` thì `Phi_ship = 0`.

# EXECUTION CHAIN & GUARDRAILS (CHUỖI THỰC THI & BẪY LỖI)
Thực hiện tuần tự 3 mắt xích sau:

## Mắt xích 1: Trích xuất & Kiểm tra tính toàn vẹn (Data Validation)
- Trích xuất: `Ten`, `SDT`, `Dia_chi`, `Danh_sach_mon_hang` (Tên SP, Số lượng).
- **Bẫy lỗi 1 (Thiếu dữ liệu):** Nếu thiếu Tên, Địa chỉ, hoặc `SDT` không đủ 10 chữ số (bắt đầu bằng 0) -> Đánh dấu trạng thái `DATA_BROKEN` (Đứt gãy dữ liệu). Tuyệt đối KHÔNG tự bịa thông tin.
- **Bẫy lỗi 2 (Số lượng vô lý):** Nếu số lượng $\le 0$ hoặc không phải số nguyên -> Đánh dấu `INVALID_QTY`.

## Mắt xích 2: Đối chiếu kho & Tính toán Logic (Chain-of-Thought)
- Kiểm tra từng món hàng với `KNOWLEDGE BASE`:
  - Nếu `Ton_kho == 0` hoặc `So_luong_dat > Ton_kho` -> Đánh dấu món đó là `OUT_OF_STOCK`.
  - Chỉ tính tiền cho các món `IN_STOCK`. Nếu toàn bộ đơn `OUT_OF_STOCK` hoặc trạng thái là `DATA_BROKEN`, **NGẮT CHUỖI TÍNH TIỀN** (không xuất hóa đơn chốt đơn).
- Công thức (nếu hợp lệ): `Tong_thanh_toan = Tong_tien_hang + Phi_ship`.

## Mắt xích 3: Xuất kết quả phản hồi (Output Generation)
- Luôn xuất quá trình suy luận của Mắt xích 1 & 2 vào thẻ `<logic_chain>` (định dạng JSON/Text rõ ràng).
- Xuất câu trả lời gửi khách hàng vào thẻ `<final_response>` với giọng văn chuyên nghiệp, lịch sự, giải quyết triệt để các lỗi phát hiện ở Mắt xích 1 & 2.

# INPUT
Tin nhắn khách hàng:
"""
{{TIN_NHAN_KHACH_HANG}}
"""
```

---

## BƯỚC 2: LẬP BẢNG SO SÁNH TRADE-OFF

Để đánh giá khách quan sự đánh đổi (Trade-off) giữa hai phương án khi đưa vào vận hành thực tế, ta xét trên 6 tiêu chí cốt lõi:

| Tiêu chí đánh giá | Giải pháp 1 (Đơn giản, ngắn gọn) | Giải pháp 2 (Chi tiết, nhiều ràng buộc) | Phân tích sự đánh đổi (Trade-off) |
| :--- | :--- | :--- | :--- |
| **Độ dễ hiểu & Triển khai** | **Cao** (Viết nhanh trong 1 phút, người không rành kỹ thuật cũng đọc hiểu được). | **Trung bình** (Cấu trúc dài, đòi hỏi tư duy thiết kế luồng và quy tắc rõ ràng). | Giải pháp 1 tiết kiệm thời gian khởi tạo ban đầu, trong khi Giải pháp 2 tốn công sức thiết kế ban đầu nhưng dễ bảo trì từng mắt xích về sau. |
| **Tốn Token (Chi phí & Tốc độ)** | **Ít** (~120 input tokens). Phản hồi nhanh, chi phí API thấp. | **Nhiều hơn** (~380 input tokens + output thẻ `<logic_chain>`). Chi phí cao gấp ~3 lần. | Đánh đổi chi phí token và độ trễ (latency) thêm vài trăm mili-giây để đổi lấy sự an toàn và chính xác tuyệt đối trong nghiệp vụ. |
| **Chặn lỗi biên (Edge Cases)** | **Trung bình** (Dễ bị AI bỏ sót lỗi SĐT sai số, hoặc vẫn tính tiền chốt đơn dù hàng đã hết). | **Cực tốt** (Bắt chết các trường hợp thiếu dữ liệu, sai SĐT, hết kho và chủ động ngắt chuỗi). | Giải pháp 1 dễ gặp ảo giác (Hallucination) khi gặp dữ liệu nhiễu; Giải pháp 2 triệt tiêu rủi ro sai sót tài chính/vận hành. |
| **Tính nhất quán đầu ra (Format)** | **Thấp** (Lúc thì AI viết đoạn văn, lúc thì gạch đầu dòng, khó tích hợp vào phần mềm/CRM). | **Rất cao** (Tách biệt rõ `<logic_chain>` để hệ thống đọc và `<final_response>` để gửi khách). | Giải pháp 1 chỉ phù hợp để người dùng đọc thủ công; Giải pháp 2 sẵn sàng để tích hợp tự động hóa qua API/Webhook. |
| **Khả năng gỡ lỗi (Debug)** | **Khó** (Khi AI tính sai tiền, không biết nó sai ở khâu đọc hiểu hay khâu cộng trừ). | **Dễ dàng** (Nhìn vào thẻ `<logic_chain>` biết ngay lỗi đứt gãy ở Mắt xích 1 hay Mắt xích 2). | Giải pháp 2 cung cấp tính minh bạch (Explainability) cao trong luồng suy luận của AI. |

---

## BƯỚC 3: LỰA CHỌN PHƯƠNG ÁN TỐI ƯU & CHẠY THỬ NGHIỆM

### 3.1. Lựa chọn phương án tối ưu nhất
* **Lựa chọn:** **Giải pháp 2 (Phương án Chi tiết, Nhiều ràng buộc)**.
* **Lý do lựa chọn:**
  1. **Đặc thù bài toán vận hành thương mại:** Trong quy trình xử lý đơn hàng và tính tiền, sai sót về tồn kho (bán hàng đã hết) hoặc đứt gãy thông tin giao nhận (sai SĐT, thiếu địa chỉ) gây thiệt hại trực tiếp đến uy tín và chi phí vận hành của cửa hàng. Việc tốn thêm ~250 token (chi phí không đáng kể với các mô hình hiện nay) là hoàn toàn xứng đáng để đổi lấy khả năng **chặn lỗi biên cực tốt**.
  2. **Kiểm soát được hiện tượng "Ảo giác" (Hallucination):** Khi gặp dữ liệu mập mờ, LLM ở Giải pháp 1 có xu hướng tự suy diễn để làm hài lòng người dùng. Giải pháp 2 sử dụng cơ chế *Ngắt chuỗi (Circuit Breaker)* khi gặp trạng thái `DATA_BROKEN` hoặc `OUT_OF_STOCK`, ép AI dừng lại để hỏi khách thay vì chốt đơn ảo.
  3. **Khả năng mở rộng (Scalability):** Việc tách thẻ `<logic_chain>` và `<final_response>` giúp lập trình viên dễ dàng dùng Regex/Parser tách phần trả lời gửi thẳng cho khách hàng qua Zalo/Messenger, trong khi phần logic được lưu lại làm log kiểm tra.

### 3.2. Kết quả chạy thử nghiệm thực tế (với Giải pháp 2)

Để kiểm chứng sức mạnh chặn lỗi biên của Giải pháp 2, ta đưa vào một **Input chứa nhiều bẫy dữ liệu phức tạp** (khách đặt 2 món trong đó 1 món hết hàng, SĐT viết thiếu số, và hỏi dồn dập):

#### Input chạy thử:
> *"Shop ơi lên đơn gấp cho mình 1 cái tai nghe bluetooth X1 với 1 bàn phím cơ K2 nhé. Mình tên Tuấn, sđt 090512345, ship về 88 Cầu Giấy, Hà Nội. Tính tổng tiền luôn giúp mình nha!"*

*(Phân tích bẫy trong Input: Bàn phím K2 đã hết hàng; SĐT `090512345` chỉ có 9 số; Khách giục "tính tổng tiền luôn" để bẫy AI chốt đơn khi chưa đủ điều kiện).*

#### Output thực tế từ AI (Áp dụng Giải pháp 2):

```xml
<logic_chain>
{
  "mat_xich_1_validation": {
    "Ten": "Tuấn",
    "SDT": "090512345",
    "Dia_chi": "88 Cầu Giấy, Hà Nội",
    "Danh_sach_mon_hang": [
      {"ten_sp": "Tai nghe Bluetooth X1", "so_luong": 1},
      {"ten_sp": "Bàn phím cơ K2", "so_luong": 1}
    ],
    "trang_thai_du_lieu": "DATA_BROKEN",
    "chi_tiet_loi": [
      "Bẫy lỗi 1 kích hoạt: Số điện thoại '090512345' chỉ có 9 chữ số (thiếu 1 số so với chuẩn 10 số)."
    ]
  },
  "mat_xich_2_inventory_and_math": {
    "kiem_tra_kho": [
      {"ten_sp": "Tai nghe Bluetooth X1", "trang_thai": "IN_STOCK", "ton_kho": 15, "don_gia": 500000},
      {"ten_sp": "Bàn phím cơ K2", "trang_thai": "OUT_OF_STOCK", "ton_kho": 0, "don_gia": 1200000}
    ],
    "hanh_dong": "NGẮT CHUỖI TÍNH TIỀN CHỐT ĐƠN. Lý do: Đơn hàng đang ở trạng thái DATA_BROKEN và có sản phẩm OUT_OF_STOCK. Chỉ báo giá tạm tính cho sản phẩm còn hàng để khách tham khảo và yêu cầu bổ sung thông tin."
  }
}
</logic_chain>

<final_response>
Chào anh Tuấn, cảm ơn anh đã liên hệ đặt hàng tại TechZone!

Em đã kiểm tra thông tin đơn hàng của anh, tuy nhiên hiện tại hệ thống chưa thể lên đơn ngay do có 2 vấn đề sau cần anh xác nhận lại giúp em ạ:

1. **Về tình trạng kho hàng:**
   - **Tai nghe Bluetooth X1 (SL: 1):** Hiện đang **còn hàng** (Giá: 500.000đ).
   - **Bàn phím cơ K2 (SL: 1):** Hiện tại mẫu này bên em vừa **tạm hết hàng** nên chưa thể giao kèm đơn này cho anh được ạ.

2. **Về thông tin liên hệ:**
   - Số điện thoại `090512345` anh cung cấp đang bị **thiếu 1 chữ số** (mới có 9 số). Anh kiểm tra và nhắn lại giúp em SĐT chính xác (đủ 10 số) để bên vận chuyển liên hệ giao hàng nhé.

*Lưu ý về chi phí:* Nếu anh đồng ý đặt trước **1 Tai nghe Bluetooth X1 (500.000đ)**, phí vận chuyển sẽ là **30.000đ** (do đơn chưa đạt mức 1.000.000đ để được Freeship), tổng thanh toán sẽ là **530.000đ**.

Anh phản hồi lại SĐT chính xác và xác nhận giúp em xem mình có muốn đi đơn trước chiếc tai nghe không nhé!
</final_response>
```

### 3.3. Đánh giá kết quả chạy thử
* **Thành công:** AI không bị mắc bẫy câu lệnh *"Tính tổng tiền luôn giúp mình"* của khách hàng. Nhờ cơ chế **Prompt Chaining** và **Guardrails**, AI đã bắt chính xác lỗi SĐT 9 số, phát hiện Bàn phím K2 hết hàng, tự động ngắt luồng chốt đơn ảo, đồng thời đưa ra phương án báo giá dự phòng (500k + 30k ship = 530k) cực kỳ logic và chuyên nghiệp.