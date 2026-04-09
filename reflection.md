# Reflection — Day 06

**Họ tên:** Dương Trịnh Hoài An  
**Nhóm:** Nhom26 — E403  
**Project:** Vinmec Pre-Visit Checklist Chatbot  

---

## Role & Đóng góp

**Phần đảm nhận:** CI/CD Pipeline + Containerization + Deployment

### Output cụ thể

| Hạng mục | Chi tiết |
|----------|----------|
| **Dockerfile (backend)** | Viết `backend/Dockerfile` — image `python:3.11-slim`, cài deps theo layer cache, chạy non-root user (`appuser`), healthcheck tự động tại `/health` |
| **Dockerfile (frontend)** | Viết `backend/Dockerfile.frontend` — build React/Vite rồi serve qua nginx |
| **docker-compose** | Cấu hình `docker-compose.yml` gốc (root-level) kết nối backend `:5000` và frontend `:8080`; cấu hình `backend/docker-compose.yml` cho stack đầy đủ gồm backend, frontend, Weaviate vector DB, SearXNG |
| **GitHub Actions — CI** | Viết `.github/workflows/ci.yml`: path-filter để chỉ build service thay đổi (backend / frontend / mobile), chạy lint + test song song, cancel-in-progress để tiết kiệm runner |
| **GitHub Actions — Publish** | Viết `.github/workflows/docker-publish.yml`: tự động build & push Docker image lên GitHub Container Registry (GHCR) khi merge vào `main` hoặc tag `v*` |
| **Deploy link** | *(`https://vinmec.ngtdt204.id.vn/`)* |

---

## Reflection cá nhân

### Điều tôi làm được

Phần CI/CD ban đầu tôi nghĩ sẽ đơn giản — chỉ là "thêm vài file YAML". Thực tế phức tạp hơn nhiều. Vì repo có 3 sub-project độc lập (backend Python, frontend React, mobile React Native), nếu push code frontend mà CI lại chạy test toàn bộ backend thì vừa chậm vừa tốn runner. Tôi phải học cách dùng **path-filter** (`dorny/paths-filter`) để chỉ trigger đúng job cần thiết — cái này mất khá nhiều thời gian debug vì `workflow_dispatch` có behavior khác với `push`.

Phần Docker cũng dạy tôi một bài về **layer caching**: lúc đầu tôi copy toàn bộ source code trước rồi mới `pip install`, mỗi lần sửa 1 dòng code là phải cài lại 2 phút. Sau khi đảo lại thứ tự — copy `requirements.txt` trước, install, rồi mới copy source — build time giảm đáng kể.

### Điều tôi chưa làm tốt

Chưa setup tốt phần **staging environment** tách biệt với production. Hiện tại pipeline push thẳng image lên GHCR và deploy luôn, chưa có bước "deploy to staging → smoke test → promote to prod". Với project thật ở công ty thì đây là gap lớn.


### Bài học lớn nhất từ Day 06

> "Trước hackathon, em nghĩ làm AI product chủ yếu là nghĩ ra feature rồi gắn model vào.   Sau khi làm bài này, em hiểu rõ hơn rằng phần khó nhất không phải chỉ là model, mà là: 

- xác định scope đủ hẹp để làm được

- thiết kế trust cho đúng ngữ cảnh

- và viết prompt sao cho bot vừa hữu ích vừa không đi quá giới hạn.  

Và em nghĩ deploy là bước cuối của pipeline — viết code, test, build image, push lên server, xong.

Sau khi làm thực tế, em hiểu rõ hơn rằng deploy không phải điểm kết thúc, mà là điểm bắt đầu của một câu hỏi khác: service đang chạy như thế nào?

Ví dụ cụ thể: em viết healthcheck trong Dockerfile (curl /health mỗi 15 giây), nhưng lúc đầu không để ý healthcheck đó báo gì khi Weaviate chưa kịp khởi động. Container vẫn healthy theo Docker, nhưng API thực tế trả lỗi vì vector DB chưa sẵn sàng. Nếu không có log và không nhìn vào response thực tế, em sẽ nghĩ deploy thành công trong khi product đang broken.

Đó là lúc em nhận ra: deploy chỉ có nghĩa khi đi kèm với khả năng quan sát — log tập trung, healthcheck đúng dependency, smoke test sau deploy. Thiếu cái đó thì pipeline chạy xanh cũng chưa có nghĩa là hệ thống đang hoạt động.

---
