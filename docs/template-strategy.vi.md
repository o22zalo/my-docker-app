# Định hướng chuyển `my-docker-app` thành template triển khai đa dịch vụ

> Mục tiêu: dùng lại stack này như một **platform template** để triển khai nhiều dịch vụ (Node.js app, Gitea, Omniroute, …), giảm số lượng biến môi trường phải chỉnh tay, và bật/tắt tính năng (Tailscale, WebSSH, Filebrowser, Logs...) bằng cấu hình đơn giản.

---

## 1) Hiện trạng nhanh của codebase

Từ `docker-compose.yml` hiện tại, stack đã có nền tảng rất tốt:

- reverse proxy qua `caddy-docker-proxy` bằng labels động;
- tunnel public qua `cloudflared`;
- nhóm dịch vụ vận hành (`dozzle`, `filebrowser`, `webssh`);
- tách profile Linux/Windows cho Tailscale & WebSSH.

### Điểm đang gây khó khi scale nhiều dịch vụ

1. **Subdomain env đang tách rời theo service** (`SUBDOMAIN_APP`, `SUBDOMAIN_DOZZLE`, …) → thêm service mới là thêm env mới.
2. **Compose định nghĩa cứng từng service** → muốn thay app image khác cần sửa compose nhiều chỗ.
3. **Bật/tắt tính năng dựa profile + comment/sửa tay** chưa có một chuẩn “feature flags” thống nhất.
4. **Cloudflare ingress** chưa có cơ chế tự sinh map hostname từ 1 nguồn cấu hình duy nhất.

---

## 2) Nguyên tắc thiết kế template nên theo

1. **Single source of truth**: chỉ 1 file khai báo dịch vụ + route.
2. **Convention over configuration**: theo quy ước tên host tự sinh, không bắt user nhập nhiều env.
3. **Composable**: tách `compose.base` + `compose.features` + `compose.services`.
4. **Feature flags rõ ràng**: `ENABLE_TAILSCALE=false` là tắt hoàn toàn.
5. **Portable CI/CD**: chạy được local, self-hosted runner, GitHub Actions, Azure Pipelines.

---

## 3) Các phương án (sắp theo mức ít code thay đổi nhất)

## Phương án A — “Compose thuần + profiles + naming convention” (ít code nhất)

### Ý tưởng

- Giữ Docker Compose làm lõi.
- Chuẩn hoá naming:
  - `BASE_DOMAIN=gitea.dpdns.org`
  - `STACK_NAME=mainapp`
  - Host mặc định sinh theo công thức: `{service}.{STACK_NAME}.{BASE_DOMAIN}`
- Dịch vụ nào optional thì cho vào profile: `obs`, `access`, `tailscale`, `ops`...
- Node app chỉ là **một service mặc định**, có thể thay image bằng env:
  - `MAIN_IMAGE=ghcr.io/org/myapp:latest` (thay cho build app cứng).

### Bật/tắt tính năng

- `COMPOSE_PROFILES=core,obs` (không có `tailscale` ⇒ Tailscale disabled).
- Hoặc split file:
  - `docker compose -f compose.yml -f compose.obs.yml up -d`.

### Ưu điểm

- Nhanh nhất, rủi ro thấp.
- Không cần thêm runtime/tool mới.
- Team dễ tiếp cận.

### Nhược điểm

- Vẫn phải khai báo service trong compose bằng tay.
- Auto-generate cloudflare ingress chưa triệt để.

### Độ phức tạp

- **Thấp** (1/5).

---

## Phương án B — “Template có generator” (khuyên dùng cân bằng)

### Ý tưởng

- Tạo 1 file khai báo mức cao, ví dụ `stack.services.yml`:

```yaml
base_domain: gitea.dpdns.org
stack_name: mainapp
features:
  tailscale: false
  webssh: true
  dozzle: true
  filebrowser: true
services:
  - name: gitea
    image: gitea/gitea:latest
    port: 3000
    expose_via_caddy: true
  - name: omniroute
    image: your-org/omniroute:latest
    port: 8080
    expose_via_caddy: true
```

- Viết script generator (Node/Python):
  - sinh `docker-compose.generated.yml`;
  - sinh `cloudflared/config.generated.yml`;
  - tự gắn labels Caddy + hostname theo convention.

### Bật/tắt tính năng

- Chỉ đổi `features.*` trong 1 file.
- `tailscale: false` thì generator không render service tailscale.

### Ưu điểm

- Giảm env cực mạnh.
- Scale thêm service nhanh: thêm 1 block YAML là xong.
- Tránh drift giữa compose và cloudflared config.

### Nhược điểm

- Cần maintain generator script.
- Cần thêm bước `generate` trong pipeline.

### Độ phức tạp

- **Trung bình** (3/5).

---

## Phương án C — “Đóng gói thành Docker image orchestrator”

### Ý tưởng

- Build một image riêng kiểu `my-docker-app-template:<version>` chứa:
  - binary/script generator,
  - template compose,
  - command `init` / `render` / `deploy`.
- Người dùng chỉ mount file config + `.env` tối giản, chạy 1 lệnh để render và deploy.

### Ưu điểm

- Trải nghiệm “cài đặt một lần, dùng nhiều nơi”.
- Rất phù hợp self-host / private platform nội bộ.

### Nhược điểm

- Tăng effort build/release version cho template image.
- Debug khó hơn một chút so với compose thuần.

### Độ phức tạp

- **Trung bình-cao** (4/5).

---

## Phương án D — “Chuyển lên Helm/K8s nhẹ (k3s)”

### Ý tưởng

- Biến stack thành Helm chart; mỗi service là values entry.
- Ingress hostnames sinh từ pattern.

### Ưu điểm

- Scale lớn rất tốt, chuẩn cloud-native.

### Nhược điểm

- Quá nặng với mục tiêu “template Docker triển khai nhanh”.
- Tăng phụ thuộc hạ tầng mạnh.

### Độ phức tạp

- **Cao** (5/5).

---

## 4) So sánh nhanh

| Tiêu chí | A. Compose thuần | B. Generator | C. Template Image | D. Helm/K8s |
|---|---:|---:|---:|---:|
| Thời gian áp dụng | Rất nhanh | Nhanh-vừa | Vừa | Chậm |
| Số code thêm | Ít nhất | Vừa | Vừa-nhiều | Nhiều |
| Giảm env thủ công | Trung bình | Rất tốt | Rất tốt | Rất tốt |
| Dễ thêm service mới | Trung bình | Tốt | Tốt | Rất tốt |
| Dễ debug | Rất tốt | Tốt | Trung bình | Trung bình |
| Phù hợp mục tiêu hiện tại | Tốt | **Tốt nhất** | Tốt (giai đoạn 2) | Thấp |

---

## 5) Đề xuất triển khai theo pha (thực dụng)

## Pha 1 (ưu tiên ngay, ít code)

Theo **Phương án A**:

1. Refactor compose thành modules:
   - `compose.core.yml` (caddy, cloudflared, network)
   - `compose.ops.yml` (dozzle, filebrowser, webssh)
   - `compose.access.yml` (tailscale linux/windows)
   - `compose.apps.yml` (service ứng dụng)
2. Chuẩn hoá 4-6 env cốt lõi:
   - `BASE_DOMAIN`, `STACK_NAME`, `CADDY_AUTH_USER`, `CADDY_AUTH_HASH`, `CADDY_EMAIL`, `COMPOSE_PROFILES`
3. Đổi app service sang image-param:
   - `image: ${MAIN_IMAGE}` thay vì build cứng.

## Pha 2 (khuyến nghị mạnh)

Nâng lên **Phương án B**:

1. Tạo `stack.services.yml` làm nguồn khai báo duy nhất.
2. Thêm script `generate`:
   - render compose + cloudflared config.
3. CI pipeline chạy:
   - `generate` → `validate` (`docker compose config`) → `deploy`.

## Pha 3 (tuỳ chọn)

Nếu team triển khai nhiều project, nhiều environment:

- đóng gói theo **Phương án C** thành image CLI template.

---

## 6) Bộ env tối giản đề xuất

```env
# Identity stack
STACK_NAME=mainapp
BASE_DOMAIN=gitea.dpdns.org

# Auth / ingress
CADDY_EMAIL=admin@example.com
CADDY_AUTH_USER=admin
CADDY_AUTH_HASH=<bcrypt>

# Feature flags (nếu dùng generator thì đặt ở stack.services.yml)
ENABLE_TAILSCALE=false
ENABLE_WEBSSH=true
ENABLE_DOZZLE=true
ENABLE_FILEBROWSER=true

# Main business service image
MAIN_IMAGE=gitea/gitea:latest
MAIN_PORT=3000
```

### Quy tắc hostname tự động

- Service business chính: `${STACK_NAME}.${BASE_DOMAIN}` → `mainapp.gitea.dpdns.org`
- Service phụ:
  - `files.${STACK_NAME}.${BASE_DOMAIN}`
  - `logs.${STACK_NAME}.${BASE_DOMAIN}`
  - `ssh.${STACK_NAME}.${BASE_DOMAIN}`
  - `gitea.${STACK_NAME}.${BASE_DOMAIN}` (nếu muốn alias riêng)

> Kết quả: thêm service mới chỉ cần khai báo tên service, không cần thêm biến subdomain riêng.

---

## 7) Tác động đến CI/CD (GitHub Actions, Azure, self-host)

### Mục tiêu chung

- Dùng cùng một entrypoint:
  - `./scripts/render.sh` (nếu có generator)
  - `docker compose -f ... up -d --remove-orphans`

### GitHub Actions

- giữ self-hosted hoặc đổi runner tùy nhu cầu;
- thêm bước validate config trước deploy.

### Azure Pipelines

- luồng hiện tại có thể giữ nguyên;
- chỉ thêm bước render (nếu dùng phương án B/C).

### Self-host manual

- 1 lệnh chuẩn giống CI để tránh khác biệt môi trường.

---

## 8) Khuyến nghị chọn phương án

- Nếu cần làm ngay, ít thay đổi nhất: **A**.
- Nếu muốn “template thật sự dùng lâu dài”, ít env, thêm service nhanh: **B (khuyến nghị chính)**.
- Nếu sau này muốn product hóa nội bộ: nâng từ B lên **C**.

---

## 9) Danh sách việc cụ thể nếu bạn đồng ý triển khai tiếp

1. Refactor compose thành nhiều file + profile rõ ràng.
2. Thiết kế schema `stack.services.yml`.
3. Viết script generator bản đầu (render compose + cloudflared).
4. Chuẩn hoá tài liệu `.env.example` mới (rút biến).
5. Cập nhật pipeline GitHub/Azure để chạy theo flow mới.

