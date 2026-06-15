# Hướng Dẫn Từng Bước (Step-by-Step) Hoàn Thành Day 13 Observability Lab

Tài liệu này hướng dẫn bạn chi tiết từng bước để tự hoàn thành bài Lab về **Monitoring, Logging, and Observability**. Sau khi làm theo hướng dẫn này, hệ thống của bạn sẽ đạt điểm tối đa **100/100** khi chạy script kiểm tra và tích hợp đầy đủ hệ thống giám sát, đồng thời hỗ trợ bạn lấy trọn vẹn điểm thưởng (Bonus +10đ).

---

## Mục Lục
1. [Bước 1: Thiết lập môi trường ảo và chạy ứng dụng](#bước-1-thiết-lập-môi-trường-ảo-và-chạy-ứng-dụng)
2. [Bước 2: Cấu hình Correlation ID (`app/middleware.py`)](#bước-2-cấu-hình-correlation-id-appmiddlewarepy)
3. [Bước 3: Làm phong phú (Enrich) Logs với Ngữ cảnh request (`app/main.py`)](#bước-3-làm-phong-phú-enrich-logs-với-ngữ-cảnh-request-appmainpy)
4. [Bước 4: Thiết lập lọc dữ liệu nhạy cảm PII (`app/pii.py` & `app/logging_config.py`)](#bước-4-thiết-lập-lọc-dữ-liệu-nhạy-cảm-pii-apppiipy--applogging_configpy)
5. [Bước 5: Chạy Script kiểm tra kết quả Logs](#bước-5-chạy-script-kiểm-tra-kết-quả-logs)
6. [Bước 6: Tích hợp Langfuse Tracing](#bước-6-tích-hợp-langfuse-tracing)
7. [Bước 7: Thiết kế Dashboard giám sát (6 Panels)](#bước-7-thiết-kế-dashboard-giám-sát-6-panels)
8. [Bước 8: Kiểm thử Cảnh báo (Alerts) & Xử lý sự cố (Incident Response)](#bước-8-kiểm-thử-cảnh-báo-alerts--xử-lý-sự-cố-incident-response)
9. [Bước 9: Thực hiện các phần điểm thưởng (Bonus Items +10đ)](#bước-9-thực-hiện-các-phần-điểm-thưởng-bonus-items-10đ)

---

## Bước 1: Thiết lập môi trường ảo và chạy ứng dụng

1. **Khởi tạo và kích hoạt môi trường ảo Python**:
   - Mở terminal trong thư mục dự án và chạy:
     ```bash
     python -m venv .venv
     ```
   - Kích hoạt môi trường ảo:
     - **Windows (PowerShell)**: `.venv\Scripts\Activate.ps1`
     - **Windows (CMD)**: `.venv\Scripts\activate.bat`
     - **macOS/Linux**: `source .venv/bin/activate`

2. **Cài đặt các thư viện phụ thuộc**:
   ```bash
   pip install -r requirements.txt
   ```

3. **Tạo file biến môi trường**:
   - Sao chép file cấu hình mẫu `.env.example` thành `.env`:
     - Trên Windows: `copy .env.example .env`
     - Trên macOS/Linux: `cp .env.example .env`

4. **Chạy server phát triển**:
   ```bash
   uvicorn app.main:app --reload
   ```
   Ứng dụng FastAPI sẽ chạy tại địa chỉ `http://127.0.0.1:8000`. Bạn có thể truy cập tài liệu API tự động tại `http://127.0.0.1:8000/docs`.

---

## Bước 2: Cấu hình Correlation ID (`app/middleware.py`)

Correlation ID (Mã định danh tương quan) giúp liên kết tất cả các log được sinh ra trong cùng một vòng đời của request.

Mở file `app/middleware.py` và thực hiện các cập nhật sau:
1. Giải phóng biến ngữ cảnh (contextvars) trước mỗi request bằng cách gọi `clear_contextvars()`.
2. Kiểm tra xem header `x-request-id` có tồn tại trong request hay không. Nếu không, hãy tạo mới một ID ngẫu nhiên theo định dạng `req-<8-char-hex>`.
3. Gắn `correlation_id` này vào contextvars của `structlog` bằng `bind_contextvars`.
4. Lưu ID vào `request.state.correlation_id` để router có thể lấy ra sử dụng.
5. Thêm `x-request-id` và thời gian xử lý request `x-response-time-ms` (làm tròn đến 2 chữ số thập phân) vào response headers.

**Mã nguồn hoàn chỉnh gợi ý cho `app/middleware.py`:**
```python
from __future__ import annotations

import time
import uuid

from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
from structlog.contextvars import bind_contextvars, clear_contextvars


class CorrelationIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # 1. Clear contextvars để tránh rò rỉ dữ liệu giữa các request khác nhau
        clear_contextvars()

        # 2. Lấy x-request-id từ headers hoặc tự động sinh một mã ngẫu nhiên 8 ký tự hex
        correlation_id = request.headers.get("x-request-id")
        if not correlation_id:
            correlation_id = f"req-{uuid.uuid4().hex[:8]}"
        
        # 3. Gắn correlation_id vào structlog contextvars
        bind_contextvars(correlation_id=correlation_id)
        
        # 4. Lưu lại vào request state
        request.state.correlation_id = correlation_id
        
        start = time.perf_counter()
        response = await call_next(request)
        
        # 5. Tính toán latency và thêm correlation_id cùng với processing time vào response headers
        duration_ms = (time.perf_counter() - start) * 1000
        response.headers["x-request-id"] = correlation_id
        response.headers["x-response-time-ms"] = f"{duration_ms:.2f}"
        
        return response
```

---

## Bước 3: Làm phong phú (Enrich) Logs với Ngữ cảnh request (`app/main.py`)

Để phục vụ phân tích log nâng cao, chúng ta cần đính kèm các trường thông tin ngữ cảnh nghiệp vụ vào log.

Mở file `app/main.py` và tìm đến API `/chat`. Đăng ký các biến ngữ cảnh bổ sung bằng hàm `bind_contextvars` ngay đầu route xử lý.

**Đoạn code cần sửa đổi tại `app/main.py`:**
```python
@app.post("/chat", response_model=ChatResponse)
async def chat(request: Request, body: ChatRequest) -> ChatResponse:
    # Enrich logs với thông tin ngữ cảnh chi tiết của request
    bind_contextvars(
        user_id_hash=hash_user_id(body.user_id),
        session_id=body.session_id,
        feature=body.feature,
        model=agent.model,
        env=os.getenv("APP_ENV", "dev"),
    )
    
    log.info(
        "request_received",
        service="api",
        payload={"message_preview": summarize_text(body.message)},
    )
...
```

---

## Bước 4: Thiết lập lọc dữ liệu nhạy cảm PII (`app/pii.py` & `app/logging_config.py`)

Bài lab yêu cầu lọc các thông tin nhạy cảm (PII) như: Email, số điện thoại, số thẻ tín dụng, CCCD, Hộ chiếu (Passport) và Địa chỉ tiếng Việt khỏi các bản ghi log.

### 4.1 Bổ sung Regex Patterns trong `app/pii.py`
Mở `app/pii.py` và thêm định nghĩa cho Passport và các từ khóa địa chỉ tiếng Việt:
```python
PII_PATTERNS: dict[str, str] = {
    "email": r"[\w\.-]+@[\w\.-]+\.\w+",
    "phone_vn": r"(?:\+84|0)[ \.-]?\d{3}[ \.-]?\d{3}[ \.-]?\d{3,4}", # Khớp 090 123 4567, 090.123.4567...
    "cccd": r"\b\d{12}\b",
    "credit_card": r"\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b",
    # Thêm các patterns cho Passport và Địa chỉ Việt Nam
    "passport": r"\b[A-Z]\d{7}\b",
    "address_vn": r"(?i)\b(?:số\s+\d+|đường|phố|quận|huyện|tỉnh|thành\s+phố|phường|xã|ấp|thôn)\b",
}
```

### 4.2 Triển khai đệ quy làm sạch PII trong `app/logging_config.py`
Mở `app/logging_config.py` và cập nhật hàm `scrub_event` để lọc PII đệ quy trên tất cả các giá trị kiểu chuỗi (string) trong log dictionary. Đồng thời, đăng ký `scrub_event` vào chuỗi xử lý logs.

*Lưu ý*: Script kiểm tra `validate_logs.py` kiểm tra toàn bộ chuỗi JSON thô (`raw = json.dumps(rec)`). Do đó, việc duyệt đệ quy qua toàn bộ dict log đảm bảo không có trường con nào (kể cả trong payload hoặc message) bị rò rỉ.

**Cập nhật `app/logging_config.py`:**
```python
def scrub_event(_: Any, __: str, event_dict: dict[str, Any]) -> dict[str, Any]:
    # Hàm đệ quy duyệt qua tất cả cấu trúc dữ liệu lồng nhau và lọc bỏ PII
    def _scrub_recursive(val: Any) -> Any:
        if isinstance(val, str):
            return scrub_text(val)
        elif isinstance(val, dict):
            return {k: _scrub_recursive(v) for k, v in val.items()}
        elif isinstance(val, list):
            return [_scrub_recursive(x) for x in val]
        return val

    return _scrub_recursive(event_dict)


def configure_logging() -> None:
    logging.basicConfig(format="%(message)s", level=getattr(logging, os.getenv("LOG_LEVEL", "INFO")))
    structlog.configure(
        processors=[
            merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso", utc=True, key="ts"),
            # Kích hoạt bộ lọc PII bằng cách uncomment dòng dưới:
            scrub_event,
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            JsonlFileProcessor(),
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        cache_logger_on_first_use=True,
    )
```

---

## Bước 5: Chạy Script kiểm tra kết quả Logs

Sau khi hoàn tất cài đặt Middleware, Log Enrichment và PII Scrubber:
1. Đảm bảo ứng dụng uvicorn đang chạy (`uvicorn app.main:app --reload`).
2. Gửi một vài request thử nghiệm đến server bằng cách chạy script load test:
   ```bash
   `python scripts/load_test.py`
   ```
3. Chạy script chấm điểm tự động:
   ```bash
   python scripts/validate_logs.py
   ```
   *Yêu cầu kết quả đạt được*: **Estimated Score: 100/100** và không phát hiện bất kỳ PII leaks nào.

---

## Bước 6: Tích hợp Langfuse Tracing

Langfuse giúp theo dõi trực quan luồng xử lý ứng dụng LLM dưới dạng Trace và Spans.

1. **Đăng ký / Đăng nhập Langfuse**:
   - Bạn có thể sử dụng [Langfuse Cloud](https://cloud.langfuse.com/) (miễn phí) hoặc tự host local bằng Docker.
2. **Tạo Dự án (Project) mới** trên giao diện Langfuse để nhận thông tin xác thực:
   - `LANGFUSE_PUBLIC_KEY`
   - `LANGFUSE_SECRET_KEY`
   - `LANGFUSE_HOST` (mặc định cho cloud là `https://cloud.langfuse.com`)
3. **Cập nhật cấu hình môi trường**:
   - Mở file `.env` và điền đầy đủ các thông tin:
     ```env
     LANGFUSE_PUBLIC_KEY=pk-lf-...
     LANGFUSE_SECRET_KEY=sk-lf-...
     LANGFUSE_HOST=https://cloud.langfuse.com
     ```
4. **Chạy ứng dụng để gửi Traces**:
   - Khởi động lại uvicorn server để nhận biến môi trường mới.
   - Chạy lệnh test để gửi tối thiểu 10-20 request:
     ```bash
     python scripts/load_test.py --concurrency 5
     ```
   - Đăng nhập vào bảng điều khiển Langfuse của bạn và xác thực rằng các traces đã hiển thị thành công. Hãy chụp ảnh màn hình danh sách Trace này và ảnh chi tiết của một Trace Waterfall để đưa vào báo cáo `docs/blueprint-template.md`.

---

## Bước 7: Sử dụng Dashboard giám sát thời gian thực (`dashboard.html`)

Để giúp bạn đạt điểm tối đa và nhận điểm thưởng **"Dashboard đẹp, chuyên nghiệp vượt mong đợi (Bonus +3đ)"**, tôi đã xây dựng sẵn một trang Web Dashboard hoàn chỉnh bằng HTML/CSS/JS sử dụng thư viện **Chart.js** với phong cách **Glassmorphism (Kính mờ) + Dark Mode** siêu đẹp.

Dashboard này kết nối trực tiếp đến API `/metrics` của ứng dụng và tự động vẽ các đồ thị chuyển động thời gian thực mỗi 10 giây.

### Cách mở và sử dụng Dashboard:
1. **Khởi chạy ứng dụng**: Đảm bảo ứng dụng FastAPI đang chạy tại cổng `8000` (`uv run uvicorn app.main:app --reload`).
2. **Mở tệp Dashboard**:
   - Bạn hãy mở tệp tin [dashboard.html](file:///d:/CODE/AITHUCCHIEN/LABS/2A202600699-HoTatBaoHoang-Day13/dashboard.html) trực tiếp bằng trình duyệt Web của bạn (nhấp đúp chuột vào file hoặc mở bằng trình duyệt).
3. **Theo dõi dữ liệu chuyển động**:
   - Nhìn vào góc trên bên phải, nếu trạng thái hiển thị **`Connected`** (chấm màu xanh lá), tức là Dashboard đã kết nối thành công với FastAPI backend.
   - Khi bạn chạy load test (`uv run python scripts/load_test.py --concurrency 5`), bạn sẽ thấy các biểu đồ trên Dashboard chuyển động vẽ các điểm dữ liệu mới thời gian thực!

### 6 Panels đạt tiêu chuẩn chất lượng (Quality Bar) gồm:
1. **Latency Trends**: Đồ thị P50, P95, P99 vẽ chung để so sánh, kèm đường đứt nét SLO cảnh báo màu đỏ ở mốc `3000ms`.
2. **Traffic & Load**: Đồ thị biểu diễn tổng lượng request tích lũy theo thời gian.
3. **Error Rate & Breakdown**: Biểu đồ hình tròn và tỷ lệ lỗi kèm danh sách liệt kê phân rã cụ thể các lỗi phát sinh (ví dụ: `RuntimeError`). Có đường đứt nét SLO ở mốc `2%`.
4. **LLM Budget & Cost**: Đồ thị biểu diễn chi phí cuộc gọi LLM tăng tiến, kèm đường ngân sách tối đa `$2.5` trong ngày.
5. **Token Utilization**: Biểu đồ so sánh cột (stacked bar) số lượng Tokens đầu vào (Input) và đầu ra (Output).
6. **Answer Quality Heuristics**: Đồ thị chất lượng trung bình câu trả lời theo thuật toán heuristc, kèm đường SLO ở mốc `0.75`.

---

## Bước 8: Kiểm thử Cảnh báo (Alerts) & Xử lý sự cố (Incident Response)

Hệ thống cung cấp sẵn các kịch bản tiêm lỗi (incident injection) để kiểm tra độ tin cậy và khả năng cảnh báo của bạn.

### Kịch bản 1: RAG Slow (Mô phỏng cơ sở dữ liệu vector bị chậm)
- **Kích hoạt sự cố**:
  ```bash
  python scripts/inject_incident.py --scenario rag_slow
  ```
- **Hành vi**: Thời gian lấy tài liệu RAG sẽ bị trễ thêm 2.5 giây cho mỗi request.
- **Phát hiện**:
  - Gửi load test để tạo lưu lượng: `python scripts/load_test.py --concurrency 3`.
  - Trên Langfuse, kiểm tra trace waterfall để chứng minh khoảng thời gian trễ nằm ở span `retrieve` (RAG) thay vì span `generate` (LLM).
  - Kiểm tra xem metric `latency_p95` từ `/metrics` có vượt ngưỡng SLO `3000ms` hay không.
- **Tắt sự cố**:
  ```bash
  python scripts/inject_incident.py --scenario rag_slow --disable
  ```

### Kịch bản 2: Tool Fail (Lỗi hệ thống vector database)
- **Kích hoạt sự cố**:
  ```bash
  python scripts/inject_incident.py --scenario tool_fail
  ```
- **Hành vi**: Lệnh `retrieve(message)` sẽ ném ra lỗi `RuntimeError("Vector store timeout")`.
- **Phát hiện**:
  - Gửi request đến chat. Hệ thống sẽ trả về mã lỗi HTTP 500.
  - Kiểm tra log file `data/logs.jsonl` hoặc kiểm tra `/metrics` sẽ thấy `error_breakdown` ghi nhận lỗi `RuntimeError`.
- **Tắt sự cố**:
  ```bash
  python scripts/inject_incident.py --scenario tool_fail --disable
  ```

### Kịch bản 3: Cost Spike (LLM sinh phản hồi quá dài gây lãng phí chi phí)
- **Kích hoạt sự cố**:
  ```bash
  python scripts/inject_incident.py --scenario cost_spike
  ```
- **Hành vi**: Token đầu ra của LLM tăng gấp 4 lần so với thông thường.
- **Phát hiện**:
  - Gửi load test và xem metric `total_cost_usd` tăng nhanh trên dashboard hoặc `/metrics`.
- **Tắt sự cố**:
  ```bash
  python scripts/inject_incident.py --scenario cost_spike --disable
  ```

---

## Bước 9: Thực hiện các phần điểm thưởng (Bonus Items +10đ)

Để nhận được điểm số tối đa vượt mong đợi của giảng viên, hãy tích hợp thêm các tính năng nâng cao sau:

### 9.1 Tách riêng Audit Logs (Bonus +2đ)
Hệ thống yêu cầu ghi nhận riêng các hành động mang tính quản trị (chẳng hạn như bật/tắt incident) vào một file riêng `data/audit.jsonl`.

1. Định nghĩa hàm ghi audit log riêng trong `app/logging_config.py`:
   ```python
   AUDIT_PATH = Path("data/audit.jsonl")

   def log_audit(event: str, action: str, details: dict) -> None:
       import json
       from datetime import datetime, timezone
       AUDIT_PATH.parent.mkdir(parents=True, exist_ok=True)
       record = {
           "ts": datetime.now(timezone.utc).isoformat(),
           "event": event,
           "action": action,
           "details": details
       }
       with AUDIT_PATH.open("a", encoding="utf-8") as f:
           f.write(json.dumps(record) + "\n")
   ```

2. Gọi hàm này trong các API bật/tắt incident tại `app/main.py`:
   - Import hàm `log_audit` từ `.logging_config`.
   - Cập nhật hàm `enable_incident` và `disable_incident`:
     ```python
     @app.post("/incidents/{name}/enable")
     async def enable_incident(name: str) -> JSONResponse:
         try:
             enable(name)
             log.warning("incident_enabled", service="control", payload={"name": name})
             # Ghi vào audit log
             log_audit("incident_change", "enable", {"incident_name": name})
             return JSONResponse({"ok": True, "incidents": status()})
         except KeyError as exc:
             raise HTTPException(status_code=404, detail=str(exc)) from exc

     @app.post("/incidents/{name}/disable")
     async def disable_incident(name: str) -> JSONResponse:
         try:
             disable(name)
             log.warning("incident_disabled", service="control", payload={"name": name})
             # Ghi vào audit log
             log_audit("incident_change", "disable", {"incident_name": name})
             return JSONResponse({"ok": True, "incidents": status()})
         except KeyError as exc:
             raise HTTPException(status_code=404, detail=str(exc)) from exc
     ```

### 9.2 Tối ưu hóa chi phí (Cost Optimization) (Bonus +3đ)
Giảng viên muốn thấy một cơ chế tối ưu chi phí LLM (ví dụ: chuyển sang model rẻ hơn nếu câu hỏi quá ngắn hoặc đơn giản).

1. Thay đổi logic trong `app/agent.py` để tự động chọn model rẻ hơn (ví dụ: `claude-haiku` thay vì `claude-sonnet-4-5`) nếu độ dài message dưới 15 ký tự:
   ```python
   # Trong file app/agent.py
   class LabAgent:
       def __init__(self, model: str = "claude-sonnet-4-5") -> None:
           self.model = model
           self.llm = FakeLLM(model=model)

       @observe()
       def run(self, user_id: str, feature: str, session_id: str, message: str) -> AgentResult:
           # Quyết định model dựa trên độ phức tạp của câu hỏi
           selected_model = self.model
           if len(message) < 15:
               selected_model = "claude-haiku"
           
           # Khởi tạo FakeLLM động hoặc đổi model
           llm_runner = FakeLLM(model=selected_model)
           
           started = time.perf_counter()
           docs = retrieve(message)
           prompt = f"Feature={feature}\nDocs={docs}\nQuestion={message}"
           
           # Sinh phản hồi từ model đã chọn
           response = llm_runner.generate(prompt)
           quality_score = self._heuristic_quality(message, response.text, docs)
           latency_ms = int((time.perf_counter() - started) * 1000)
           cost_usd = self._estimate_cost(response.usage.input_tokens, response.usage.output_tokens, selected_model)

           # Cập nhật thông tin trace với model thực tế đã sử dụng
           langfuse_context.update_current_trace(
               user_id=hash_user_id(user_id),
               session_id=session_id,
               tags=["lab", feature, selected_model],
           )
           ...
   ```
2. Điều chỉnh hàm `_estimate_cost` để hỗ trợ giá tiền của các models khác nhau:
   ```python
       def _estimate_cost(self, tokens_in: int, tokens_out: int, model_name: str) -> float:
           if model_name == "claude-haiku":
               # Giá rẻ hơn 10 lần
               input_cost = (tokens_in / 1_000_000) * 0.25
               output_cost = (tokens_out / 1_000_000) * 1.25
           else:
               # Giá của claude-sonnet-4-5
               input_cost = (tokens_in / 1_000_000) * 3
               output_cost = (tokens_out / 1_000_000) * 15
           return round(input_cost + output_cost, 6)
   ```

### 9.3 Tự động hóa kiểm thử (Bonus +2đ)
Viết một script kiểm tra tự động trước khi triển khai (pre-deployment check) để đảm bảo không xảy ra lỗi cú pháp và log format luôn đúng:
Tạo một file script `scripts/pre_deploy_test.py`:
```python
import subprocess
import sys

def run_tests():
    print("Executing Unit Tests...")
    test_result = subprocess.run(["pytest", "tests/"], capture_output=True, text=True)
    if test_result.returncode != 0:
        print("Unit tests failed!")
        print(test_result.stdout)
        print(test_result.stderr)
        sys.exit(1)
    print("Unit tests passed successfully.")

if __name__ == "__main__":
    run_tests()
```
Đăng ký script này chạy định kỳ hoặc chạy thủ công trước khi nộp bài để tăng điểm tự động hóa.
