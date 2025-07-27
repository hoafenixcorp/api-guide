# Hướng dẫn setup Dialogflow CX cho dự án

## 1. Setup tài khoản, service account

### 1.1 Setup tài khoản

Đăng ký tài khoản GCP (Visa/Mastercard required)

Đi đến trang dashboard của GCP: [GCP Console](https://console.cloud.google.com/)

Tạo một project mới

<img width="1796" height="1084" alt="Screenshot 2025-07-27 at 16 21 21" src="https://github.com/user-attachments/assets/b117426b-9ff6-44c4-b118-e400e6ff2612" />

Enable các API cần thiết, bao gồm:

- Dialogflow API
- Cloud Run Admin API
- Cloud Functions API
- Cloud Build API
- Cloud Text-to-Speech API
- Cloud Speech-to-Text API

### 1.2 Tạo service account

#### Tạo service account & json key tương ứng để có thể sử dụng các API bên trên

- Click menu button ở góc trái dashboard và chọn Service Account tab

<img width="546" height="718" alt="Screenshot 2025-07-27 at 16 31 40" src="https://github.com/user-attachments/assets/77ea3999-ae11-4ee6-b9e5-4991333759f8" />

- Tạo một service account mới và assign những role bên dưới

<img width="665" height="680" alt="Screenshot 2025-07-27 at 16 34 47" src="https://github.com/user-attachments/assets/6d2ea3e4-60cf-4fbd-9d53-b0a40d70d5c5" />

- Tạo json file key -> key file để 3CX có thể connect và sử dụng TTS, STT. Vào key tab, click Add key và chọn JSON format

<img width="1796" height="775" alt="Screenshot 2025-07-27 at 16 38 16" src="https://github.com/user-attachments/assets/e477b278-33e5-439e-9f0e-85563c927c72" />

---

## 2. Setup conversational AI project

Truy cập vào [Conversational AI console](https://conversational-agents.cloud.google.com/projects) và taọ một project mới, refer đến project đã tạo từ GCP cloud

<img width="1797" height="1038" alt="Screenshot 2025-07-27 at 16 44 13" src="https://github.com/user-attachments/assets/585a9c06-f8f7-46d2-a7b4-6e4779b7e2c3" />

Tạo một agent mới, setup language code (ja) và region (asia)

* Note: có thể tạo 2 project khác nhau tương ứng cho Stag / Prod environment

<img width="3596" height="1176" alt="image" src="https://github.com/user-attachments/assets/65e73429-c6d4-4ac7-90f1-bcfc9f9758b4" />

### 2.1 MemberCodeVerification Flow setup

Prompt format: `［3CXの質問］: 会員番号を読んでください。［答え］: {text}。［ホットライン］: {hotline}`

Đầu tiên, chúng ta sẽ quan tâm đến: Entities, Webhook và Intent
<img width="1798" height="860" alt="Screenshot 2025-07-27 at 16 54 26" src="https://github.com/user-attachments/assets/e5d7413d-73c8-45e8-8412-e5fa5c0a6096" />


#### a. Entities

Dùng để config data type, giúp cho việc match intent dễ dàng hơn
Ví dụ:
`verify-member-request`: dùng để map với `［3CXの質問］: 会員番号を読んでください。`

<img width="909" height="947" alt="Screenshot 2025-07-27 at 16 55 36" src="https://github.com/user-attachments/assets/a5882266-9cc4-499b-9147-17cfb178a245" />

tương tự cho `verify-member-hotline`: map với `［ホットライン］`

#### b. Webhooks

Dùng để setup URL của verify_member_code api của Backend server (PHP Laravel)

Click tạo một webhook mới, set URL của api phù hợp

<img width="918" height="982" alt="Screenshot 2025-07-27 at 17 01 18" src="https://github.com/user-attachments/assets/ec3d35b0-fd6c-4074-8afc-25b06bdfa2c2" />

#### c. Intents
Đây là nơi để tạo và train data để có thể hiểu và bóc tách parameter từ prompt của User

Tạo một intent mới và đặt tên phù hợp, ví dụ: member.code.verification

<img width="912" height="966" alt="Screenshot 2025-07-27 at 17 04 47" src="https://github.com/user-attachments/assets/f5ffa44c-0d85-41a5-bc86-89920538e9c5" />

Insert training phase để huấn luyện ý định verify member code, có thể là dãy số, dãy chữ, có khoảng trắng hoặc không
(có thể tạo trong file CSV và upload lên)

Quan trọng: Bôi đen text và map nó với entity tương ứng để có thể lấy value

<img width="897" height="581" alt="Screenshot 2025-07-27 at 17 07 33" src="https://github.com/user-attachments/assets/c2ca7468-e0e1-465b-bba7-b5dccc13d42b" />

Giải thích:
- Entity có tiền tố @sys là entity sẵn có, và ở đây sẽ thấy những entity custom đã tạo ở trên
- parameter_id có thể đổi tên cho phù hợp với setup
- Bôi đen `［3CXの質問］: 会員番号を読んでください。` và map nó với `@verify-member-request`
- Bôi đen `［ホットライン］` và map nó với `@verify-member-hotline`
- Bôi đen những chuỗi nào là member_code và map nó với @sys.number-sequence -> NLU sẽ tự hiểu là chúng ta cần một dãy số và tự parse ra đúng kết quả
  + 12345 --> 12345
  + 3 4 5 6 7 --> 34567
  + 二〇七 六 五 --> 20765
- Bôi đen số điện thoại và map với `@sys.phone-number`

* Note: Càng nhiều training phase càng tốt, càng đa dạng input type càng tốt


Từ những setup trên, chúng ta sẽ setup flow đầu tiên cho MemberCodeVerification

Mặc định, sẽ có sẵn một flow tên là Default Welcome Flow, có thể giữ nguyên hoặc đổi tên để sử dụng luôn

Tạo một Page mới, ví dụ MemberCodeVerification Page, đây là nơi sẽ gọi webhook URL để xử lý logic và get Data

<img width="335" height="699" alt="Screenshot 2025-07-27 at 17 22 32" src="https://github.com/user-attachments/assets/eeb3253c-a521-4832-94de-3cf05fbcb259" />

Click vào Flow, chúng ta sẽ thấy một diagram dùng để design Flow, click vào `Start Page`

Ở đây, chọn intent đã tạo bên trên vào phần Route

<img width="1229" height="938" alt="Screenshot 2025-07-27 at 17 20 11" src="https://github.com/user-attachments/assets/72996e84-bcdd-4318-974d-836d67c46211" />

Kéo xuống dưới cùng và map Transition vào Page tạo bên trên

<img width="1241" height="1019" alt="Screenshot 2025-07-27 at 17 21 19" src="https://github.com/user-attachments/assets/161c413b-95db-4a1e-a673-5ab67e929e12" />

Click vào Page, ở đây chúng ta thêm desc cho tường minh, và chú ý đến Entry Fulfillment

<img width="607" height="845" alt="Screenshot 2025-07-27 at 17 26 36" src="https://github.com/user-attachments/assets/c93dee56-3eab-40aa-9baa-d4f756d4b332" />

Ở phần edit fulfillment, chú ý đến Parameter presets: chúng ta sẽ define request param ở đây
Expand Webhook settings và Enable Webhook, sau đó chọn Webhook tương ứng đã tạo bên trên

<img width="914" height="973" alt="Screenshot 2025-07-27 at 17 27 46" src="https://github.com/user-attachments/assets/967edc0f-286a-42e4-8f19-610bda711823" />

Giải thích:
- Giá trị của member code sẽ có dạng: `"$intent.params.number-sequence.resolved"` vì chúng ta map nó với number-sequence (nếu không đổi tên), và resolved để lấy giá trị sau khi parse
- Giá trị của hotline sẽ là: `"$intent.params.phone-number"` (tương tự)
- Tag ở webhook settings là bắt buộc
- Enable `Return partial response`
- Xóa Agent response mặc định để có thể lấy message từ webhook response

Vậy là xong phần setup MemberCodeVerification Flow, có thể test trực tiếp từ Preview UI bên phải màn hình
<img width="1788" height="1083" alt="Screenshot 2025-07-27 at 17 33 41" src="https://github.com/user-attachments/assets/cef5418e-7b2d-438c-8bb5-111cc84367cd" />
<img width="548" height="968" alt="Screenshot 2025-07-27 at 17 35 53" src="https://github.com/user-attachments/assets/9e5cd25d-6eb4-4575-95a0-def5f423d134" />


