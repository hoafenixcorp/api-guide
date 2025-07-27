# Setup Proxy API

Proxy API sẽ làm cầu nối giữa 3CX và Dialogflow CX.
Nó sẽ được viết bằng Python và deploy bằng Cloud Functions
Nhiệm vụ:

- Nhận prompt từ 3CX
- Nếu là session mới, check xem có session id từ request không, nếu không thì tự tạo mới (uuid)
- Nhận response từ Dialogflow (Webhook response), check `business_status` để có api status code phù hợp

---

## Code implementation

Lưu ý: quay lại conversation ai console page, ta sẽ có url có format như sau

`https://conversational-agents.cloud.google.com/projects/{project_id}/locations/{location}/agents/{agent_id}/`

thay những giá trị tương ứng cho các biến bên dưới

```python
import functions_framework
import json
from google.cloud import dialogflowcx_v3beta1 as dialogflowcx
import uuid
import os

PROJECT_ID = os.environ.get("GCP_PROJECT_ID", "project_id")
CX_AGENT_LOCATION = os.environ.get("CX_AGENT_LOCATION", "location")
DIALOGFLOW_AGENT_ID = os.environ.get("CX_AGENT_ID", "agent_id")

if not all([PROJECT_ID, CX_AGENT_LOCATION, DIALOGFLOW_AGENT_ID]):
    raise ValueError("Missing one or more required environment variables: GCP_PROJECT_ID, CX_AGENT_LOCATION, CX_AGENT_ID")

client_options = {"api_endpoint": f"{CX_AGENT_LOCATION}-dialogflow.googleapis.com"}
dialogflow_cx_client = dialogflowcx.SessionsClient(client_options=client_options)


@functions_framework.http
def handle_3cx_proxy_request(request):
    response_status_code = 200  # Default HTTP status code

    try:
        request_json = request.get_json(silent=True)

        if not request_json:
            print("Error: Invalid JSON request.")
            response_status_code = 400
            return {
                "status": "error",
                "bot_reply_text": "エラー：無効なリクエストです。",
                "dialogflow_intent": "N/A"
            }, response_status_code

        full_prompt = request_json.get('prompt', '').strip()
        session_id = request_json.get('session_id')
        received_language_code = request_json.get('language_code', 'ja')

        if not full_prompt:
            print("Error: 'prompt' missing in the request.")
            response_status_code = 400
            return {
                "status": "error",
                "bot_reply_text": "エラー：「プロンプト」がリクエストに含まれていません。",
                "dialogflow_intent": "N/A"
            }, response_status_code

        if not session_id:
            session_id = str(uuid.uuid4())
            print(f"Generated new session_id: {session_id}")

        session_path = f"projects/{PROJECT_ID}/locations/{CX_AGENT_LOCATION}/agents/{DIALOGFLOW_AGENT_ID}/sessions/{session_id}"

        text_input = dialogflowcx.TextInput(text=full_prompt)
        query_input = dialogflowcx.QueryInput(text=text_input, language_code=received_language_code)

        print(
            f"Sending to Dialogflow CX: Session={session_id}, Prompt='{full_prompt}', Lang='{received_language_code}'")

        dfcx_response = dialogflow_cx_client.detect_intent(
            request={
                "session": session_path,
                "query_input": query_input
            }
        )

        query_result = dfcx_response.query_result

        intent_display_name = "No Intent Matched"
        if query_result.match.intent:
            intent_display_name = query_result.match.intent.display_name

        bot_reply_text = ""
        if query_result.response_messages:
            for message in query_result.response_messages:
                if message.text and message.text.text:
                    bot_reply_text = message.text.text[0]
                    break

        business_status = "fail"
        cx_session_parameters = query_result.parameters if query_result.parameters else {}

        if 'business_status' in cx_session_parameters:
            business_status = str(cx_session_parameters['business_status'])

        # Build final response
        response_for_client = {
            "status": business_status,
            "bot_reply_text": bot_reply_text,
            "session_id": session_id,
            "dialogflow_intent": intent_display_name
        }

        response_status_code = 200
        if business_status == 'fail':
            response_status_code = 400

        return response_for_client, response_status_code

    except Exception as e:
        print(f"An unexpected error occurred: {e}")
        import traceback
        traceback.print_exc()

        response_status_code = 500
        return {
            "status": "error",
            "bot_reply_text": "システム内部エラーが発生しました。しばらくしてから再度お試しください。",
            "error_details": str(e),
            "session_id": session_id if 'session_id' in locals() else "unknown",
            "dialogflow_intent": "Error"
        }, response_status_code
```

File requirements.txt
```text
functions-framework
google-cloud-dialogflow-cx
```

---

## Deploy

Search Cloud Run và truy cập dashboard

<img width="3584" height="1144" alt="image" src="https://github.com/user-attachments/assets/6d2a23ff-102a-4826-8b93-fe21b5762363" />

Chọn Write a function và setup thông số như sau

<img width="879" height="940" alt="Screenshot 2025-07-27 at 18 08 40" src="https://github.com/user-attachments/assets/37d873f9-3a0b-4be7-82b3-98052996e067" />

Copy code vào, save & deploy

<img width="1545" height="659" alt="Screenshot 2025-07-27 at 18 09 49" src="https://github.com/user-attachments/assets/6a49a107-39ed-4b46-873b-5d4951cc5200" />

* Note: function name và Function entry point phải trùng nhau

Copy Function URL và setup để thực hiện API call từ 3CX

ví dụ cURL

```bash
curl --location '{proxy_api_function_URL}' \
--header 'Content-Type: application/json' \
--data '{
    "prompt":"［3CXの質問］: 会員番号を読んでください。［答え］: 会員です、番号は 四〇六七〇 です。［ホットライン]：08054321068"
}'
```
