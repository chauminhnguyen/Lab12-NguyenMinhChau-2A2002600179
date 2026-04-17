# Deployment Information

## Public URL
https://my-production-agent-production-db98.up.railway.app

## Platform
Railway

## Test Commands

### Health Check
```bash
curl https://my-production-agent-production-db98.up.railway.app/health
```

### Readiness Check
```bash
crylake@DESKTOP-L1EAHF5:/mnt/d/vinuni/Lab12-NguyenMinhChau-2A202600179$ curl -i https://my-production-agent-production-049b.up.railway.app/health
HTTP/2 200 
x-railway-cdn-edge: fastly/cache-hkg17932-HKG
content-type: application/json
date: Fri, 17 Apr 2026 18:43:07 GMT
referrer-policy: strict-origin-when-cross-origin
server: railway-edge
x-content-type-options: nosniff
x-frame-options: DENY
x-railway-edge: railway/asia-southeast1-eqsg3a
x-railway-request-id: jERLtHNITPKRijlFAXC71g
x-cache: MISS
x-cache-hits: 0
x-served-by: cache-hkg17932-HKG
content-length: 193

{"status":"degraded","version":"1.0.0","environment":"development","uptime_seconds":22.2,"total_requests":2,"checks":{"llm":"mock","redis":false},"timestamp":"2026-04-17T18:43:08.080375+00:00"}crylake@DESKTOP-L1EAHF5:/mnt/d/vinuni/Lab12-NguyenMinhChau-2A202600179$
```

### API Test (with authentication)
```bash
crylake@DESKTOP-L1EAHF5:/mnt/d/vinuni/Lab12-NguyenMinhChau-2A202600179$ curl -X POST https://my-production-agent-production-049b.up.railway.app/ask -H "X-API-Key:123" -H "Content-Type: application/json" -d '{"question": "Hello"}'ask -H "X-API-Key:123" -H "Content-Type: application/json" -d '{"question": "Hello"}'
{"question":"Hello","answer":"This is a mock response. Replace with real LLM provider in production.","model":"gpt-4o-mini","timestamp":"2026-04-17T18:43:48.264831+00:00"}crylake@DESKTOP-L1EAHF5:/mnt/d/vinuni/Lab12-NguyenMinhChau-2A202600179$

```
### Without authentication

```
crylake@DESKTOP-L1EAHF5:/mnt/d/vinuni/Lab12-NguyenMinhChau-2A202600179$ curl -X POST https://my-production-agent-production-049b.up.railway.app/ask -H "Content-Type: application/json" -d '{"question": "Hello"}'      
{"detail":"Invalid or missing API key. Include header X-API-Key."}
```

## Environment Variables Set
- PORT: Port mà ứng dụng listen (mặc định 8080 trên Railway)
- REDIS_URL: Kết nối tới Redis instance (để stateless)
- AGENT_API_KEY: Key bảo mật để truy cập API
- LOG_LEVEL: Mức độ ghi log (INFO/DEBUG)
- DAILY_BUDGET_USD: Hạn mức chi phí mỗi ngày

## Screenshots
- [Deployment dashboard](extras/railway-deployment.png)
- [Service running](extras/railway-deployment-status.png)
- [Test results](extras/API-Security-with-key.png)
