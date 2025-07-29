
# Hướng dẫn Backend API: Xây dựng và Xử lý Webhook từ Dialogflow CX

Hướng dẫn này mô tả cách Backend API (PHP Laravel) sẽ tương tác với Dialogflow CX thông qua Webhook. Mục tiêu là đảm bảo tất cả các API đều nhận và trả về dữ liệu theo một cấu trúc thống nhất, giúp việc tích hợp và bảo trì trở nên dễ dàng.

---

## 1. Cấu trúc chung của API
Tất cả các API được gọi từ Dialogflow CX Webhook sẽ tuân theo cấu trúc chung sau:

- API method: `POST`
- Request Body: Luôn là JSON theo định dạng của Dialogflow CX Webhook Request.
- Response Body: Luôn là JSON theo định dạng chuẩn của Dialogflow CX Webhook Response.

---

## 2. Xây dựng Response chuẩn cho Dialogflow CX Webhook
Tất cả các API được gọi từ Dialogflow CX Webhook phải trả về dữ liệu theo định dạng chuẩn của Dialogflow CX Webhook Response. Việc này đảm bảo tính nhất quán và giảm thiểu lỗi.
Bạn nên tạo một hàm tiện ích (utility function) chung, để xử lý việc này.

Cấu trúc Response chuẩn:

```json
{
  "fulfillmentResponse": {
    "messages": [
      {
        "text": {
          "text": [
            "Đây là câu trả lời sẽ được đọc cho người dùng."
          ]
        }
      }
    ]
  },
  "sessionInfo": {
    "parameters": {
      "param1": "value1",
      "param2": "value2"
    }
  }
}
```
Trong đó:

`fulfillmentResponse.messages.text.text`: Đây là phần quan trọng nhất để trả lời người dùng. Giá trị trong mảng này ("text": ["Nội dung"]) chính là văn bản sẽ được Dialogflow CX chuyển đổi thành giọng nói (Text-to-Speech) và phát lại cho người dùng thông qua 3CX. Đảm bảo nội dung này rõ ràng, ngắn gọn và bằng ngôn ngữ tiếng Nhật.

`sessionInfo.parameters`: Phần này dùng để lưu trữ các giá trị cần được duy trì (persist) trong suốt một phiên (session) hội thoại của Dialogflow CX. Bất kỳ tham số nào bạn đặt vào đây sẽ được Dialogflow CX lưu trữ và có thể sử dụng lại ở các bước sau của cuộc hội thoại (ví dụ: trên các "page" khác, hoặc trong các intent tiếp theo).

Gợi ý

```php
// Ví dụ: app/Helpers/DialogflowWebhookHelper.php
<?php

namespace App\Helpers;

class DialogflowWebhookHelper
{
    /**
     * Builds a standardized webhook response for Dialogflow CX.
     *
     * @param string $replyText The text message to be spoken back to the user.
     * @param array $sessionParams Associative array of parameters to persist in the session.
     * @return array The formatted webhook response array.
     */
    public static function buildWebhookResponse(string $replyText, array $sessionParams = []): array
    {
        return [
            "fulfillmentResponse" => [
                "messages" => [
                    [
                        "text" => [
                            "text" => [$replyText]
                        ]
                    ]
                ]
            ],
            "sessionInfo" => [
                "parameters" => $sessionParams
            ]
        ];
    }
}
```

---

## 3. Hướng dẫn cụ thể cho API: `verify_member_code`

### 3.1. Trích xuất tham số từ Request
Request từ Dialogflow CX sẽ gửi đến Backend API dưới dạng JSON. Các tham số mà Dialogflow CX đã trích xuất từ câu nói của người dùng sẽ nằm trong cấu trúc `request.sessionInfo.parameters`.

Các tham số cho verify_member_code:

- member_code
- hotline

Ví dụ cấu trúc Request JSON từ Dialogflow CX:

```json
{
  "fulfillmentInfo": {
    "tag": "verify_member_code_webhook_tag"
  },
  "sessionInfo": {
    "session": "projects/your-project-id/locations/your-location/agents/your-agent-id/sessions/your-session-id",
    "parameters": {
      "member_code": "12345",
      "hotline": "08054321068"
    }
  },
  // ... các trường khác của Dialogflow CX Request
}
```

### 3.2. Chuẩn bị dữ liệu cho Response (trước khi gọi buildWebhookResponse)
Trước khi gọi hàm buildWebhookResponse, các API cần thực hiện logic nghiệp vụ và chuẩn bị hai loại dữ liệu chính:

$replyText (string): Chuỗi văn bản sẽ được nói cho người dùng. Đây là thông điệp phản hồi trực tiếp sau khi xử lý. Luôn đảm bảo replyText được định dạng rõ ràng, ngắn gọn và bằng ngôn ngữ tiếng Nhật.

$sessionParams (array): Một mảng kết hợp chứa các tham số cần được lưu trữ trong phiên làm việc của Dialogflow CX. Các tham số này có thể bao gồm:

- business_status - "success" hoặc "fail"
- member_code
- name
- club_id

---

## Sample

### 1. Case success

Request
```json
{
    "sessionInfo": {
      "parameters": {
        "member_id": "12345",
        "hotline": "0123455679"
      }
    },
    "session": "projects/your-project-id/locations/your-location/agents/your-agent-id/sessions/your-session-id-123",
    "intentInfo": {
      "lastMatchedIntent": "projects/your-project-id/locations/your-location/agents/your-agent-id/intents/VerifyMemberCodeIntent"
    }
}
```
Response
```json
{
  "fulfillmentResponse": {
    "messages": [
      {
        "text": {
          "text": [
            "会員コード 12345 の認証に成功しました。ようこそ、江戸川 コナンさん。次から選んでください：1. チケット予約、2. その他のリクエスト。"
          ]
        }
      }
    ]
  },
  "sessionInfo": {
    "parameters": {
      "status": true,
      "club_id": "ABCD",
      "member_code": "12345",
      "name": "江戸川 コナン"
    }
  }
}
```
### 2. Case fail
Request
```json
{
    "sessionInfo": {
      "parameters": {
        "member_id": "400",
        "hotline": "0123455679",
      }
    },
    "session": "projects/your-project-id/locations/your-location/agents/your-agent-id/sessions/your-session-id-123",
    "intentInfo": {
      "lastMatchedIntent": "projects/your-project-id/locations/your-location/agents/your-agent-id/intents/VerifyMemberCodeIntent"
    }
}
```
Response
```json
{
   "fulfillmentResponse":{
      "messages":[
         {
            "text":{
               "text":[
                  "会員コード 400 は無効です。確認して再度お試しください。"
               ]
            }
         }
      ]
   },
   "sessionInfo":{
      "parameters":{
         "member_code":"400",
         "club_id":"ABCD",
         "name":"江戸川 コナン",
         "status":false
      }
   }
}
```

---

## 4. AI matching function integration

Với event verification flow, dùng function này để thực hiện việc tìm event chính xacs nhất từ event list (query từ DB) so với event name được gửi từ prompt của 3CX

cURL mẫu

```bash
curl --location '{lien_he_de_hoi_url_chinh_xac_khong_nen_public}' \
--header 'Content-Type: application/json' \
--data '{
    "event_name_stt": "にほんおんがくさい",
    "list_event_name_db": [
        "日本音楽祭",
        "Japan Music Fair",
        "Tokyo Rock Festival"
    ]
}'
```

Case matched:

```json
{
    "matched_event_name": "日本音楽祭"
}
```

Case unmatched:

```json
{
    "matched_event_name": "No Match"
}
```