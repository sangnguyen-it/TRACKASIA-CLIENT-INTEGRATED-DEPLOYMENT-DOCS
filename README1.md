# TrackAsia Client API – Hướng dẫn tích hợp cho đối tác (v1)

> **Mục tiêu tài liệu**: Cung cấp một bản **mô tả API và dữ liệu triển khai** thật rõ ràng, kèm đánh giá/khuyến nghị tích hợp để đội kỹ thuật đối tác có thể **kết nối nhanh, đúng, ổn định** với nhóm API *Client* của TrackAsia. Tài liệu này **loại bỏ sơ đồ/flowchart**, tập trung thuần vào đặc tả API, mô hình dữ liệu, quy trình tích hợp, kiểm thử và vận hành.

---

## 1) Phạm vi & nguyên tắc chung
- **Base URL (prod):** `https://tms.track-asia.com/api/v1`
- **Module trong phạm vi:** Client Auth & Account, Bookings, Notifications, Catalogues, Devices, Upload base64.
- **Định dạng:** JSON (đa số) và `application/x-www-form-urlencoded` (một số endpoint nêu rõ bên dưới).
- **Xác thực:** Sau khi đăng nhập, mọi request cần auth gửi header `Authorization: {token_user}`.
- **Phiên bản:** `v1` (ghi trong URL). Khuyến nghị đóng gói adapter để tiện nâng cấp.

---

## 2) Mục tiêu triển khai (Deliverables)
1. Kết nối đăng nhập/đăng xuất, lưu và tự làm mới `token_user` khi cần.
2. Tạo – cập nhật – huỷ – tra cứu **Booking** với dữ liệu địa chỉ/điểm dừng hợp lệ.
3. Đồng bộ **Catalogue** (danh mục) phục vụ chọn loại hàng/nhóm dịch vụ.
4. Kéo **Notification** có phân trang; hiển thị theo ngữ cảnh.
5. Quản trị **tài khoản** (đổi mật khẩu, cập nhật hồ sơ, avatar) và **thiết bị**.
6. Tải tệp dạng base64 (hoá đơn, POD, ảnh chứng từ) an toàn, đúng định dạng.
7. Có bộ kiểm thử tối thiểu, checklist và quy trình xử lý lỗi thống nhất.

---

## 3) Đặc tả API
### 3.1 Headers chung
| Header | Giá trị |
|---|---|
| `Content-Type` | `application/json` **hoặc** `application/x-www-form-urlencoded` (nêu rõ từng endpoint) |
| `Authorization` | `{token_user}` (bắt buộc với endpoint yêu cầu xác thực) |

### 3.2 Mã trạng thái HTTP
`200/201` thành công · `400/422` sai dữ liệu · `401` token sai/hết hạn · `403` không có quyền · `404` không thấy · `500` lỗi server.

### 3.3 Định dạng lỗi chuẩn (khuyến nghị xử lý)
```json
{
  "error": "Error description",
  "message": "Detailed error message"
}
```
- UI/Client nên hiển thị ngắn gọn từ `message` (nếu an toàn), log đầy đủ cho dev, tránh lộ dữ liệu nhạy cảm.

---

### 3.4 Auth & Account
#### POST `/clients/login`  *(x-www-form-urlencoded)*
**Body**
| Trường | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `loginkey` | string | ✅ | Email hoặc số điện thoại |
| `password` | string | ✅ |  |
| `device_id` | string | ✅ | UUID thiết bị |

**Response mẫu (rút gọn)**
```json
{
  "data": {
    "token_user": "...",
    "user": {"id": 12, "name": "Nguyễn Văn A", "email": "a@example.com"}
  }
}
```

#### POST `/clients/logout`
- Vô hiệu hoá token hiện tại.

#### POST `/clients/update-current-password` *(x-www-form-urlencoded)*
| Trường | Kiểu | Bắt buộc |
|---|---|---|
| `current_password` | string | ✅ |
| `new_password` | string | ✅ |
| `new_confirm_password` | string | ✅ |

#### POST `/clients/update-user` *(x-www-form-urlencoded)*
- Một số trường thường gặp: `name`, `address`, `birthday(YYYY-MM-DD)`, `email`.

#### POST `/clients/update-avarta` *(x-www-form-urlencoded)*
| Trường | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `img` | base64 string | ✅ | **Chỉ** phần dữ liệu, không kèm `data:image/...;base64,` |
| `img_format` | enum | ✅ | `jpeg`/`jpg`/`png` |

---

### 3.5 Bookings
#### 3.5.1 Tạo booking – POST `/clients/bookings` *(application/json)*
**Body (tối thiểu khuyến nghị)**
| Trường | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `address_from` | string | ✅ | Địa chỉ lấy |
| `address_to` | string | ✅ | Địa chỉ giao |
| `schedule_time` | `YYYY-MM-DD` | ✅ | Ngày hẹn |
| `contact_number` | string | ✅ | SĐT liên hệ |
| `locations_attributes` | array | ✅ | Xem cấu trúc bên dưới |
| `reference_no` | string | ➕ | Khuyến nghị đặt duy nhất để đối soát |
| `company_name` | string | ➖ | Tuỳ nhu cầu |
| `charges` | number/string | ➖ | Phí tạm tính/thoả thuận |
| `description` | string | ➖ | Ghi chú |
| `distance` | number | ➖ | Km (nếu có) |
| `etd_time` | `YYYY-MM-DD HH:mm` | ➖ | Dự kiến lấy |
| `eta_time` | `YYYY-MM-DD HH:mm` | ➖ | Dự kiến giao |
| `quantity` | number | ➖ | Số kiện |
| `weight` | number | ➖ | Kg |
| `volume` | number | ➖ | m³ |

**`locations_attributes[]`**
| Trường | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `name` | string | ✅ | Tên/địa chỉ điểm |
| `latitude` | number | ✅ |  |
| `longitude` | number | ✅ |  |
| `position` | number | ✅ | 0: điểm đi, 1: điểm đến, ≥2: điểm dừng |
| `distance` | number | ➖ | Tuỳ chọn |
| `id`, `_destroy` | number, bool | ➖ | Dùng khi cập nhật/xoá điểm |

**Response (rút gọn)**: trả về `id`, `booking_code`, các trường đã lưu.

#### 3.5.2 Cập nhật booking – PUT `/clients/bookings/{id}` *(application/json)*
- Gửi **chỉ các trường cần đổi**. Muốn xoá điểm dừng: phần tử `{"id": <id>, "_destroy": true}` trong `locations_attributes`.

#### 3.5.3 Huỷ booking – DELETE `/clients/bookings/{id}`

#### 3.5.4 Danh sách booking – GET `/clients/bookings`
- Hỗ trợ lọc theo `status[]=...` nhiều giá trị.

#### 3.5.5 Chi tiết booking – GET `/clients/bookings/{id}`

**Enum trạng thái chuẩn (mapping UI khuyến nghị)**
| Code | VN | EN |
|---|---|---|
| `pending` | Chờ xử lý | Pending |
| `confirm` | Đã xác nhận | Confirmed |
| `waiting_get_item` | Chờ lấy hàng | Waiting to pick up |
| `getting_item` | Đang lấy hàng | Getting item |
| `got_item` | Lấy hàng thành công | Got item |
| `waiting_delivery` | Chuẩn bị giao | Waiting delivery |
| `delivering` | Đang giao | Delivering |
| `delivered` | Đã giao | Delivered |
| `completed` | Hoàn tất | Completed |
| `canceled` | Đã huỷ | Canceled |
| `rejected` | Bị từ chối | Rejected |
| `not_yet_delivery` | Chưa giao được | Not yet delivered |
| `rejected_item` | Từ chối nhận | Rejected item |

**Quy ước số liệu**: Parse về dạng số trước khi lưu (`distance`, `weight`, `volume`, `quantity`).

---

### 3.6 Notifications
#### GET `/clients/notifications?page=1&limit=20`
- **Phân trang** qua `page` & `limit`.
- Có thể có các trường như `view_type`, `action_type` để định tuyến UI.

#### GET `/clients/notifications/{id}` – Chi tiết
#### DELETE `/clients/notifications/{id}` – Xoá

---

### 3.7 Catalogues
#### GET `/clients/catalogues`
- Dùng để **cache** cục bộ; cập nhật định kỳ khi phiên bắt đầu hoặc theo chu kỳ (ví dụ mỗi 24h).

---

### 3.8 Devices
#### POST `/devices` *(x-www-form-urlencoded, không cần auth)*
| Trường | Kiểu | Bắt buộc |
|---|---|---|
| `device_type` | enum | ✅ | `android`/`ios`/`web` (ví dụ) |
| `device_id` | string | ✅ |
| `device_name` | string | ✅ |
| `os_version` | string | ➖ |
| `app_version` | string | ➖ |
| `device_token` | string | ➖ | Token FCM/APNs |

#### PUT `/devices/{id}` *(x-www-form-urlencoded, cần auth)*
- Ví dụ trường thường gặp: `company_id`.

---

### 3.9 Upload base64
#### POST `/upload-base64` *(x-www-form-urlencoded)*
| Trường | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `data` | base64 string | ✅ | **Chỉ dữ liệu**, không kèm prefix |
| `format` | enum | ✅ | `jpeg`/`png`/`pdf` ... |

---

## 4) Khuyến nghị triển khai (Analysis & Best Practices)
### 4.1 Quản lý phiên & bảo mật
- Lưu `token_user` trong **server-side secure store**. Không đặt trong localStorage phía trình duyệt công cộng.
- Khi trả về `401`, thực hiện login lại **một lần** có backoff; nếu vẫn lỗi → báo người dùng & dừng retry.
- Ẩn/mask token, mật khẩu, SĐT trong log; chuẩn hoá **PII policy**.

### 4.2 Idempotency & đối soát
- Đính `reference_no` duy nhất khi tạo đơn. Hệ thống đối tác nên chặn trùng theo `reference_no`.

### 4.3 Dữ liệu địa lý & tuyến đường
- Bắt buộc `latitude/longitude` hợp lệ; `position` theo quy ước (0→1→…)
- Với nhiều điểm dừng, sắp xếp theo `position` tăng dần.

### 4.4 Phân trang, cache & hiệu năng
- **Cache** `catalogues`; tránh gọi lặp lại.
- Tải **notifications** theo trang; khi cần cập nhật realtime có thể tăng tần suất polling ở tầng ứng dụng đối tác.

### 4.5 Upload tệp an toàn
- Chỉ truyền **phần base64** (không có prefix). Giới hạn kích thước (khuyến nghị ≤ 5MB/ảnh).
- Kiểm soát loại tệp theo whitelist `format`.

---

## 5) Quy trình tích hợp (Step-by-step)
1) **Login** bằng `/clients/login` → lưu `token_user`.
2) **Đồng bộ catalogues** để dựng UI đặt đơn.
3) **Tạo booking** bằng JSON tối thiểu; ghi nhận `id`, `booking_code`.
4) **Tra cứu** danh sách/chi tiết booking; lọc `status[]` khi cần.
5) **Cập nhật/huỷ** booking theo tình huống nghiệp vụ.
6) **Đồng bộ notifications** (phân trang); cho phép mở chi tiết liên quan.
7) **Quản trị tài khoản** (đổi mật khẩu, hồ sơ, avatar) & **thiết bị**.
8) **Upload base64** (POD/chứng từ), liên kết bản ghi ở tầng nghiệp vụ của đối tác.

---

## 6) Kiểm thử & nghiệm thu
**Test tối thiểu**
- Auth: login sai/đúng, logout, token hết hạn → re-login.
- Booking: tạo tối thiểu/đủ, cập nhật 1 điểm dừng, xoá điểm dừng (`_destroy`), huỷ đơn.
- Lọc theo `status[]` nhiều giá trị.
- Notifications: phân trang, đọc/xoá.
- Catalogues: tải & cache; so khớp hiển thị.
- Upload base64: ảnh/pdf hợp lệ & bị chặn khi sai định dạng.

**Tiêu chí pass**
- 100% API trong phạm vi trả về 2xx ở đường đi hạnh phúc.
- Log sạch, không rò rỉ PII/token.
- UI hiển thị trạng thái theo bảng enum.

---

## 7) Ví dụ cắt gọn
**Login (cURL)**
```bash
curl -X POST "https://tms.track-asia.com/api/v1/clients/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "loginkey=user@example.com&password=yourpassword&device_id=device-uuid"
```

**GET bookings (JS fetch)**
```javascript
fetch('https://tms.track-asia.com/api/v1/clients/bookings?status[]=pending&status[]=delivering', {
  headers: { Authorization: token }
}).then(r => r.json()).then(console.log);
```

**POST booking (Python)**
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
    {"name":"123 Nguyễn Huệ","latitude":10.7769,"longitude":106.7009,"position":0},
    {"name":"456 Lê Lợi","latitude":10.7756,"longitude":106.6867,"position":1}
  ]
}
print(requests.post(f"{BASE}/clients/bookings", headers=headers, json=body).json())
```

---

## 8) Tài nguyên đi kèm
- **Postman collection**: `TrackAsia_Client_API_Postman.json`
- **Mẫu payload nhanh**
  - `/clients/update-avarta` *(form)*: `img=<BASE64_ONLY>&img_format=jpeg|png|jpg`
  - `/upload-base64` *(form)*: `data=<BASE64_ONLY>&format=jpeg|png|pdf`
  - `locations_attributes` khi cập nhật xoá điểm: `[{"id": 33, "_destroy": true}]`

*Phiên bản tài liệu: 1.1 (tập trung đặc tả API & dữ liệu, loại bỏ sơ đồ).*

