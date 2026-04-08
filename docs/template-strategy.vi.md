# Định hướng chuyển `my-docker-app` thành template đa dịch vụ

## Mục tiêu

- Dùng cùng một bộ khung để triển khai nhiều dịch vụ khác nhau (Node.js app, Gitea, Omniroute, v.v.).
- Giảm số lượng biến môi trường phải sửa thủ công.
- Bật/tắt dịch vụ bằng cấu hình (ví dụ tắt Tailscale khi không dùng).
- Vẫn giữ các nghiệp vụ vận hành quan trọng: logs, files, web terminal, reverse proxy, tunnel.
- Dễ triển khai trên GitHub Actions, Azure Pipelines, self-host.

---

## Đánh giá nhanh hiện trạng

Hiện tại stack đã có nền tốt:
- `caddy-docker-proxy` đọc label từ Docker services để route tự động.
- Có đủ “ops tools” (Dozzle, Filebrowser, WebSSH).
- Đã dùng Docker Compose profiles cho Linux/Windows.

Điểm chưa tối ưu cho mô hình template:
- Env đang phân tán theo từng service (`SUBDOMAIN_*`, nhiều biến lặp).
- Route hostnames chưa có cơ chế sinh tự động từ 1-2 biến gốc.
- Khó thay “app image” bằng service khác mà vẫn đồng bộ convention (domain, auth, volume, enable/disable).

---

## Nguyên tắc thiết kế template mới

1. **Base domain-driven**: lấy 1 biến gốc (vd `BASE_DOMAIN=gitea.dpdns.org`) và 1 prefix stack (vd `STACK=mainapp`) để tự sinh hostnames.
2. **Service registry**: định nghĩa danh sách service theo chuẩn chung (`name`, `image`, `port`, `subdomain`, `enabled`).
3. **Composable layers**:
   - `compose.base.yml`: hạ tầng chung (caddy, optional tunnel/tailscale, ops tools).
   - `compose.services.yml`: các ứng dụng nghiệp vụ.
   - `compose.override.yml`: tuỳ môi trường.
4. **Feature flags**: bật/tắt qua env + profile, tránh phải sửa YAML nhiều nơi.
5. **One-command deploy**: 1 lệnh build/render + `docker compose up -d`.

---

## Phương án đề xuất (xếp theo ít phải code thêm -> nhiều hơn)

## Phương án A — “Compose thuần + chuẩn hoá env/profiles” (ít code nhất)

### Ý tưởng
- Giữ Docker Compose làm trung tâm.
- Chuẩn hoá biến env về mức tối thiểu:
  - `BASE_DOMAIN`
  - `STACK`
  - `ENABLE_TAILSCALE=true|false`
  - `ENABLE_CLOUDFLARED=true|false`
  - `ENABLE_WEBSSH=true|false`
- Dùng Compose `profiles` để bật/tắt nhóm service.
- Đặt quy ước subdomain mặc định:
  - app chính: `${STACK}.${BASE_DOMAIN}`
  - files: `files.${STACK}.${BASE_DOMAIN}`
  - logs: `logs.${STACK}.${BASE_DOMAIN}`
  - ssh: `ssh.${STACK}.${BASE_DOMAIN}`

### Ưu điểm
- Triển khai nhanh, gần như không thay đổi kiến trúc.
- Team dễ tiếp cận, ít công cụ mới.
- Phù hợp nếu số lượng service thêm không quá lớn (<=10-15).

### Nhược điểm
- Vẫn cần sửa compose khi thêm service mới nhiều.
- Khả năng tái sử dụng xuyên dự án ở mức trung bình.

### Mức effort
- **Thấp** (1-2 ngày): refactor env + profile + docs.

---

## Phương án B — “Template generator (khuyên dùng)”

### Ý tưởng
- Tạo 1 file khai báo stack (vd `stack.yaml`) rồi sinh ra:
  - `.env.generated`
  - `compose.generated.yml`
  - `cloudflared/config.generated.yml`
- Viết script nhẹ (Node/Python) để render template (Jinja2/Handlebars/yq).
- Người dùng chỉ sửa `stack.yaml` + 1 file secrets.

### Ví dụ cấu hình mức cao
```yaml
stack: mainapp
base_domain: gitea.dpdns.org
features:
  cloudflared: true
  tailscale: false
  webssh: true
services:
  - key: gitea
    image: gitea/gitea:latest
    port: 3000
    subdomain: mainapp
  - key: filebrowser
    image: filebrowser/filebrowser:latest
    port: 80
    subdomain: files
  - key: dozzle
    image: amir20/dozzle:latest
    port: 8080
    subdomain: logs
  - key: omniroute
    image: your-org/omniroute:latest
    port: 8081
    subdomain: route
```

### Ưu điểm
- Scale tốt cho nhiều dịch vụ và nhiều môi trường.
- Giảm lỗi cấu hình tay, giảm số env lặp.
- Dễ đóng gói thành “template product” để tái dùng nhiều repo.

### Nhược điểm
- Cần thêm 1 lớp build/render (script + template).
- Team cần học luồng mới (`generate -> deploy`).

### Mức effort
- **Trung bình** (3-6 ngày): script generator + chuẩn cấu hình + CI wiring.

---

## Phương án C — “Helm/K8s-lite hoặc Nomad module” (nhiều code/công nhất)

### Ý tưởng
- Nâng từ Compose template sang chart/module orchestration.
- Dùng values để bật/tắt service + tạo ingress/tunnel tương ứng.

### Ưu điểm
- Chuẩn enterprise, mở rộng mạnh ở quy mô lớn.
- Quản trị đa cluster/multi-env tốt.

### Nhược điểm
- Overkill cho nhu cầu self-host nhỏ/nhóm vừa.
- Tăng đáng kể độ phức tạp vận hành và CI/CD.

### Mức effort
- **Cao** (2-4 tuần hoặc hơn).

---

## Khuyến nghị chốt

- **Ngắn hạn**: bắt đầu từ **Phương án A** để “dọn” env/profile ngay.
- **Trung hạn (nên làm)**: tiến tới **Phương án B** để đạt đúng mục tiêu “chỉ chỉnh env/config là deploy được nhiều dịch vụ”.
- **Dài hạn**: chỉ lên C khi thật sự cần scale lớn hoặc nhiều team vận hành độc lập.

---

## Thiết kế env tinh gọn đề xuất

## Bộ biến tối thiểu (public config)

```dotenv
# Core identity
STACK=mainapp
BASE_DOMAIN=gitea.dpdns.org

# Exposure mode
EXPOSE_MODE=cloudflare   # cloudflare | public | local

# Feature toggles
ENABLE_TAILSCALE=false
ENABLE_WEBSSH=true
ENABLE_FILEBROWSER=true
ENABLE_DOZZLE=true

# Shared auth
ADMIN_AUTH_USER=admin
ADMIN_AUTH_HASH=<bcrypt>
```

## Bộ biến secrets
- `CF_TUNNEL_TOKEN` hoặc `cloudflared credentials file`
- `TAILSCALE_AUTHKEY`
- SSH private key cho webssh (nếu bật)

> Tách secrets thành `.env.secrets` hoặc secret manager CI (GitHub/Azure variable group).

---

## Quy ước route hostname tự động

Dùng 1 hàm quy ước (trong script generator hoặc naming convention):
- `main`: `${STACK}.${BASE_DOMAIN}` -> ví dụ `mainapp.gitea.dpdns.org`
- `files`: `files.${STACK}.${BASE_DOMAIN}` -> `files.mainapp.gitea.dpdns.org`
- `logs`: `logs.${STACK}.${BASE_DOMAIN}`
- `ssh`: `ssh.${STACK}.${BASE_DOMAIN}`
- service custom `X`: `${X}.${STACK}.${BASE_DOMAIN}`

Nhờ vậy khi đổi từ `gitea.dpdns.org` sang domain khác, chỉ đổi **1 biến**.

---

## Cách đóng gói template để tái sử dụng

## Lựa chọn 1: “Template repo” (đơn giản nhất)
- Giữ repo này làm “golden template”.
- Dự án mới dùng “Use this template” trên GitHub.
- Chạy script init để đặt `STACK`, `BASE_DOMAIN`, chọn service cần bật.

## Lựa chọn 2: “CLI init”
- Viết lệnh `npx my-stack-init` hoặc `python -m stack_init`.
- Sinh file compose/env từ blueprint.
- Phù hợp khi muốn dùng chung cho cả GitHub/Azure/self-host.

## Lựa chọn 3: “Đóng gói thành 1 Docker image điều phối”
- Tạo image chứa script generator + docker compose plugin.
- Chạy container để render/deploy stack.
- Tiện cho môi trường runner đồng nhất nhưng cần cân nhắc quyền Docker socket.

---

## Gợi ý pipeline “ít phụ thuộc”

## GitHub Actions
- Job 1: validate config (`stack.yaml`, schema).
- Job 2: generate compose/env.
- Job 3: `docker compose config` check.
- Job 4: deploy (self-hosted runner hoặc remote Docker host).

## Azure Pipelines
- Tương tự GitHub Actions, tách stage `Validate -> Generate -> Deploy`.
- Secrets lấy từ Variable Group / KeyVault.

## Self-host (không CI)
- Script local:
  - `./bin/generate`
  - `./bin/deploy`
  - `./bin/status`

---

## Lộ trình triển khai đề xuất

1. **Phase 1 (cleanup, nhanh)**
   - Chuẩn hoá env: gom còn 8-12 biến chính.
   - Chuyển enable/disable service sang profile/toggle rõ ràng.
   - Viết docs migration từ `.env` cũ.

2. **Phase 2 (template hóa)**
   - Tạo `stack.yaml` + schema validate.
   - Viết generator sinh compose/env/cloudflared config.
   - Update CI để chạy generate + validate tự động.

3. **Phase 3 (mở rộng dịch vụ)**
   - Thêm catalog service mẫu: gitea, omniroute, n8n, minio, postgres...
   - Mỗi service có module gắn volume/network/healthcheck chuẩn.

---

## Rủi ro và cách giảm thiểu

- **Rủi ro route conflict** khi nhiều service trùng subdomain  
  -> Validate unique subdomain lúc generate.

- **Rủi ro lộ secrets**  
  -> Cấm commit `.env.secrets`, dùng secret store.

- **Rủi ro phụ thuộc OS** (Linux/Windows runner)  
  -> Giữ profile tách riêng `linux-only`, `windows-only`, validate từng profile.

---

## Kết luận ngắn

Nếu mục tiêu là **nhanh, ít sửa code, dùng lại ngay**: chọn **Phương án A**.

Nếu mục tiêu là **template chuẩn, thêm dịch vụ liên tục, ít sai config tay**: đi theo **Phương án B** (đề xuất chính).

