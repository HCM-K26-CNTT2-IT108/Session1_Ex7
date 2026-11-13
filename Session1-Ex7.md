# Bài 7: Thiết kế Quy trình Che mờ (Data Masking) Khách VIP

## Phần giải thích kiến thức

- **PII**: thông tin nhận dạng cá nhân, cho biết "đây là ai" (họ tên, SĐT, email, địa chỉ...).
- **Data Masking (che mờ dữ liệu)**: thay PII thật bằng **nhãn thay thế (Placeholder)** trước khi đưa vào AI công cộng.
- **De-masking (khôi phục dữ liệu)**: sau khi AI trả kết quả, **thay nhãn về lại thông tin thật** trước khi gửi cho khách.

**Quy trình tổng quát:**

```
Email gốc (có PII)
      │  1. Lập bảng che mờ (giữ nội bộ)
      ▼
Masked Input (chỉ còn nhãn)
      │  2. Gửi AI công cộng
      ▼
AI soạn thư (vẫn chứa nhãn)
      │  3. De-masking: thay nhãn bằng thông tin thật (người thực hiện)
      ▼
Thư hoàn chỉnh → nhân viên CSKH duyệt → gửi khách
```

**Nguyên tắc quan trọng:**
1. **Bảng đối chiếu (Masking Table) chỉ lưu nội bộ**, tuyệt đối không gửi cho AI.
2. **Mỗi loại PII một nhãn riêng, đặt tên có nghĩa**, để AI hiểu vai trò của từng nhãn và không mất ngữ cảnh.
3. Giữ lại thông tin **không định danh** (sản phẩm, giá, thời gian giao trễ) để AI hiểu đúng vấn đề.

---

## Phần A: Bảng che mờ (Masking Table)

| Thông tin PII thật | Nhãn che mờ (Placeholder) | Loại dữ liệu |
|---|---|---|
| Phạm Minh C | `[TÊN_KHÁCH_VIP]` | Họ và tên |
| 0988.111.222 | `[SĐT_KHÁCH]` | Số điện thoại |
| minhc@gmail.com | `[EMAIL_KHÁCH]` | Địa chỉ email |
| Căn hộ 502 Chung cư Sunrise, Q7 | `[ĐỊA_CHỈ_KHÁCH]` | Địa chỉ nhà |

**Thông tin được giữ lại (không phải PII, cần cho ngữ cảnh):**

| Thông tin | Lý do giữ lại |
|---|---|
| Ngày 20/09 | Mốc thời gian sự cố |
| 2kg thịt bò Wagyu, 4.500.000 VNĐ | Sản phẩm và giá trị đơn hàng để AI đánh giá mức độ nghiêm trọng |
| MartX District 7 | Tên chi nhánh của công ty, không phải thông tin cá nhân khách |
| Giao chậm 3 tiếng, hỏng tiệc gia đình | Nội dung khiếu nại, cần để AI soạn thư đúng trọng tâm |

> Ghi chú: nếu công ty yêu cầu chặt hơn, có thể che thêm mã đơn hàng bằng `[MÃ_ĐƠN]`. Trong đề bài không có mã đơn nên không cần.

---

## Phần B: Văn bản khiếu nại đã che mờ (Masked Input)

> "Tôi là `[TÊN_KHÁCH_VIP]` (SĐT: `[SĐT_KHÁCH]`, Email: `[EMAIL_KHÁCH]`, ở `[ĐỊA_CHỈ_KHÁCH]`). Ngày 20/09 tôi mua 2kg thịt bò Wagyu giá 4.500.000 VNĐ tại MartX District 7 nhưng bị giao chậm 3 tiếng làm hỏng tiệc gia đình. Tôi yêu cầu bồi thường 100%!"

**Kiểm tra lại (checklist 100% PII đã được che):**

- [x] Họ tên đã thay bằng `[TÊN_KHÁCH_VIP]`
- [x] Số điện thoại đã thay bằng `[SĐT_KHÁCH]`
- [x] Email đã thay bằng `[EMAIL_KHÁCH]`
- [x] Địa chỉ nhà (căn hộ, chung cư, quận) đã thay bằng `[ĐỊA_CHỈ_KHÁCH]`
- [x] Không còn tên riêng, số, địa chỉ nào của khách trong văn bản

---

## Phần C: Prompt gửi AI

```
Bạn là chuyên viên chăm sóc khách hàng cao cấp của siêu thị MartX.

NGỮ CẢNH:
Dưới đây là email khiếu nại của một khách VIP. Thông tin cá nhân đã được
thay bằng các nhãn trong ngoặc vuông, ví dụ [TÊN_KHÁCH_VIP], [SĐT_KHÁCH],
[EMAIL_KHÁCH], [ĐỊA_CHỈ_KHÁCH].

EMAIL KHIẾU NẠI:
"Tôi là [TÊN_KHÁCH_VIP] (SĐT: [SĐT_KHÁCH], Email: [EMAIL_KHÁCH], ở
[ĐỊA_CHỈ_KHÁCH]). Ngày 20/09 tôi mua 2kg thịt bò Wagyu giá 4.500.000 VNĐ
tại MartX District 7 nhưng bị giao chậm 3 tiếng làm hỏng tiệc gia đình.
Tôi yêu cầu bồi thường 100%!"

NHIỆM VỤ:
Soạn một thư xin lỗi gửi khách hàng này.

YÊU CẦU:
1. Giữ nguyên các nhãn trong ngoặc vuông, KHÔNG tự bịa tên, số điện thoại,
   email hay địa chỉ thật. Khi cần gọi tên khách, dùng đúng nhãn
   [TÊN_KHÁCH_VIP] và xưng hô lịch sự "Quý khách".
2. Giọng văn chân thành, chuyên nghiệp, đồng cảm với việc khách bị hỏng
   tiệc gia đình.
3. Nêu rõ sự việc: đơn hàng 2kg thịt bò Wagyu ngày 20/09 tại MartX
   District 7 bị giao chậm 3 tiếng.
4. KHÔNG cam kết mức bồi thường cụ thể (như 100%). Chỉ nói rằng yêu cầu
   của khách sẽ được bộ phận có thẩm quyền xem xét và phản hồi sớm.
5. Hẹn liên hệ lại với khách qua [SĐT_KHÁCH] hoặc [EMAIL_KHÁCH] trong
   vòng 24 giờ làm việc.
6. Độ dài khoảng 150-200 chữ, có lời chào đầu và lời kết, ký tên
   "Bộ phận Chăm sóc khách hàng MartX".
```

**Giải thích các lựa chọn trong prompt:**

| Phần trong prompt | Mục đích |
|---|---|
| Giải thích ý nghĩa các nhãn | Để AI hiểu nhãn là gì, không mất ngữ cảnh xưng hô |
| "Giữ nguyên nhãn, không bịa thông tin" | Tránh AI tự điền tên giả, giúp de-mask chính xác |
| "Không cam kết bồi thường cụ thể" | Quyết định bồi thường thuộc thẩm quyền con người (Human-in-the-loop) |
| Hẹn liên hệ trong 24 giờ | Tạo cam kết hành động rõ ràng, đúng chuẩn CSKH |

---

## Phần D: Văn bản De-mask

### D1. Thư AI trả về (còn nhãn)

```
Kính gửi Quý khách [TÊN_KHÁCH_VIP],

Thay mặt siêu thị MartX, chúng tôi xin gửi lời xin lỗi chân thành nhất đến
Quý khách về sự cố xảy ra với đơn hàng 2kg thịt bò Wagyu ngày 20/09 tại
MartX District 7.

Chúng tôi hiểu rằng việc giao hàng chậm 3 tiếng đã làm ảnh hưởng đến bữa
tiệc gia đình của Quý khách, và đó là một trải nghiệm không xứng đáng với
sự tin tưởng mà Quý khách dành cho MartX. Đây là thiếu sót của chúng tôi
và chúng tôi hoàn toàn nhận trách nhiệm.

Yêu cầu của Quý khách đã được ghi nhận và chuyển đến bộ phận có thẩm quyền
để xem xét nghiêm túc. Chúng tôi sẽ liên hệ lại với Quý khách qua số điện
thoại [SĐT_KHÁCH] hoặc email [EMAIL_KHÁCH] trong vòng 24 giờ làm việc để
thông báo hướng giải quyết.

Một lần nữa, MartX xin chân thành xin lỗi và mong Quý khách tiếp tục cho
chúng tôi cơ hội phục vụ.

Trân trọng,
Bộ phận Chăm sóc khách hàng MartX
```

### D2. Bước De-masking (đối chiếu bảng ở Phần A)

| Nhãn trong thư | Thay bằng thông tin thật |
|---|---|
| `[TÊN_KHÁCH_VIP]` | Phạm Minh C |
| `[SĐT_KHÁCH]` | 0988.111.222 |
| `[EMAIL_KHÁCH]` | minhc@gmail.com |
| `[ĐỊA_CHỈ_KHÁCH]` | Căn hộ 502 Chung cư Sunrise, Q7 (thư này không dùng nhãn này) |

### D3. Thư xin lỗi cuối cùng (sau khi De-mask)

```
Kính gửi Quý khách Phạm Minh C,

Thay mặt siêu thị MartX, chúng tôi xin gửi lời xin lỗi chân thành nhất đến
Quý khách về sự cố xảy ra với đơn hàng 2kg thịt bò Wagyu ngày 20/09 tại
MartX District 7.

Chúng tôi hiểu rằng việc giao hàng chậm 3 tiếng đã làm ảnh hưởng đến bữa
tiệc gia đình của Quý khách, và đó là một trải nghiệm không xứng đáng với
sự tin tưởng mà Quý khách dành cho MartX. Đây là thiếu sót của chúng tôi
và chúng tôi hoàn toàn nhận trách nhiệm.

Yêu cầu của Quý khách đã được ghi nhận và chuyển đến bộ phận có thẩm quyền
để xem xét nghiêm túc. Chúng tôi sẽ liên hệ lại với Quý khách qua số điện
thoại 0988.111.222 hoặc email minhc@gmail.com trong vòng 24 giờ làm việc để
thông báo hướng giải quyết.

Một lần nữa, MartX xin chân thành xin lỗi và mong Quý khách tiếp tục cho
chúng tôi cơ hội phục vụ.

Trân trọng,
Bộ phận Chăm sóc khách hàng MartX
```

> **Lưu ý cuối:** nhân viên CSKH phải **đọc lại thư lần cuối** trước khi gửi (kiểm tra không còn nhãn sót, thông tin đúng, nội dung không cam kết vượt thẩm quyền). Đây là bước Human-in-the-loop. Việc có bồi thường 100% hay không do người có thẩm quyền quyết định, không phải AI.

---

## Tóm tắt nộp bài

- **Phần A:** Bảng che mờ 4 PII: họ tên → `[TÊN_KHÁCH_VIP]`, SĐT → `[SĐT_KHÁCH]`, email → `[EMAIL_KHÁCH]`, địa chỉ → `[ĐỊA_CHỈ_KHÁCH]`.
- **Phần B:** Văn bản khiếu nại đã thay 100% PII bằng nhãn, giữ nguyên thông tin sự việc (sản phẩm, giá, thời gian).
- **Phần C:** Prompt giải thích nhãn cho AI, yêu cầu giữ nguyên nhãn, không bịa thông tin và không cam kết bồi thường.
- **Phần D:** Thư AI soạn (còn nhãn) → thay lại thông tin thật theo bảng → thư hoàn chỉnh, được nhân viên duyệt trước khi gửi.
