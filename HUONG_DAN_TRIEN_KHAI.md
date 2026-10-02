# Hướng dẫn triển khai — IT Renewal Management | Nguồn: LHH JSC

Bộ 3 file đi kèm:

- `renewals-sheet-template.xlsx` — dữ liệu 24 mục đã dọn sạch từ file excel, sẵn sàng import vào Google Sheets.
- `Code.gs` — backend Apps Script: trả dữ liệu JSON cho web app, nhận cập nhật khi bấm "Gia hạn", và gửi email nhắc trước hạn.
- `index.html` — trang dashboard, deploy lên Netlify.

Làm theo đúng thứ tự bên dưới — mỗi bước phụ thuộc bước trước.

## Bước 1 — Đưa dữ liệu vào Google Sheets

1. Vào [drive.google.com](https://drive.google.com), bấm **New → File upload**, chọn `renewals-sheet-template.xlsx`.
2. Nhấp đúp vào file vừa tải lên → chọn **Open with Google Sheets** (Drive tự chuyển sang định dạng Sheets).
3. Kiểm tra tên tab dữ liệu phải là **`Renewals`** (đã đặt sẵn đúng tên — không đổi tên tab này, vì `Code.gs` tìm theo đúng tên đó).
4. Sheet có sẵn 24 dòng dữ liệu thật + 40 dòng trống đã định dạng sẵn để nhập thêm. 

## Bước 2 — Gắn Apps Script vào Sheet

1. Trong Google Sheet, vào menu **Extensions → Apps Script**.
2. Xoá nội dung mặc định trong file `Code.gs` của trình soạn thảo, dán toàn bộ nội dung file `Code.gs` vào.
3. Bấm biểu tượng 💾 **Save**.
4. Vào **Project Settings** (biểu tượng bánh răng bên trái) → mục **Script Properties** → **Add script property**, thêm:
   - `NOTIFY_EMAIL` = email bạn muốn nhận thông báo (ví dụ `lamhiephung88@gmail.com`)
   - `LEAD_DAYS` = `7`

## Bước 3 — Deploy Apps Script thành Web App

1. Quay lại tab **Editor**, chọn hàm `installDailyTrigger` ở thanh chọn hàm trên cùng, bấm **Run**. Lần đầu chạy, Google sẽ yêu cầu cấp quyền — chọn tài khoản của bạn → **Advanced** → **Go to (tên project) (unsafe)** → **Allow**. Đây là bước bình thường vì script chưa được Google xác minh công khai.
2. Bấm nút **Deploy → New deployment**.
3. Chọn loại **Web app**.
4. Thiết lập:
   - **Execute as**: Me 
   - **Who has access**: Anyone
5. Bấm **Deploy**, cấp quyền lần nữa nếu được hỏi.
6. Copy **Web app URL** hiện ra (dạng `https://script.google.com/macros/s/xxxxx/exec`) — đây là URL sẽ dán vào `index.html`.

> Mỗi lần sửa `Code.gs` sau này, phải **Deploy → Manage deployments → biểu tượng bút chì → New version → Deploy** thì thay đổi mới có hiệu lực trên URL cũ.

## Bước 4 — Cấu hình `index.html`

1. Mở `index.html` bằng trình soạn thảo text bất kỳ (Notepad, VS Code…).
2. Tìm đoạn:
   ```js
   const CONFIG = {
     WEB_APP_URL: "",
     LEAD_DAYS: 7
   };
   ```
3. Dán URL đã copy ở Bước 3 vào giữa hai dấu ngoặc kép:
   ```js
   const CONFIG = {
     WEB_APP_URL: "https://script.google.com/macros/s/xxxxx/exec",
     LEAD_DAYS: 7
   };
   ```
4. Lưu file.

## Bước 5 — Deploy lên Netlify

Cách nhanh nhất, không cần tài khoản GitHub:

1. Vào [app.netlify.com/drop](https://app.netlify.com/drop).
2. Kéo thả file `index.html` (hoặc cả thư mục chứa nó) vào khung trên trang.
3. Netlify tự tạo một URL dạng `https://random-name-xxxx.netlify.app` — trang chạy ngay lập tức.
4. (Tuỳ chọn) Vào **Site settings → Change site name** để đổi thành tên dễ nhớ hơn, ví dụ `it-renewal-lhh.netlify.app`.

## Bước 6 — Kiểm tra

1. Mở URL Netlify vừa tạo — dashboard phải hiển thị đúng 24 mục, số liệu ở "System Status" khớp với dữ liệu trong Sheet.
2. Thử bấm **Gia hạn** trên một dòng, chọn ngày mới, xác nhận → mở lại Google Sheet, kiểm tra cột `ExpiryDate` đã đổi.
3. Để kiểm tra email nhắc hạn ngay mà không cần chờ đến 7 giờ sáng hôm sau: trong Apps Script editor, chọn hàm `dailyCheckAndNotify`, bấm **Run** — nếu có mục nào đủ điều kiện (≤ 7 ngày hoặc quá hạn), email sẽ được gửi ngay tới địa chỉ đã cấu hình.

## Vận hành hàng ngày

- Sửa ngày hết hạn, thêm dịch vụ mới, đổi trạng thái… làm trực tiếp trong Google Sheet như bình thường (hoặc dùng nút "Gia hạn" trên dashboard) — dashboard luôn đọc dữ liệu mới nhất, tự làm mới mỗi 5 phút.
- Trigger `dailyCheckAndNotify` chạy tự động lúc 7:00 sáng mỗi ngày, gửi email nếu có mục đến hạn trong vòng `LEAD_DAYS` ngày (mặc định 7) hoặc đã quá hạn — mỗi mục chỉ gửi tối đa 1 email/ngày.
- Muốn ngừng theo dõi một dịch vụ mà không xoá dữ liệu: đổi cột `Status` của dòng đó thành `cancelled`.
- Hết 40 dòng trống có sẵn: chọn dòng cuối cùng, copy rồi dán (Paste, không phải Paste values) xuống các dòng tiếp theo để giữ định dạng.

## Xử lý sự cố thường gặp

| Hiện tượng | Nguyên nhân thường gặp |
|---|---|
| Dashboard báo "MẤT KẾT NỐI" | URL trong `CONFIG.WEB_APP_URL` sai, hoặc deployment chưa ở chế độ "Anyone" |
| Bấm "Gia hạn" báo lỗi | Deployment Apps Script chưa được **New version** sau khi sửa `Code.gs` |
| Không nhận được email | Chưa chạy `installDailyTrigger`, hoặc `NOTIFY_EMAIL` chưa thiết lập trong Script Properties |
| Dữ liệu không đổi sau khi sửa Sheet | Dashboard cache 5 phút — bấm nút **⟳ Làm mới** để cập nhật ngay |


Nếu gặp bất kỳ khó khăn gì, bạn có thể liên hệ trực tiếp tôi qua github eagle-nett
