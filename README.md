# TrackAsia TMS Client API – Kế hoạch tích hợp & tài liệu tham chiếu (v2)

> **Mục tiêu tài liệu**  
> Cung cấp một bộ hướng dẫn tích hợp cho đối tác khi kết nối nhóm API *Client* của TrackAsia TMS. Tài liệu tập trung vào **đặc tả API**, **mô hình dữ liệu**, **quy trình triển khai**, **tiêu chuẩn bảo mật**, kèm **hình minh họa** đã chuẩn bị sẵn. (Loại bỏ các flowchart tự sinh; dùng hình chính xác do bạn cung cấp.)


## 1) Phạm vi & nguyên tắc
- **Base URL (prod)**: `https://tms.track-asia.com/api/v1`
- **Module thuộc phạm vi**: Auth & Account, Bookings (CRUD), Notifications, Catalogues, Devices, Upload base64.
- **Content-Type**: `application/json` hoặc `application/x-www-form-urlencoded` (ghi rõ theo endpoint).
- **Auth**: Header `Authorization: {token_user}` cho các endpoint yêu cầu xác thực (không prefix "Bearer").
- **Không thuộc phạm vi**: API nội bộ khác, webhooks/push ngoài thông báo hệ thống, quy trình vận hành nội bộ doanh nghiệp.

---

## 2) Sơ đồ tổng quan tích hợp
*Hình thức: poster tổng thể, nhóm chức năng và mối liên hệ Partner ↔ TrackAsia API ↔ Data/Status.*

![Sơ đồ tổng hợp tích hợp API](https://github.com/sangnguyen-it/TRACKASIA-CLIENT-INTEGRATED-DEPLOYMENT-DOCS/blob/main/images/sodo-all.jpg)

---

## 3) Luồng Booking end‑to‑end
*Minh họa các bước: Auth → Tạo đơn → Theo dõi → Chi tiết → Cập nhật → Huỷ → Trạng thái.*

![Sơ đồ luồng Booking](https://github.com/sangnguyen-it/TRACKASIA-CLIENT-INTEGRATED-DEPLOYMENT-DOCS/blob/main/images/sodo-booking.jpg)

**Bảng trạng thái (tham chiếu nhanh)**

| Code | VN | EN |
|---|---|---|
| pending | Chờ xử lý | Pending |
| confirm | Đã xác nhận | Confirmed |
| waiting_get_item | Chờ lấy hàng | Waiting to pick up |
| getting_item | Đang lấy hàng | Getting item |
| got_item | Lấy hàng thành công | Got item |
| waiting_delivery | Chuẩn bị giao | Waiting delivery |
| delivering | Đang giao | Delivering |
| delivered | Đã giao | Delivered |
| completed | Hoàn tất | Completed |
| canceled | Đã huỷ | Canceled |
| rejected | Bị từ chối | Rejected |
| not_yet_delivery | Chưa giao được | Not yet delivered |
| rejected_item | Từ chối nhận | Rejected item |

---

## 4) "Nhắc việc" tích hợp theo nhóm API
*Checklist trực quan cho đội tích hợp – hiển thị headers chung, nhóm endpoint và HTTP codes.*

![Sơ đồ nhắc việc tích hợp API](https://github.com/sangnguyen-it/TRACKASIA-CLIENT-INTEGRATED-DEPLOYMENT-DOCS/blob/main/images/sodo-other.jpg)

---

## 5) Đặc tả kỹ thuật (API Reference)

### 5.1 Headers & HTTP Codes
**Headers chung**

| Header | Giá trị |
|---|---|
| `Authorization` | `{token_user}` (bắt buộc nếu endpoint yêu cầu auth) |
| `Content-Type` | `application/json` **hoặc** `application/x-www-form-urlencoded` |

**HTTP codes**: `200/201` OK/Created · `400/422` Invalid · `401` Unauthorized (→ re-login) · `403` Forbidden · `404` Not Found · `500` Server Error.

**Khung lỗi chuẩn**
```json
{
  "error": "Error type",
  "message": "Chi tiết lỗi cụ thể"
}
```

### 5.2 Auth & Account
- **POST** `/clients/login` *(form)* → nhận `data.token_user`; Body: `loginkey`, `password`, `device_id`.
- **POST** `/clients/logout` *(auth)* → vô hiệu hoá token hiện tại.
- **POST** `/clients/update-current-password` *(form, auth)* → `current_password`, `new_password`, `new_confirm_password`.
- **POST** `/clients/update-user` *(form, auth)* → một số trường thường gặp: `name`, `address`, `birthday(YYYY-MM-DD)`, `email`.
- **POST** `/clients/update-avarta` *(form, auth)* → `img` (base64 **data-only**), `img_format` (`jpeg|jpg|png`).

### 5.3 Bookings (CRUD)
- **GET** `/clients/bookings` *(auth)*: lọc `status[]=...` (đa giá trị).
- **GET** `/clients/bookings/{id}` *(auth)*: chi tiết đơn.
- **POST** `/clients/bookings` *(JSON, auth)*: tối thiểu `address_from`, `address_to`, `schedule_time (YYYY-MM-DD)`, `contact_number`, `locations_attributes[]`. Khuyến nghị `reference_no` (idempotency). 
  - `locations_attributes[]`: `{ name, latitude, longitude, position(1=đi,2=đến,>=3=dừng), [distance], [id], [_destroy] }`
- **PUT** `/clients/bookings/{id}` *(JSON, auth)*: chỉ gửi trường cần cập nhật; xoá điểm dừng bằng `{ "id": <id>, "_destroy": true }`.
- **DELETE** `/clients/bookings/{id}` *(auth)*.

### 5.4 Notifications
- **GET** `/clients/notifications?page=1&limit=20` *(auth)* – phân trang.
- **GET** `/clients/notifications/{id}` *(auth)* – chi tiết.
- **DELETE** `/clients/notifications/{id}` *(auth)* – xoá.

### 5.5 Catalogues
- **GET** `/clients/catalogues` *(auth)* – khuyến nghị **cache** cục bộ.

### 5.6 Devices
- **POST** `/devices` *(form, no auth)* → `device_type`, `device_id`, `device_name`, `[os_version]`, `[app_version]`, `[device_token]`.
- **PUT** `/devices/{id}` *(form, auth)* → `company_id`.

### 5.7 Upload base64
- **POST** `/upload-base64` *(form, auth)* → `data` (base64 **data-only**), `format` (`jpeg|png|pdf`...).

---

## 6) Mô hình dữ liệu (Data Models)

### 6.1 User
`{ id, name, phone_number, email, token_user, image, address, birthday, current_device_id }`

### 6.2 Booking
Các trường nổi bật: `id, booking_code, status, contact_number, company_name, customer_email, charges, distance, weight, volume, quantity, description, reference_no, catalogue_id, schedule_time, etd_time, eta_time, location_from, location_to, locations_attributes[], updated_at ...`

- **location_from/location_to**: `{ id, name, latitude, longitude, position, distance }`
- **locations_attributes[]** khi cập nhật: cho phép `{ id, _destroy: true }` để xoá điểm.

### 6.3 Notification
`{ id, target_id, title, content, booking_code, view_type, action_type, reason_reject, created_at }`

### 6.4 Catalogue
`{ id, name }`

---

## 7) Khuyến nghị triển khai (Best Practices)
- **Bảo mật token**: lưu ở secure store (Keychain/EncryptedSharedPreferences); mask trong log; xoá token khi 401.
- **Idempotency**: dùng `reference_no` duy nhất khi tạo đơn; chặn trùng phía đối tác.
- **Chuẩn hóa số liệu**: parse `distance`, `weight`, `volume`, `quantity` về kiểu số.
- **Địa lý**: luôn kèm `latitude/longitude` hợp lệ; `position` tăng dần theo lộ trình.
- **Cache** `catalogues`; giảm số lần gọi lặp.
- **Upload**: chỉ truyền **BASE64 data-only**; kiểm soát `format`; giới hạn kích thước (khuyến nghị ≤ 5MB/ảnh).

---

## 8) Quy trình tích hợp (Step‑by‑step)
1. **Login** bằng `/clients/login` → lưu `token_user` an toàn.
2. **Tải Catalogues** → cache để dựng UI.
3. **Tạo Booking** (JSON tối thiểu) → nhận `id`, `booking_code`.
4. **Theo dõi/Tra cứu** danh sách & chi tiết bằng `status[]` và `{id}`.
5. **Cập nhật/Huỷ** booking theo nghiệp vụ.
6. **Đồng bộ Notifications** (phân trang); hiển thị liên kết đến đơn liên quan.
7. **Quản trị Account** (đổi mật khẩu, hồ sơ, avatar) & **Devices**.
8. **Upload base64** tài liệu/POD và liên kết vào nghiệp vụ nội bộ.

---

## 9) Kiểm thử & nghiệm thu
**Test case tối thiểu**
- Auth: sai mật khẩu, đúng; logout; 401 → re-login.
- Booking: tạo tối thiểu/đủ; cập nhật một điểm; xoá điểm (`_destroy`); huỷ đơn.
- Lọc `status[]` nhiều giá trị.
- Notifications: phân trang, đọc, xoá.
- Catalogues: tải + cache; hiển thị mapping đúng.
- Upload base64: gửi ảnh/pdf hợp lệ; chặn sai định dạng.

**Tiêu chí pass**: 2xx ở đường đi hạnh phúc; mapping trạng thái đúng; log sạch; không rò rỉ PII/token.

---

## 10) Ví dụ nhanh
**cURL – Login**
```bash
curl -X POST "https://tms.track-asia.com/api/v1/clients/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "loginkey=user@example.com&password=yourpassword&device_id=device-uuid"
```

**JS – Lấy bookings theo trạng thái**
```javascript
fetch('https://tms.track-asia.com/api/v1/clients/bookings?status[]=pending&status[]=delivering', {
  headers: { Authorization: token }
}).then(r => r.json()).then(console.log);
```

**Python – Tạo booking tối thiểu**
```python
import requests
BASE='https://tms.track-asia.com/api/v1'
headers={'Authorization': token, 'Content-Type': 'application/json'}
body={
  "address_from":"123 Nguyễn Huệ, Q1",
  "address_to":"456 Lê Lợi, Q3",
  "schedule_time":"2025-12-10",
  "contact_number":"0912345678",
  "locations_attributes":[
    {"name":"123 Nguyễn Huệ","latitude":10.7769,"longitude":106.7009,"position":1},
    {"name":"456 Lê Lợi","latitude":10.7756,"longitude":106.6867,"position":2}
  ]
}
print(requests.post(f"{BASE}/clients/bookings", headers=headers, json=body).json())
```

---

## 11) Phụ lục – Mẫu payload nhanh
- **/clients/update-avarta (form)**: `img=<BASE64_ONLY>&img_format=jpeg|png|jpg`
- **/upload-base64 (form)**: `data=<BASE64_ONLY>&format=jpeg|png|pdf`
- **locations_attributes** khi xoá điểm: `[{"id": 33, "_destroy": true}]`

---

*Phiên bản 2.0 – tái cấu trúc theo chuẩn API reference, chèn hình minh hoạ chính xác; giữ nguyên nội dung kỹ thuật cốt lõi.*

