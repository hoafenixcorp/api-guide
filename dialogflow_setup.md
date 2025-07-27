# Hướng Dẫn Cài Đặt Dialogflow CX Cho Dự Án

## 1. Cài Đặt Tài Khoản Và Service Account

### 1.1 Cài Đặt Tài Khoản

- Đăng ký tài khoản GCP (cần Visa/Mastercard)
- Truy cập GCP Console: [GCP Console](https://console.cloud.google.com/)
- Tạo một project mới

![GCP Dashboard](https://github.com/user-attachments/assets/b117426b-9ff6-44c4-b118-e400e6ff2612)

- Bật các API cần thiết:
  - Dialogflow API
  - Cloud Run Admin API
  - Cloud Functions API
  - Cloud Build API
  - Cloud Text-to-Speech API
  - Cloud Speech-to-Text API

### 1.2 Tạo Service Account

- Truy cập menu Service Account trong dashboard

![Service Account Tab](https://github.com/user-attachments/assets/77ea3999-ae11-4ee6-b9e5-4991333759f8)

- Tạo service account mới và gán các role cần thiết

![Assign Role](https://github.com/user-attachments/assets/6d2ea3e4-60cf-4fbd-9d53-b0a40d70d5c5)

- Tạo JSON key:
  - Vào tab Key, chọn "Add Key" → JSON

![Create JSON Key](https://github.com/user-attachments/assets/e477b278-33e5-439e-9f0e-85563c927c72)

---

## 2. Thiết Lập Dự Án Conversational AI

- Truy cập [Conversational AI Console](https://conversational-agents.cloud.google.com/projects)
- Tạo project mới và liên kết với project GCP

![New Project](https://github.com/user-attachments/assets/585a9c06-f8f7-46d2-a7b4-6e4779b7e2c3)

- Tạo Agent mới, chọn language code (ja) và region (asia)
- Có thể tạo 2 project riêng cho môi trường Staging/Production

![Agent Setting](https://github.com/user-attachments/assets/65e73429-c6d4-4ac7-90f1-bcfc9f9758b4)

### 2.1 Thiết Lập MemberCodeVerification Flow

**Prompt format:** `［3CXの質問］: 会員番号を読んでください。［答え］: {text}。［ホットライン］: {hotline}`

#### a. Entities

- Dùng để định nghĩa data type giúp nhận diện intent
- Ví dụ:
  - `verify-member-request` map với `［3CXの質問］: 会員番号を読んでください。`
  - `verify-member-hotline` map với `［ホットライン］`

![Entities Example](https://github.com/user-attachments/assets/a5882266-9cc4-499b-9147-17cfb178a245)

#### b. Webhooks

- Dùng để kết nối với API verify_member_code từ backend Laravel

![Webhook Setup](https://github.com/user-attachments/assets/ec3d35b0-fd6c-4074-8afc-25b06bdfa2c2)

#### c. Intents

- Dùng để huấn luyện chatbot hiểu ý định của người dùng

![Create Intent](https://github.com/user-attachments/assets/f5ffa44c-0d85-41a5-bc86-89920538e9c5)

- Chèn các training phrase đa dạng, ví dụ: số liền nhau, có khoảng trắng, chữ tiếng Nhật, v.v.
- Map đoạn văn bản với entity phù hợp:

![Training Phrase](https://github.com/user-attachments/assets/c2ca7468-e0e1-465b-bba7-b5dccc13d42b)

**Lưu ý:**
- `@sys` là entity hệ thống, custom entity sẽ hiển thị sau khi tạo
- Map đoạn `会員番号を読んでください` với `@verify-member-request`
- Map `ホットライン` với `@verify-member-hotline`
- Map mã số với `@sys.number-sequence`, ví dụ:
  - `12345` → `12345`
  - `3 4 5 6 7` → `34567`
  - `二〇七 六 五` → `20765`
- Map số điện thoại với `@sys.phone-number`

**Tạo Page và Flow:**

- Mặc định có flow tên `Default Welcome Flow`
- Tạo page mới tên `MemberCodeVerification` và kết nối với webhook

![Create Page](https://github.com/user-attachments/assets/eeb3253c-a521-4832-94de-3cf05fbcb259)

- Vào Flow → chọn `Start Page` → Route → chọn Intent đã tạo

![Start Page Routing](https://github.com/user-attachments/assets/72996e84-bcdd-4318-974d-836d67c46211)

- Scroll xuống để map Transition đến page `MemberCodeVerification`

![Map Transition](https://github.com/user-attachments/assets/161c413b-95db-4a1e-a673-5ab67e929e12)

- Vào page vừa tạo, mô tả rõ và cấu hình Entry Fulfillment

![Entry Fulfillment](https://github.com/user-attachments/assets/c93dee56-3eab-40aa-9baa-d4f756d4b332)

- Trong Fulfillment:
  - Bật webhook và chọn đúng webhook
  - Định nghĩa param như:
    - Member Code: `"$intent.params.number-sequence.resolved"`
    - Hotline: `"$intent.params.phone-number"`

![Webhook Fulfillment](https://github.com/user-attachments/assets/967edc0f-286a-42e4-8f19-610bda711823)

**Lưu ý:**
- Tag webhook bắt buộc
- Enable "Return partial response"
- Xóa default agent response để dùng phản hồi từ backend

**Test Flow:**

![Test 1](https://github.com/user-attachments/assets/cef5418e-7b2d-438c-8bb5-111cc84367cd)
![Test 2](https://github.com/user-attachments/assets/9e5cd25d-6eb4-4575-95a0-def5f423d134)

Endgame cho MemberCodeVerification
---
