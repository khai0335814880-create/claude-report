# Hướng phát triển: Hệ thống đọc tài liệu giấy thành dữ liệu bằng Claude

Tài liệu này gồm ba phần: (A) kiểm chứng xem đề xuất “Agentic Document Intelligence Pipeline” đúng đến đâu, (B) bản thiết kế đã chỉnh cho khả thi và thực tế, (C) cách đưa vào trang web hiện tại.

Ngày kiểm chứng: 21/11/2026. Giá, giới hạn và điểm benchmark thay đổi nhanh, hãy mở lại nguồn ở cuối trước khi nộp.

---

## 0. Kết luận ngắn

**Hướng đi đúng và khả thi.** Đọc hóa đơn, phiếu, bảng kê scan thành dữ liệu có cấu trúc là bài toán có thật, đã có người làm bằng Claude, và có đủ tính năng kỹ thuật để làm (đọc PDF trực tiếp, trả JSON đúng schema, giảm giá khi xử lý hàng loạt).

**Nhưng tài liệu gốc có 8 chỗ cần sửa trước khi đem đi trình bày**, vì nếu người chấm kiểm tra sẽ thấy sai hoặc nói quá:

| # | Chỗ cần sửa | Mức độ |
|---|---|---|
| 1 | “Chạy toàn bộ trong VPC nội bộ, không gửi dữ liệu ra ngoài” không đúng với cách Claude hoạt động | Nghiêm trọng |
| 2 | Chọn Sonnet 5.5 vì “hiểu ngữ cảnh” nhưng OCR thuần của nó thấp hơn Opus và Fable, cần định tuyến theo độ khó | Quan trọng |
| 3 | Số “95–99%” là con số tự công bố của nhà cung cấp, không phải kiểm chứng độc lập | Quan trọng |
| 4 | Case study Boldare đúng nhưng là tự báo cáo, không nêu cỡ mẫu; và đó là đơn đặt hàng (PO), không phải hóa đơn | Vừa |
| 5 | `claude-ocr-pipeline` không tìm thấy dưới tên này; nên thay bằng dự án có thật | Vừa |
| 6 | Không cần MCP Server render PDF thành ảnh; Claude nhận PDF trực tiếp | Vừa |
| 7 | “Hệ thống đa tác nhân” là thiết kế thừa; một quy trình cố định với vài điểm gọi AI đơn giản hơn, rẻ hơn, dễ kiểm thử hơn | Vừa |
| 8 | KPI “< 0,5% lỗi” và “70–80% tự động” chưa có cơ sở cho tiếng Việt và dữ liệu của khách; phải đo trên bộ mẫu riêng | Vừa |

---

## A. Kiểm chứng từng nhận định

Ký hiệu: ✅ đúng · ⚠️ đúng nhưng cần chú thích · ❓ không xác minh được · ❌ sai hoặc nên bỏ.

### A1. Về tính năng của Claude

| Nhận định trong tài liệu gốc | Kết quả | Chi tiết và nguồn |
|---|---|---|
| Claude xử lý PDF trực tiếp, giữ được bố cục | ✅ | Tài liệu Anthropic: mọi mô hình đang hoạt động đều hỗ trợ PDF; hệ thống chuyển mỗi trang thành ảnh và trích văn bản của trang đó, đưa cả hai cho Claude. Boldare cũng gửi PDF thẳng, không chuyển ảnh. [Docs PDF](https://platform.claude.com/docs/en/build-with-claude/pdf-support) |
| Cần `pdf-mcp` để render PDF thành PNG | ❌ | Không cần với PDF thông thường. Chỉ cần tiền xử lý khi file là ảnh scan nghiêng, mờ, hoặc định dạng không hỗ trợ (Word, Excel phải đổi sang PDF hoặc văn bản trước). |
| Structured Output đảm bảo dữ liệu đúng schema | ⚠️ | Đúng về **hình dạng**: dùng `output_config.format` với `type: "json_schema"`, kết quả là JSON hợp lệ khớp schema, không cần thử lại vì sai định dạng. Nhưng **giá trị bên trong vẫn có thể sai** (đọc nhầm 7 thành 1). Vì vậy giai đoạn kiểm tra vẫn bắt buộc. [Docs Structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs) |
| Giới hạn PDF | ⚠️ | Bản docs hiện tại ghi: tối đa 600 trang mỗi yêu cầu (100 trang nếu cửa sổ ngữ cảnh của yêu cầu dưới 1 triệu token), kích thước yêu cầu 128 MB trên API trực tiếp, không mật khẩu. Mỗi trang tốn khoảng 1.500–3.000 token văn bản, chưa kể phần ảnh. Trên Amazon Bedrock Converse API phải bật “citations” mới có khả năng hiểu hình ảnh đầy đủ. Các trang tài liệu cũ vẫn ghi 32 MB, nên hãy kiểm tra lại khi triển khai. |
| “Claude Vision mức Medium / High” | ❓ | Không tìm thấy tham số tên “vision medium/high”. Roboflow đo theo mức nỗ lực (effort) thấp và cao. Cách nói an toàn: “chỉnh mức nỗ lực suy luận theo độ khó của tài liệu”. Cần xác nhận tên tham số trong docs lúc cài đặt. |
| Dùng Haiku/mô hình nhẹ để phân loại | ✅ | Haiku 5.5 (ra mắt 7/10/2026) được Anthropic định vị cho phân loại, định tuyến, trích xuất và làm tác tử phụ. [Tổng hợp mô hình](https://www.gradually.ai/en/ai-models/anthropic/) |

### A2. Về benchmark và bằng chứng

| Nhận định | Kết quả | Chi tiết và nguồn |
|---|---|---|
| Sonnet 5.5 OCR 66,3% và Structured JSON 83,8% trên Roboflow Vision Evals | ✅ về số, ⚠️ về cách dùng | Đúng các con số ở mức effort thấp (ảnh chụp 10–11/2026). Nhưng cùng bảng đó cho thấy Fable 5.2 đạt OCR 79,5% và Structured JSON 92,8%; Opus 5.5 đạt OCR 78,3% và Structured JSON 91,5%. Claude cũng không dẫn đầu OCR thuần (một mô hình của Google đạt 87,2%). Đây là bài đo thị giác chung, **không phải hóa đơn tiếng Việt**. [So sánh Fable 5.2 vs Sonnet 5.5](https://playground.roboflow.com/models/compare/claude-fable-5-2-vs-claude-sonnet-5-5), [Opus 5.5](https://playground.roboflow.com/models/compare/claude-opus-5-5-vs-qwen2-5-vl-7b-instruct), [Gemini 4 Carbon](https://playground.roboflow.com/models/compare/claude-sonnet-5-5-vs-gemini-4-carbon) |
| Boldare: 15 phút xuống dưới 3 phút, lỗi 5% xuống dưới 0,5% | ⚠️ | Có thật. Bài ghi 15 phút → 2–3 phút, lỗi ~5% → <0,5%. Chuỗi xử lý: Google Drive → Claude Sonnet 4 Vision → Databricks → ứng dụng Streamlit để nhân viên duyệt. Hạn chế: tự báo cáo, bài viết quảng bá dịch vụ, không nêu số lượng tài liệu hay cách đo lỗi. Và là **Purchase Order song ngữ Ả Rập–Anh**, không phải hóa đơn. [Boldare](https://www.boldare.com/blog/automated-purchase-order-processing-claude-vision-databricks) |
| `claude-ocr-pipeline` mã nguồn mở | ❓ | Không tìm thấy kho mã tên này. Có các dự án tương tự thật: **Noteker** (MCP server đổi PDF ghi chú viết tay thành Markdown bằng Claude Vision) và hướng dẫn dùng Claude Code đọc ảnh vở viết tay. Chi tiết “tự đẩy action item vào app nhắc việc” không xác minh được. Nên dẫn Noteker thay thế. [Noteker](https://images.glama.ai/mcp/servers/andras-tkcs/noteker), [Hướng dẫn Termdock](https://www.termdock.com/blog/ai-cli-digitize-handwritten-notes) |
| Extend.ai đạt 95–99% | ⚠️ | Đúng là Extend công bố 95–99%. Trang tổng hợp ngành ghi rõ đây là số tự báo cáo, chưa có kiểm chứng độc lập trên bộ dữ liệu chuẩn. Dùng như “nhà cung cấp tuyên bố”, không dùng như sự thật. [IDP Software](https://idp-software.com/vendors/extend-ai/) |
| “Agentic extraction chính xác 95–99%” như một phương pháp | ❌ cách diễn đạt | Con số thuộc về sản phẩm cụ thể, không phải của cả phương pháp. Bài đo ExtractBench (LlamaIndex và Kaggle, 370 tài liệu) có mức cao nhất 95,6% cho chính sản phẩm của họ, chi phí 8,1 xu mỗi trang. Chính LlamaIndex vừa tổ chức bài đo vừa có sản phẩm đứng đầu, nên cũng cần đọc thận trọng. [ExtractBench](https://llamaindex.ai/blog/introducing-extractbench) |

### A3. Về bảo mật và dữ liệu

| Nhận định | Kết quả | Chi tiết |
|---|---|---|
| “Chạy trong VPC nội bộ, dùng MCP nội bộ, không gửi dữ liệu ra ngoài” | ❌ | Mô hình Claude không có bản tải về để tự chạy trong máy chủ riêng (không mô hình nào công bố trọng số mở). Ảnh và văn bản của tài liệu **luôn phải đi tới** dịch vụ chạy Claude. Cách đúng: dùng Claude qua Amazon Bedrock hoặc Vertex AI để dữ liệu nằm trong ranh giới đám mây của công ty bạn, chọn vùng dữ liệu phù hợp. Trên Bedrock, AWS xử lý dữ liệu và không chia sẻ với Anthropic; còn “Claude Platform on AWS” thì dữ liệu do Anthropic xử lý bên ngoài ranh giới AWS. [AWS](https://aws.amazon.com/claude-platform/) |
| Anthropic không dùng dữ liệu thương mại để huấn luyện | ✅ | Mặc định không dùng dữ liệu từ các triển khai thương mại để huấn luyện. [Claude regional compliance](https://claude.com/regional-compliance) |
| Kiểm tra Data Processing Agreement | ✅ | Có DPA của Anthropic áp dụng cho các cách dùng thương mại. Với dữ liệu nhạy cảm, hỏi thêm về Zero Data Retention. |
| Lưu ý riêng cho Fable | ⚠️ | InfoQ (6/2026): dùng Fable 5 hoặc Mythos 5 trên Bedrock buộc bật chế độ chia sẻ dữ liệu suy luận cho Anthropic, lưu 30 ngày có người xem xét. Nếu hồ sơ rất nhạy cảm, kiểm tra điều kiện hiện hành của từng mô hình trước khi chọn. [InfoQ](https://infoq.com/news/2026/06/bedrock-fable-5-data-sharing) |

### A4. Về chi phí

| Nhận định | Kết quả | Chi tiết |
|---|---|---|
| Tính chi phí theo giá Sonnet 5.5 | ✅ | Sonnet 5.5: 2 USD / 10 USD mỗi triệu token (vào / ra). Đọc từ bộ nhớ đệm giảm còn 0,10 USD từ 7/10/2026. Batch API giảm 50%. [Unite.AI](https://www.unite.ai/?p=481687), [Docs giá](https://platform.claude.com/docs/en/about-claude/pricing?m=1) |
| Giá các bậc khác | ⚠️ | Opus 5.5 ~4 / 20 USD ([nguồn tổng hợp](https://claudefa.st/blog/models)); Haiku 5.5 ~0,10 / 0,50 USD ([developersdigest](https://www.developersdigest.tech/models/provider/anthropic)). Hãy đối chiếu bảng giá chính thức khi chốt. |

### A5. Về kiến trúc

| Thiết kế trong tài liệu gốc | Nhận xét |
|---|---|
| 4 giai đoạn: tiếp nhận, trích xuất, xác thực, bàn giao | ✅ Khung đúng và chuẩn. Giữ lại. |
| Orchestrator Agent + Validator Agent (đa tác nhân) | ⚠️ Phần lớn là quy trình cố định. “Kiểm toán viên” nên là **mã chương trình** (cộng tổng, kiểm tra MST, đối chiếu danh mục), rẻ và chắc chắn hơn AI. Chỉ dùng AI ở hai chỗ: đọc, và đọc lại vùng nghi ngờ. Đây là nhận định thiết kế của mình, không phải trích nguồn. |
| Người duyệt song song ảnh gốc và dữ liệu | ✅ Giữ nguyên. Boldare cũng làm vậy (“AI trích xuất, con người duyệt”). |
| Audit trail: phiên bản mô hình, thời điểm, JSON | ✅ Giữ nguyên. Boldare cũng ghi các mục này. |
| Gọi lại Claude với vùng ảnh cắt cho ô mờ | ✅ Giữ nguyên, ý hay và thực tế. |

---

## B. Thiết kế đã chỉnh

### B1. Nguyên tắc thiết kế

1. **AI chỉ làm việc AI giỏi** (đọc, hiểu bố cục). Việc có luật rõ ràng (cộng tổng, đối chiếu danh mục) giao cho mã chương trình.
2. **Chọn mô hình theo độ khó**, không dùng một mô hình cho mọi thứ.
3. **Luôn có người duyệt** trong giai đoạn đầu; chỉ cho tự động qua khi số đo đã chứng minh được.
4. **Đo trước, hứa sau**: mọi con số KPI phải đến từ bộ tài liệu mẫu của chính khách hàng.

### B2. Sơ đồ luồng

```
Đầu vào (PDF, scan, ảnh chụp, giấy viết tay)
   │
   ▼
[1] Tiếp nhận & phân loại ......... Haiku 5.5 xem trang đầu → loại giấy tờ → chọn schema
   │                                (file nhiều trang: tách theo tài liệu)
   ▼
[2] Trích xuất .................... Sonnet 5.5 đọc PDF/ảnh → JSON đúng schema + độ tự tin từng ô
   │        ▲                       Tài liệu khó / ô độ tự tin thấp → Opus 5.5
   │        │ hỏi lại vùng ảnh cắt
   ▼        │
[3] Kiểm tra ...................... Mã chương trình: schema, cộng tổng, MST, ngày, danh mục NCC
   │                                Ô nghi ngờ → quay lại [2] tối đa 1–2 lần
   ▼
[4] Người duyệt ................... Giao diện: ảnh gốc | dữ liệu | cờ cảnh báo → Sửa → Duyệt
   │
   ▼
Dữ liệu có cấu trúc → ERP / CRM / CSDL      (mọi bước ghi nhật ký kiểm toán)
```

### B3. Chi tiết từng giai đoạn

**Giai đoạn 1: Tiếp nhận và phân loại**
- Nguồn vào: tải lên qua web, email, thư mục theo dõi.
- Trước khi gọi AI: kiểm tra định dạng. Word hoặc Excel phải đổi sang PDF hoặc văn bản vì khối tài liệu không nhận định dạng nhị phân.
- Dùng mô hình nhẹ (Haiku 5.5) trả về `{loai: "hoa_don" | "hop_dong" | "bang_ke" | "phieu_viet_tay" | "khac", do_tu_tin: 0..1}`.
- Tài liệu loại “khác” hoặc độ tự tin thấp: chuyển thẳng cho người, không đoán.
- Tính hash của file để không xử lý trùng (Boldare cũng làm cách này).

**Giai đoạn 2: Trích xuất**
- Mỗi loại giấy tờ có một schema riêng. Ví dụ hóa đơn:

```json
{
  "so_hoa_don": "string",
  "ngay_hoa_don": "YYYY-MM-DD",
  "nha_cung_cap": { "ten": "string", "mst": "string" },
  "dong_hang": [
    { "ten_hang": "string", "so_luong": "number", "don_gia": "number", "thanh_tien": "number" }
  ],
  "tong_tien_hang": "number",
  "tien_vat": "number",
  "tong_thanh_toan": "number",
  "do_tu_tin": { "ten_truong": "0..1" },
  "ghi_chu_nghi_ngo": ["string"]
}
```

- Gọi API với `output_config.format` kiểu `json_schema` để luôn nhận JSON hợp lệ.
- Dặn rõ trong prompt: “nếu không đọc được thì để null và ghi vào ghi_chu_nghi_ngo, không được đoán”. Đây là cách giảm ảo giác quan trọng nhất.
- “Độ tự tin” do mô hình tự khai nên chỉ là **tín hiệu phụ**; tín hiệu chính là kết quả kiểm tra ở giai đoạn 3.
- Định tuyến: mặc định Sonnet 5.5; chuyển Opus 5.5 khi là chữ viết tay, bản scan xấu, hoặc giai đoạn 3 báo lỗi lần đầu. Fable chỉ cân nhắc cho bộ tài liệu rất khó vì giá cao và có điều kiện dữ liệu riêng.
- Khối lượng lớn, không gấp: dùng Batch API giảm 50%.
- Nội dung lặp lại (hướng dẫn, schema) đặt phía trước để tận dụng bộ nhớ đệm.

**Giai đoạn 3: Kiểm tra bằng mã chương trình**
Bộ luật tối thiểu cho hóa đơn:
1. JSON đúng schema (Zod hoặc Pydantic), đủ trường bắt buộc.
2. Tổng `thanh_tien` các dòng bằng `tong_tien_hang`.
3. `tong_tien_hang + tien_vat` bằng `tong_thanh_toan`; thuế suất nằm trong danh sách hợp lệ theo quy định hiện hành.
4. Mã số thuế đúng định dạng (10 hoặc 13 chữ số) và nên đối chiếu với danh mục.
5. Ngày hóa đơn hợp lý (không ở tương lai, không quá cũ).
6. Nhà cung cấp có trong danh mục; nếu chưa có thì gắn cờ vàng.

Hai mức cờ: **đỏ** (chặn duyệt tới khi sửa) và **vàng** (cảnh báo, người quyết định).

Khi có ô nghi ngờ: cắt vùng ảnh quanh ô đó, hỏi lại Claude một câu hẹp (“số trong ô khoanh đỏ là gì?”). Giới hạn 1–2 lần, sau đó chuyển cho người.

**Giai đoạn 4: Người duyệt và bàn giao**
- Giao diện hai cột: ảnh gốc bên trái, dữ liệu bên phải, ô có cờ được tô nổi và nhảy tới đúng vị trí trên ảnh.
- Nút **Duyệt** chỉ bật khi hết cờ đỏ.
- Mọi chỉnh sửa của người được lưu lại; đây là dữ liệu quý để cải thiện schema và prompt.
- Sau duyệt mới đẩy vào hệ thống đích (SQL, API ERP).
- Nhật ký kiểm toán: mã file, phiên bản mô hình, thời điểm, JSON gốc, JSON đã sửa, người duyệt.

### B4. Một lưu ý riêng cho bối cảnh Việt Nam (cần xác minh với khách)

Kiến thức của mình, chưa tra cứu cho tài liệu này: nhiều hóa đơn ở Việt Nam hiện là **hóa đơn điện tử có kèm tệp XML**. Nếu khách có sẵn XML thì nên đọc XML trước (chính xác 100% và gần như miễn phí), chỉ dùng AI cho bản giấy, bản scan, hoặc tệp chỉ có PDF. Hãy hỏi khách tỷ lệ này, vì nó quyết định giá trị thực của hệ thống. Chữ viết tay tiếng Việt có dấu cũng là điểm cần đo riêng.

---

## C. Đo lường (KPI) và cách đo

Tài liệu gốc đặt mục tiêu; ở đây thêm **cách chứng minh**.

| KPI | Mục tiêu khởi điểm (giả định, cần hiệu chỉnh) | Cách đo |
|---|---|---|
| Độ chính xác theo ô | ≥ 98% với tài liệu in; ≥ 90% với chữ viết tay | Đối chiếu với đáp án do người nhập tay 2 lần, trên 200 tài liệu |
| Tỷ lệ tài liệu “đúng hoàn toàn” | ≥ 85% | Số tài liệu không ô nào sai ÷ tổng |
| Tỷ lệ qua tự động (không cần sửa) | Bắt đầu 0%, tăng dần lên 50–70% khi dữ liệu chứng minh | Số tài liệu được duyệt mà không chỉnh ÷ tổng |
| Thời gian mỗi tài liệu | < 3 phút gồm cả duyệt | Đồng hồ trong giao diện duyệt |
| Chi phí mô hình mỗi tài liệu | < 0,05 USD | Cộng token từ trường `usage` của API |
| Lỗi lọt qua duyệt | < 0,5% | Lấy mẫu kiểm tra ngẫu nhiên hằng tuần |

Các mục tiêu “< 3 phút” và “< 0,5% lỗi” lấy cảm hứng từ Boldare nhưng **không thể coi là cam kết**, vì kết quả đó chưa nêu cỡ mẫu và là tài liệu khác. Hãy nói: “mục tiêu thử nghiệm, sẽ chốt sau khi đo trên bộ mẫu của khách”.

**Cách xây bộ mẫu đánh giá:** 200 tài liệu thật (đã che thông tin nhạy cảm), chia 4 nhóm (in rõ, scan xấu, bảng nhiều dòng, viết tay), người nhập tay hai lần để có đáp án chuẩn, chạy hệ thống, tính độ chính xác từng nhóm.

---

## D. Chi phí

**Giả định (đổi được):** 3 trang mỗi tài liệu, 2.500 token vào mỗi trang (từ khoảng 1.500–3.000 token văn bản mỗi trang theo docs, cộng phần ảnh), 700 token ra cho JSON.

Chi phí mô hình mỗi tài liệu (một lần gọi, không batch):

| Mô hình | Giá (vào / ra mỗi triệu token) | Mỗi tài liệu | 2.200 tài liệu/tháng | 22.000 tài liệu/tháng |
|---|---|---|---|---|
| Haiku 5.5 | 0,10 / 0,50 USD | 0,0011 USD | 2,42 USD | 24,2 USD |
| Sonnet 5.5 | 2 / 10 USD | 0,022 USD | 48,4 USD | 484 USD |
| Opus 5.5 | 4 / 20 USD | 0,044 USD | 96,8 USD | 968 USD |
| **Hỗn hợp** (Haiku phân loại 1 trang, 85% Sonnet, 15% Opus) | | ~0,026 USD | ~56 USD | ~563 USD |

Dùng Batch API giảm một nửa các con số trên khi không cần kết quả ngay.

**Tiết kiệm công:** nếu nhập tay mất 15 phút và duyệt mất 3 phút, mỗi tài liệu tiết kiệm 12 phút: 2.200 tài liệu = 440 giờ/tháng; 22.000 tài liệu = 4.400 giờ/tháng. (Các số phút là giả định lấy từ Boldare, hãy thay bằng đo thực tế.)

Kết luận: chi phí mô hình rất nhỏ so với công người. **Rủi ro chính là chất lượng và thời gian duyệt, không phải tiền AI.**

---

## E. Rủi ro và cách chặn

| Rủi ro | Cách chặn |
|---|---|
| Chữ viết tay xấu, tài liệu mờ, rách | Người duyệt bắt buộc; kiểm tra mã bắt lỗi số học; hỏi lại bằng ảnh cắt; chuyển lên Opus |
| Đọc sai số mà vẫn đúng định dạng | Quy tắc cộng tổng và đối chiếu; lấy mẫu kiểm tra hằng tuần; không bao giờ ghi thẳng vào ERP khi chưa qua kiểm tra |
| Ảo giác (bịa giá trị khi không thấy) | Prompt “không chắc thì null”; ô null buộc người điền; đo tỷ lệ bịa trên bộ mẫu |
| Tài liệu chứa câu lệnh độc hại (prompt injection, ví dụ dòng chữ “bỏ qua hướng dẫn trước”) | Coi toàn bộ nội dung tài liệu là dữ liệu, không phải lệnh; AI chỉ có quyền trả JSON, không có quyền gọi công cụ ghi; kiểm tra đầu ra bằng schema |
| Lộ dữ liệu nhạy cảm | Dùng Claude qua Bedrock hoặc Vertex AI, chọn vùng dữ liệu; hỏi về Zero Data Retention; che dữ liệu cá nhân không cần thiết; phân quyền người duyệt |
| Phụ thuộc một nhà cung cấp | Đóng gói lời gọi mô hình sau một lớp riêng; giữ bộ mẫu đánh giá để so sánh mô hình khác khi cần |
| Giới hạn kỹ thuật (số trang, kích thước) | Tách file dài thành nhiều PDF; giảm độ phân giải ảnh nhúng; kiểm tra giới hạn hiện hành trong docs |
| Mô hình đổi phiên bản làm kết quả lệch | Ghi phiên bản trong nhật ký; chạy lại bộ mẫu trước khi nâng cấp |

---

## F. Lộ trình gợi ý (4 tuần cho bản thử nghiệm)

| Tuần | Việc | Kết quả giao |
|---|---|---|
| 1 | Thu thập 200 tài liệu mẫu, chốt 1–2 loại giấy tờ, viết schema, người nhập đáp án chuẩn | Bộ mẫu + schema |
| 2 | Dựng luồng trích xuất (Sonnet) và bộ kiểm tra bằng mã; chạy bộ mẫu lần 1 | Báo cáo độ chính xác lần 1 |
| 3 | Thêm định tuyến Haiku/Opus, hỏi lại ảnh cắt, giao diện duyệt hai cột, nhật ký kiểm toán | Bản chạy được đầu–cuối |
| 4 | Chạy thử với người dùng thật, đo KPI, chỉnh prompt, báo cáo chi phí | Báo cáo KPI và đề xuất mở rộng |

---

## G. Cách đưa vào trang web hiện tại

Đã thực hiện trong trang web: thêm **chương 09 “Ứng dụng thực tế”**, đặt sau chương “Giới hạn” và trước “Lời của Claude”. Cấu trúc chương:

1. **Đặt vấn đề**: dữ liệu bị “giam” trong giấy tờ (3 câu, ngôn ngữ đời thường).
2. **Sơ đồ SVG** 4 giai đoạn, có nhãn mô hình ở từng bước và vòng “hỏi lại vùng ảnh mờ”.
3. **Demo tương tác**: chọn 1 trong 3 tài liệu mẫu (hóa đơn in, phiếu viết tay, bảng kê), bấm chạy để xem 4 giai đoạn; các phép kiểm tra được tính thật bằng JavaScript; ô đọc sai bị gắn cờ đỏ, người dùng sửa rồi bấm Duyệt. Có ghi rõ đây là dữ liệu mẫu, không gọi AI thật.
4. **Máy tính chi phí**: số tài liệu mỗi ngày, số trang, mô hình, Batch hay không; ra chi phí mỗi tài liệu, mỗi tháng, và số giờ công tiết kiệm.
5. **Vì sao chọn Claude** và **bằng chứng** (có ghi rõ mức độ tin cậy từng nguồn).
6. **KPI** và **rủi ro**.
7. **Lộ trình 4 tuần**.

Nếu bài nộp không dành cho Alliance ITSC, chỉ cần sửa dòng tiêu đề nhỏ của chương.

---

## Nguồn

- Boldare, case study trích xuất PO song ngữ: https://www.boldare.com/blog/automated-purchase-order-processing-claude-vision-databricks
- Claude Docs, PDF support: https://platform.claude.com/docs/en/build-with-claude/pdf-support
- Claude Docs, Structured outputs: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- Claude Docs, Pricing: https://platform.claude.com/docs/en/about-claude/pricing
- Giá cache đọc Sonnet 5.5 giảm còn 0,10 USD (Unite.AI): https://www.unite.ai/?p=481687
- Roboflow Vision Evals, Fable 5.2 vs Sonnet 5.5: https://playground.roboflow.com/models/compare/claude-fable-5-2-vs-claude-sonnet-5-5
- Roboflow, Opus 5.5: https://playground.roboflow.com/models/compare/claude-opus-5-5-vs-qwen2-5-vl-7b-instruct
- Roboflow, Sonnet 5.5 vs Gemini 4 Carbon: https://playground.roboflow.com/models/compare/claude-sonnet-5-5-vs-gemini-4-carbon
- Extend (IDP Software, nhận định về số tự báo cáo): https://idp-software.com/vendors/extend-ai/
- ExtractBench (LlamaIndex): https://llamaindex.ai/blog/introducing-extractbench
- Noteker, MCP server đọc chữ viết tay bằng Claude Vision: https://images.glama.ai/mcp/servers/andras-tkcs/noteker
- Hướng dẫn số hóa vở viết tay bằng Claude Code: https://www.termdock.com/blog/ai-cli-digitize-handwritten-notes
- AWS, Claude Platform vs Bedrock: https://aws.amazon.com/claude-platform/
- Claude regional compliance: https://claude.com/regional-compliance
- InfoQ, Fable 5 trên Bedrock và chia sẻ dữ liệu: https://infoq.com/news/2026/06/bedrock-fable-5-data-sharing
- Tổng hợp mô hình Anthropic (Haiku 5.5): https://www.gradually.ai/en/ai-models/anthropic/
- Tổng hợp giá mô hình: https://claudefa.st/blog/models và https://www.developersdigest.tech/models/provider/anthropic
