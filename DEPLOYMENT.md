# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Phạm Quân |
| Mã học viên | 2A202602890 |
| Repo | https://github.com/phamquan123158/K4-L3A-DAY12-PhamQuan-2A202602890-CloudServiceAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-8bc1.up.railway.app |
| Platform | Railway |
| Ngày deploy | Chưa xác minh; endpoint được kiểm tra ngày 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway cấp tự động |
| `AGENT_API_KEY` | ✅ | Giá trị lưu trong Railway Variables, không ghi ở đây |
| `REDIS_URL` | ✅ | Railway reference: `${{redis.REDIS_URL}}`; `/ready` cần xác nhận sau redeploy |
| `RATE_LIMIT_PER_MINUTE` | ⚠️ | Chưa xác minh trong Railway Variables; mặc định ứng dụng là 10 |
| `MONTHLY_BUDGET_USD` | ⚠️ | Chưa xác minh trong Railway Variables; mặc định ứng dụng là 10.0 |
| `LOG_LEVEL` | ⚠️ | Chưa xác minh trong Railway Variables; mặc định ứng dụng là INFO |

## Lệnh Kiểm Tra

Các lệnh dưới đây dùng Public URL đã deploy:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent-production-8bc1.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent-production-8bc1.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent-production-8bc1.up.railway.app/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời. Đặt AGENT_API_KEY trong shell an toàn.
curl -i -X POST https://day12-agent-production-8bc1.up.railway.app/ask -H "Content-Type: application/json" -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv-test" -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent-production-8bc1.up.railway.app/ask -H "Content-Type: application/json" -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: sv-test" -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Đã xác minh ngày 2026-09-28, không gửi API key:

```text
/health: HTTP 200 {"status":"ok","service":"day12-agent","version":"1.0.0"}
/ready: HTTP 500 (Redis readiness chưa đạt; cần kiểm tra logs và REDIS_URL trên Railway)
/ask không có API key: HTTP 401
/ask có API key và rate limit: chưa kiểm tra; DEPLOY_API_KEY chưa được cấu hình cục bộ.
CP5 public: 7 passed, 1 failed (/ready), 5 skipped (authenticated ask chưa có DEPLOY_API_KEY; local fallback không dùng).
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — cần chụp từ Railway Dashboard
- `screenshots/health.png` — cần chụp kết quả gọi `/health`

---
