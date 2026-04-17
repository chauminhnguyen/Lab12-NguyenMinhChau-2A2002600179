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
curl https://my-production-agent-production-db98.up.railway.app/
```

Result:
```
StatusCode        : 200
StatusDescription : 
Content           : {"status":"degraded","version":"1.0.0","environment":"development","uptime_seconds":3083.2,"total_requests
                    ":20,"checks":{"llm":"mock","redis":false},"timestamp":"2026-04-17T18:06:40.689922+00:00"}
RawContent        : HTTP/1.1 200 
                    Connection: keep-alive
                    x-railway-cdn-edge: fastly/cache-hkg17926-HKG
                    referrer-policy: strict-origin-when-cross-origin
                    x-content-type-options: nosniff
                    x-frame-options: DENY
                    x-railwa...
Forms             : {}
Headers           : {[Connection, keep-alive], [x-railway-cdn-edge, fastly/cache-hkg17926-HKG], [referrer-policy,
                    strict-origin-when-cross-origin], [x-content-type-options, nosniff]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 196
```

### API Test (with authentication)
```bash
curl -X POST https://my-production-agent-production-db98.up.railway.app/ask -H "X-API-Key: my-secret-key-123" -H "Content-Type: application/json" -d '{"question": "Hello"}'
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
