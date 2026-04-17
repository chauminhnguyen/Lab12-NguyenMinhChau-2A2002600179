# Lab 12 — Complete Production Agent

Kết hợp TẤT CẢ những gì đã học trong 1 project hoàn chỉnh.

## Checklist Deliverable

- [x] Dockerfile (multi-stage, < 500 MB)
- [x] docker-compose.yml (agent + redis)
- [x] .dockerignore
- [x] Health check endpoint (`GET /health`)
- [x] Readiness endpoint (`GET /ready`)
- [x] API Key authentication
- [x] Rate limiting
- [x] Cost guard
- [x] Config từ environment variables
- [x] Structured logging
- [x] Graceful shutdown
- [x] Public URL ready (Railway / Render config)

---

## Cấu Trúc

```
my-production-agent/
├── app/
│   ├── main.py         # Entry point — kết hợp tất cả
│   ├── config.py       # 12-factor config
│   ├── auth.py         # API Key + JWT
│   ├── rate_limiter.py # Rate limiting
│   └── cost_guard.py   # Budget protection
├── Dockerfile          # Multi-stage, production-ready
├── docker-compose.yml  # Full stack
├── railway.toml        # Deploy Railway
├── render.yaml         # Deploy Render
├── .env.example        # Template
├── .dockerignore
└── requirements.txt
```

---

## Chạy Local

```bash
# 1. Setup
cp .env.example .env

# 2. Chạy với Docker Compose
docker compose up

# 3. Test
curl http://localhost/health
curl http://localhost/ready

# 4. Lấy API key từ .env, test endpoint
API_KEY=$(grep AGENT_API_KEY .env | cut -d= -f2)
curl -H "X-API-Key: $API_KEY" \
     -X POST http://localhost/ask \
     -H "Content-Type: application/json" \
     -d '{"question": "What is deployment?"}'
```

---

## Deploy Railway bằng CLI (chi tiết)

Playbook này dành cho trường hợp deploy mới hoàn toàn, có thể đổi account, tránh link nhầm service, và chạy được ngay.

### 0) Đứng đúng thư mục project

```bash
cd day12_ha-tang-cloud_va_deployment/my-production-agent
```

### 1) Cài CLI và đăng nhập đúng account

```bash
# Có thể dùng npx (không cần cài global)
npx @railway/cli login

# Kiểm tra account hiện tại
npx @railway/cli whoami
npx @railway/cli list
```

Nếu cần đổi account:

```bash
npx @railway/cli logout
rm -rf .railway
rm -rf ~/.railway ~/.config/railway
npx @railway/cli login
```

### 2) Tạo project mới và link local folder

```bash
# Interactive tạo project mới
npx @railway/cli init

# Kiểm tra context hiện tại
npx @railway/cli status
```

### 3) Tạo Redis service trong project

```bash
npx @railway/cli add --database redis --service redis

# Link sang Redis service để lấy connection string
npx @railway/cli service link redis
npx @railway/cli variables
```

Lưu lại giá trị `REDIS_URL` dạng nội bộ:

```txt
redis://default:<REDIS_PASSWORD>@redis.railway.internal:6379
```

### 4) Tạo app service và deploy snapshot đầu tiên

```bash
# Tạo service app (nếu chưa có)
npx @railway/cli add --service my-production-agent

# Link về app service (KHONG chon Redis)
npx @railway/cli service link my-production-agent
npx @railway/cli status

# Deploy lan dau de tao snapshot
npx @railway/cli up --detach
```

### 5) Tạo secret và set toàn bộ biến môi trường cho app

Tạo nhanh secret an toàn:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"
```

Set biến (trong app service):

```bash
npx @railway/cli variable set ENVIRONMENT=production
npx @railway/cli variable set AGENT_API_KEY=<paste_api_key_hex>
npx @railway/cli variable set JWT_SECRET=<paste_jwt_secret_hex>
npx @railway/cli variable set REDIS_URL='redis://default:<REDIS_PASSWORD>@redis.railway.internal:6379'
npx @railway/cli variable set RATE_LIMIT_PER_MINUTE=10
npx @railway/cli variable set MONTHLY_BUDGET_USD=10.0

# Verify
npx @railway/cli variables
```

### 6) Deploy lại sau khi set biến

```bash
npx @railway/cli up --detach
npx @railway/cli domain
```

### 7) Test production endpoint

```bash
curl -i https://<your-domain>/health
curl -i https://<your-domain>/ready
```

Test endpoint protected:

```bash
curl -i -X POST https://<your-domain>/ask \
     -H "X-API-Key: <AGENT_API_KEY>" \
     -H "Content-Type: application/json" \
     -d '{"question":"What is deployment?"}'
```

### 8) Troubleshooting nhanh

- `Cannot redeploy without a snapshot`: chưa có deploy đầu tiên cho app service. Chạy `npx @railway/cli up --detach` trong app service trước.
- Mở domain bị `404`: thường do link nhầm service Redis hoặc app chưa running. Kiểm tra `npx @railway/cli status` + `npx @railway/cli logs`.
- Link nhầm service: chạy `npx @railway/cli service` và chọn lại app service.
- Không chạy lệnh có chữ giải thích như `Deploy lại:` trong terminal. Chỉ copy dòng command.

---

## Deploy Render

1. Push repo lên GitHub
2. Render Dashboard → New → Blueprint
3. Connect repo → Render đọc `render.yaml`
4. Set env vars bắt buộc: `ENVIRONMENT=production`, `AGENT_API_KEY`, `JWT_SECRET`, `REDIS_URL`
5. Set env vars gợi ý: `RATE_LIMIT_PER_MINUTE=10`, `MONTHLY_BUDGET_USD=10.0`
6. Deploy → Nhận URL!

---

## Kiểm Tra Production Readiness

```bash
python check_production_ready.py
```

Script này kiểm tra tất cả items trong checklist và báo cáo những gì còn thiếu.
