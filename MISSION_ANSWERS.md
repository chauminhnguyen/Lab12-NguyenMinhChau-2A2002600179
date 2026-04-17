# Day 12 Lab - Mission Answers
> **Student Name:** Nguyễn Minh Châu
> 
> **Student ID:** 2A202600179
> 
> **Date:** 2026-04-17

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. Lộ Hardcoded secrets: Các biến OPENAI_API_KEY và DATABASE_URL (chứa cả credential của PostgreSQL) được gắn cứng ngay trong source code -> rủi ro bảo mật nghiêm trọng; nếu code được push lên GitHub, key sẽ bị quét và bị lạm dụng ngay lập tức. Các thông tin này bắt buộc phải được nạp qua biến môi trường (Environment variables) hoặc các secret manager.
2. Thiếu hệ thống quản lý cấu hình: Các tham số hệ thống như DEBUG = True hay MAX_TOKENS = 500 đang bị hardcode. Khi vận hành hệ thống AI thực tế qua nhiều môi trường (Dev, Staging, Prod), thiết kế này triệt tiêu khả năng thay đổi hành vi của agent nếu không sửa trực tiếp source code. Giải pháp chuẩn là sử dụng các thư viện như pydantic-settings kết hợp file .env.
3. Sử dụng print sai mục đích và rò rỉ secret qua Log: Việc dùng print thay vì thư viện logging chuẩn (như loguru hoặc module logging của Python) khiến hệ thống không thể phân loại log (INFO, WARN, ERROR) khi monitor. Hơn nữa, dòng code print(f"[DEBUG] Using key: {OPENAI_API_KEY}") trực tiếp đẩy secret ra standard output, khiến key bị lưu lại vĩnh viễn trên các hệ thống thu thập log.
4. Không có health check endpoint: Ứng dụng hoàn toàn thiếu một route định tuyến trạng thái (ví dụ /health hoặc /ping). Khi agent được đóng gói chạy dưới dạng container (như Docker) hoặc quản lý bởi các cloud platform, hệ thống điều phối bắt buộc phải gọi endpoint này để kiểm tra xem tiến trình có đang phản hồi không. Nếu không có, hệ thống sẽ không biết để tự động restart khi agent bị crash hoặc treo.
5. Gắn cứng cấu hình khởi chạy mạng (Host, Port, Reload): * host="localhost": Khiến ứng dụng chỉ nhận request nội bộ, không thể giao tiếp ra ngoài (đặc biệt lỗi khi chạy trong container, host bắt buộc phải là 0.0.0.0).

port=8000: Gây xung đột và lỗi deploy trên các nền tảng cloud (như Railway) vì các hệ thống này yêu cầu ứng dụng listen trên một port động được cấp phát qua biến môi trường PORT.

reload=True: Đây là tính năng chỉ dành cho môi trường phát triển (Local/Dev). Nếu để lọt ra production, nó sẽ theo dõi file system liên tục, gây hao tổn tài nguyên nghiêm trọng và rò rỉ bộ nhớ.
...

### Exercise 1.3: Comparison table
### 2. So sánh Basic vs Advanced (12-Factor Compliant Agent)

| Feature | Basic | Advanced | Tại sao quan trọng? |
| :--- | :--- | :--- | :--- |
| **Config** | Hardcode | Env vars (`config.settings`) | Đảm bảo bảo mật thông tin nhạy cảm. Cho phép thay đổi hành vi hệ thống linh hoạt trên đa môi trường mà không cần sửa đổi source code. |
| **Health check** | Không có | Có (`/health`, `/ready`) | Là cơ chế giao tiếp bắt buộc với hệ thống điều phối (Kubernetes, Docker). Giúp tự động nhận diện agent bị treo để restart, hoặc chặn traffic vào agent chưa tải xong. |
| **Logging** | `print()` | Structured JSON (`logging`) | Cho phép phân loại mức độ sự kiện. Định dạng JSON giúp các công cụ (Datadog, ELK) dễ dàng parse, truy vấn và thiết lập cảnh báo tự động. Tránh rò rỉ secret. |
| **Shutdown** | Đột ngột | Graceful (Lifespan, SIGTERM) | Tránh tình trạng mất request khi hệ thống scale down. Cho phép agent có thời gian hoàn tất các request đang xử lý và đóng an toàn kết nối database/API. |
| **Network & Binding** | Cố định (`localhost:8000`, `reload=True`) | Động (`0.0.0.0`, port từ `$PORT`) | `0.0.0.0` bắt buộc để nhận traffic ngoài container. Port động giúp nền tảng Cloud tự động cấp phát không gây xung đột. Tắt `reload` tiết kiệm tài nguyên. |

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
### 1. Base image là gì?
Base image được sử dụng trong file là `python:3.11`. Đây là bản phân phối đầy đủ (full distribution) của Python, có dung lượng khá lớn (khoảng 1 GB).

### 2. Working directory là gì?
Working directory (thư mục làm việc mặc định bên trong container) được thiết lập là `/app`. Mọi câu lệnh `COPY`, `RUN`, hay `CMD` phía sau sẽ được thực thi tại thư mục này.

### 3. Tại sao COPY requirements.txt trước?
Thao tác này nhằm tối ưu hóa cơ chế **Docker layer cache**. 

Trong quá trình develop, mã nguồn ứng dụng sẽ thay đổi liên tục, trong khi danh sách các thư viện (`requirements.txt`) ít bị cập nhật hơn. Bằng cách copy `requirements.txt` và chạy `pip install` trước, Docker sẽ lưu lại (cache) layer chứa các thư viện này. Nếu lần build tiếp theo chỉ sửa code (không sửa `requirements.txt`), Docker sẽ sử dụng lại cache cũ thay vì phải tải và cài đặt lại toàn bộ thư viện từ đầu, giúp tiết kiệm tối đa thời gian build.

### 4. CMD vs ENTRYPOINT khác nhau thế nào?

- **CMD (Command):** Định nghĩa lệnh và tham số mặc định khi container khởi chạy. Điểm yếu là nó **rất dễ bị ghi đè** (override). Ví dụ: nếu bạn chạy `docker run <image_name> /bin/bash`, lệnh `/bin/bash` sẽ lập tức thay thế hoàn toàn lệnh `python app.py`.
- **ENTRYPOINT:** Cấu hình container chạy như một file thực thi cố định. Lệnh trong `ENTRYPOINT` **không bị ghi đè** bởi các tham số truyền qua `docker run` (trừ khi cố tình dùng cờ `--entrypoint`). Các tham số từ `docker run` sẽ được tự động nối vào phía sau chuỗi lệnh của `ENTRYPOINT` như những arguments phụ. 

**Best practice:** Thường dùng `ENTRYPOINT` để khai báo tệp thực thi chính (vd: `ENTRYPOINT ["python", "app.py"]`) và dùng `CMD` để truyền các tham số mặc định có thể thay đổi được (vd: `CMD ["--port", "8000"]`).

### Exercise 2.3: Image size comparison
- Develop: 1.66 GB
<p align="center">
  <img src="extras/myagent_develop_image_size.png" width="400">
</p>
- Production: 236 MB
<p align="center">
  <img src="extras/myagent_advanced_image_size.png" width="400">
</p>
- Difference: 85.8%

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://lab12-production-5585.up.railway.app
- Screenshot:
<p align="center">
  <img src="extras/railway-deployment.png" width="400">
</p>
- Deployment status:
<p align="center">
  <img src="extras/railway-deployment-status.png" width="400">
</p>

## Part 4: API Security

### Exercise 4.1-4.3: Test results
- **Lần 1: Không có key**
  - Câu lệnh: `curl.exe -i -X POST -H "Content-Type: application/json" -d '{"question":"hello"}' http://localhost:8000/ask`
  - Kết quả: `HTTP/1.1 401 Unauthorized`.
  - Log output:
    ```json
    {"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}
    ```
- **Lần 2: Có key "demo-key-123"**
  - Câu lệnh: `curl.exe -i -X POST -H "X-API-Key: demo-key-123" -H "Content-Type: application/json" -d '{"question":"Hello"}' http://localhost:8000/ask`
  - Kết quả: `HTTP/1.1 200 OK`.
  - Log output:
    ```json
    {
      "question": "Hello",
      "answer": "Chào bạn, tôi là Chuyên gia Tâm lý AI. Tôi luôn sẵn lòng lắng nghe bạn.",
      "usage": { "prompt_tokens": 2, "completion_tokens": 40, "total_tokens": 42, "cost": 0.000027 }
    }
    ```

#### Ex 4.2: JWT Authentication Flow
1. **Lấy Token**: Gửi username/password tới endpoint `/auth/token`.
   - Log output (Success):
     ```json
     {
       "access_token": "eyJhbGciOiJIUzI1NiIs...",
       "token_type": "bearer",
       "expires_in_minutes": 60
     }
     ```
2. **Sử dụng Token**: Token được đính kèm vào header `Authorization: Bearer <token>`.
3. **Xác thực**: Server giải mã token để lấy thông tin người dùng.

#### Ex 4.3: Rate Limiting
- **Kết quả test**: Khi gọi API liên tục 15 lần, từ request thứ 11 hệ thống trả về:
  - Log output (429):
    ```json
    {
      "detail": {
        "error": "Rate limit exceeded",
        "limit": 10,
        "window_seconds": 60,
        "retry_after_seconds": 59
      }
    }
    ```

### Exercise 4.4: Cost guard implementation
- **Cách tiếp cận**: Hệ thống theo dõi chi phí sử dụng LLM dựa trên số lượng token của input và output. 
- **Logic thực hiện**:
  1. Mỗi người dùng có một hạn mức ngân sách hàng ngày (ví dụ: $1.0/ngày).
  2. Trước khi gọi LLM, hệ thống kiểm tra ngân sách còn lại. Nếu đã hết, trả về lỗi `402 Payment Required`.
  3. Sau khi LLM phản hồi, hệ thống tính toán chi phí (dựa trên giá của model như gpt-4o-mini) và trừ vào ngân sách của người dùng đó.
  4. Có hệ thống cảnh báo (Warning log) khi người dùng sử dụng vượt quá 80% hạn mức.
- **Lợi ích**: Bảo vệ hệ thống khỏi việc bị lạm dụng hoặc các hóa đơn API LLM tăng đột biến ngoài ý muốn.

## Part 5: Scaling & Reliability

### Exercise 5.1-5.5: Implementation notes
**Nhiệm vụ:** Implement 2 endpoints:

```python
@app.get("/health")
def health():
    """Liveness probe — container còn sống không?"""
    redis_ok = False
    if USE_REDIS:
        try:
            _redis.ping()
            redis_ok = True
        except Exception:
            redis_ok = False

    status = "ok" if (not USE_REDIS or redis_ok) else "degraded"

    return {
        "status": status,
        "instance_id": INSTANCE_ID,
        "uptime_seconds": round(time.time() - START_TIME, 1),
        "storage": "redis" if USE_REDIS else "in-memory",
        "redis_connected": redis_ok if USE_REDIS else "N/A",
    }

@app.get("/ready")
def ready():
    """Readiness probe — sẵn sàng nhận traffic không?"""
    if USE_REDIS:
        try:
            _redis.ping()
        except Exception:
            raise HTTPException(503, "Redis not available")
    return {"ready": True, "instance": INSTANCE_ID}
```
```bash
/mnt/c/Users/MSI/vinuni/day12_ha-tang-cloud_va_deployment/04-api-gateway/production$ curl http://localhost:8000/health

{"status":"ok","instance_id":"instance-9f59f4","uptime_seconds":40.1,"storage":"in-memory","redis_connected":"N/A"}

/mnt/c/Users/MSI/vinuni/day12_ha-tang-cloud_va_deployment/04-api-gateway/production$ curl http://localhost:8000/ready

{"ready":true,"instance":"instance-9f59f4"}
```

### Exercise 5.2: Graceful shutdown

**Implementation analysis:**
- Server sử dụng FastAPI lifespan context manager
- Handle startup và shutdown events properly
- Log instance startup và shutdown messages

**Features implemented:**
- **Lifespan management** với asynccontextmanager
- **Startup logging** với instance ID
- **Shutdown logging** khi terminate
- **Resource cleanup** trong shutdown phase

### Exercise 5.3: Stateless design

**Implementation analysis:**

**✅ Stateless architecture:**
```python
# State trong Redis/external storage, không trong memory
def save_session(session_id: str, data: dict, ttl_seconds: int = 3600):
    """Lưu session vào Redis với TTL."""
    if USE_REDIS:
        _redis.setex(f"session:{session_id}", ttl_seconds, serialized)
    else:
        _memory_store[f"session:{session_id}"] = data

def load_session(session_id: str) -> dict:
    """Load session từ Redis."""
    if USE_REDIS:
        data = _redis.get(f"session:{session_id}")
        return json.loads(data) if data else {}
    return _memory_store.get(f"session:{session_id}", {})
```

**Test results:**

```bash
curl http://localhost:8000/chat -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```
```bash
/mnt/c/Users/MSI/vinuni/day12_ha-tang-cloud_va_deployment/04-api-gateway/production$ curl http://localhost:8000/chat -X POST \
>   -H "Content-Type: application/json" \
>   -d '{"question": "What is Docker?"}'
{"session_id":"6b2e81a1-5ac1-489f-afd8-4b1461a8fed4","question":"What is Docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!","turn":2,"served_by":"instance-9f59f4","storage":"in-memory"}
```


```bash
/mnt/c/Users/MSI/vinuni/day12_ha-tang-cloud_va_deployment/04-api-gateway/production$ curl http://localhost:8000/chat -X POST \
>   -H "Content-Type: application/json" \
>   -d '{"question": "Why do we need containers?", "session_id": "055ebc09-17be-45f1-acb2-a753aecdf4bc"}'

{"session_id":"055ebc09-17be-45f1-acb2-a753aecdf4bc","question":"Why do we need containers?","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","turn":3,"served_by":"instance-9f59f4","storage":"in-memory"}
```

```bash
# 3. Check conversation history
/mnt/c/Users/MSI/vinuni/day12_ha-tang-cloud_va_deployment/04-api-gateway/production$ curl http://localhost:8000/chat/055ebc09-17be-45f1-acb2-a753aecdf4bc/history


{"session_id":"055ebc09-17be-45f1-acb2-a753aecdf4bc","messages":[{"role":"user","content":"Why do we need containers?","timestamp":"2026-04-17T12:47:06.469903+00:00"},{"role":"assistant","content":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","timestamp":"2026-04-17T12:47:06.593912+00:00"},{"role":"user","content":"Why do we need containers?","timestamp":"2026-04-17T12:47:28.754591+00:00"},{"role":"assistant","content":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","timestamp":"2026-04-17T12:47:28.875416+00:00"}],"count":4}
```

## Exercise 5.4: Load Balancing & Scaling

### Trạng thái triển khai
Hệ thống đã thực hiện scale-out thành công bằng lệnh `docker compose up --scale agent=3 -d`. Các dịch vụ hiện đang hoạt động ổn định:

| Service | Status | Instance ID (Ví dụ) |
| :--- | :--- | :--- |
| **agent (x3)** | Up | `instance-e229ed`, `instance-3e5f7e`, `instance-e702a3` |
| **nginx** | Up | Port 8080 (External) -> 80 (Internal) |
| **redis** | Up (healthy) | Port 6379 (Internal) |

### Kết quả phân phối tải (Round-robin)
Kiểm tra qua Nginx với 10 request liên tiếp cho thấy cơ chế cân bằng tải hoạt động chính xác. Requests được luân chuyển đều qua cả 3 instance:
* **Lượt 1**: instance-e229ed
* **Lượt 2**: instance-3e5f7e
* **Lượt 3**: instance-e702a3
*(Chu kỳ lặp lại cho các lượt tiếp theo)*

---

## Exercise 5.5: Stateless Test Verification

### Kiểm thử tính nhất quán (Stateless Test)
Chạy thành công script `test_stateless.py` để xác nhận khả năng duy trì phiên làm việc trong môi trường phân tán:

* **Số lượng request**: 5 yêu cầu liên tiếp.
* **Độ bao phủ**: Cả 3 instance đều tham gia xử lý các request trong cùng một session.
* **Kết quả lưu trữ**: Lịch sử hội thoại được bảo toàn đầy đủ (**Total messages: 10**) nhờ cơ sở dữ liệu dùng chung (Redis).



### Kết luận
Hệ thống đạt tiêu chuẩn **Stateless Architecture**:
1. **Tính linh hoạt**: Client có thể kết nối tới bất kỳ instance nào mà không bị mất dữ liệu.
2. **Khả năng mở rộng**: Có thể thêm/bớt instance tùy ý mà không ảnh hưởng đến trải nghiệm người dùng.
3. **Độ tin cậy**: Dữ liệu session được tách biệt khỏi vòng đời của container (externalized state).