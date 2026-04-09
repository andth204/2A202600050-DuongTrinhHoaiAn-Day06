# Đề xuất CI/CD cho hệ thống — Extras

**Tác giả:** Dương Trịnh Hoài An  
**Ngữ cảnh:** Đề xuất cải tiến pipeline sau khi hoàn thành Day 06

---

## Vấn đề với pipeline hiện tại

Pipeline trong sprint này đã chạy được — build, test, push image lên GHCR — nhưng có 3 điểm mà nếu đưa vào production thật sẽ gặp vấn đề:

**1. Không có staging environment**  
Hiện tại pipeline merge vào `main` là deploy thẳng lên production. Nếu có bug thoát qua test, user thật sẽ thấy ngay. Không có chỗ để kiểm tra lần cuối trước khi release.

**2. Healthcheck chưa đủ thông minh**  
Healthcheck trong Dockerfile chỉ ping `/health` của backend. Nhưng backend có thể trả `200 OK` trong khi Weaviate chưa sẵn sàng — lúc đó bot vẫn nhận request nhưng RAG trả về rỗng. User không thấy lỗi, nhưng câu trả lời sai hoàn toàn.

**3. Không có smoke test sau deploy**  
Sau khi container lên, pipeline không tự kiểm tra xem flow thực sự có chạy không. Chỉ biết "container đang chạy", không biết "chatbot đang trả lời đúng".

---

## Đề xuất cải tiến

### Sơ đồ pipeline mục tiêu

```
push to main
     │
     ▼
[CI] lint + unit test
     │
     ▼
[Build] docker build + push image :staging
     │
     ▼
[Deploy Staging] deploy lên môi trường staging
     │
     ▼
[Smoke Test] gọi thử /health + 1 câu hỏi mẫu → kiểm tra response
     │
  ┌──┴──┐
PASS    FAIL
  │       └─→ alert + dừng, không promote
  ▼
[Promote] re-tag image :staging → :latest
     │
     ▼
[Deploy Production]
     │
     ▼
[Post-deploy check] ping production, log kết quả
```

### Chi tiết từng bước

#### Smoke test sau deploy staging

Thay vì chỉ check container health, chạy một script gọi thực tế vào API:

```python
# scripts/_run_smoke.py (đã có trong repo, cần tích hợp vào CI)
import httpx, sys

BASE = "https://staging.nhom26-vinmec.fly.dev"

def test_health():
    r = httpx.get(f"{BASE}/health", timeout=5)
    assert r.status_code == 200, f"Health check failed: {r.status_code}"

def test_chat_response():
    r = httpx.post(f"{BASE}/chat", json={"message": "Tôi cần khám xét nghiệm máu"}, timeout=10)
    assert r.status_code == 200
    data = r.json()
    # Kiểm tra response có nội dung thực, không phải fallback lỗi
    assert len(data.get("answer", "")) > 20, "Response quá ngắn — có thể RAG bị lỗi"

if __name__ == "__main__":
    try:
        test_health()
        test_chat_response()
        print("✓ Smoke test passed")
    except AssertionError as e:
        print(f"✗ Smoke test failed: {e}")
        sys.exit(1)
```

#### Healthcheck phân tầng trong docker-compose

```yaml
# Thay vì chỉ check backend, check cả dependency
services:
  weaviate:
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:8080/v1/.well-known/ready"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    depends_on:
      weaviate:
        condition: service_healthy   # ← đợi Weaviate thật sự sẵn sàng
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:8000/health"]
      interval: 15s
      timeout: 5s
      retries: 3    
```

Cái này giải quyết đúng bug em gặp trong sprint: backend báo healthy nhưng Weaviate chưa lên xong.

#### Tách workflow thành 3 file rõ ràng

| File | Trigger | Làm gì |
|------|---------|--------|
| `ci.yml` | mọi PR, mọi push | lint, unit test, build check |
| `staging.yml` | merge vào `main` | build → deploy staging → smoke test |
| `release.yml` | tag `v*` | promote image staging → prod, deploy prod, post-check |

Hiện tại `ci.yml` và `docker-publish.yml` đang gộp một số việc lại, hơi khó đọc khi cần debug xem bước nào fail.

---

## Ưu tiên nếu có thêm thời gian

1. **Tích hợp `_run_smoke.py`** vào CI — file đã có sẵn trong repo, chỉ cần thêm 1 job trong `staging.yml`
2. **`depends_on: condition: service_healthy`** cho Weaviate — sửa nhỏ nhưng fix đúng bug healthcheck
3. **Tách `staging.yml` / `release.yml`** — dài hơn nhưng dễ maintain hơn về sau

---
