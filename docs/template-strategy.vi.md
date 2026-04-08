# Đề xuất chuyển `my-docker-app` thành template triển khai đa dịch vụ

> Mục tiêu: dùng lại stack hiện tại để triển khai nhiều loại dịch vụ (Node.js, Omniroute, Gitea, ...), chỉ cần đổi ít biến môi trường, bật/tắt thành phần như Tailscale/WebSSH/Dozzle/Filebrowser, và có thể deploy ở GitHub Actions, Azure Pipelines, self-host.

---

## 1) Hiện trạng nhanh (điểm nghẽn chính)

1. **Biến env đang rời rạc theo từng service** (`SUBDOMAIN_APP`, `SUBDOMAIN_DOZZLE`, `SUBDOMAIN_FILEBROWSER`, ...), gây phải chỉnh tay nhiều khi thêm/xoá dịch vụ.
2. **Dịch vụ chính (`app`) đang hard-code theo Node.js build local** nên khi thay bằng image khác thì phải sửa compose.
3. **Bật/tắt dịch vụ phụ thuộc profile theo OS** (linux-only/windows-only), nhưng chưa có cơ chế bật/tắt logic nghiệp vụ rõ bằng env (vd `ENABLE_TAILSCALE=false`).
4. **Cloudflared config cần chỉnh thủ công** theo hostname/service map.

---

## 2) Nguyên tắc thiết kế template mới

- **Một nguồn sự thật (single source of truth)** cho domain/subdomain.
- **Service catalog**: mô tả dịch vụ theo format chuẩn, có `name`, `image`, `port`, `enabled`, `expose_public`.
- **Feature flags qua env**: `ENABLE_TAILSCALE`, `ENABLE_WEBSSH`, `ENABLE_FILES`, `ENABLE_LOGS`.
- **Không ràng buộc app type**: service chính chạy từ image cấu hình (`PRIMARY_IMAGE`) thay vì build cố định Node.
- **Compose tối giản + profile rõ ràng**: tránh fork file compose cho từng use case.

---

## 3) Các phương án (sắp theo mức ít code phải thêm → nhiều code hơn)

## Phương án A — “Chuẩn hoá env + compose override” (ít code nhất, triển khai nhanh)

### Ý tưởng
- Giữ `docker-compose.yml` làm base.
- Tách thành:
  - `docker-compose.base.yml` (caddy, cloudflared, network, auth chung)
  - `docker-compose.services.yml` (các dịch vụ business)
  - `docker-compose.ops.yml` (dozzle/filebrowser/webssh/tailscale)
- Dùng env tối giản theo quy ước:
  - `BASE_DOMAIN=gitea.dpdns.org`
  - `STACK_SLUG=mainapp`
  - `PRIMARY_NAME=browser`
  - `PRIMARY_IMAGE=your-image:tag`
  - `PRIMARY_PORT=8080`
- Rule hostname tự sinh bằng convention:
  - `${STACK_SLUG}.${BASE_DOMAIN}` cho primary
  - `files.${STACK_SLUG}.${BASE_DOMAIN}`
  - `logs.${STACK_SLUG}.${BASE_DOMAIN}`
  - `ssh.${STACK_SLUG}.${BASE_DOMAIN}`

### Cách bật/tắt
- Bật/tắt bằng compose profile + env:
  - `COMPOSE_PROFILES=core,ops` hoặc `core`
  - `ENABLE_TAILSCALE=false` → không include `tailscale` override file.

### Ưu / Nhược
- ✅ Nhanh nhất, ít rủi ro, gần với code hiện tại.
- ✅ Team dễ hiểu vì vẫn là Docker Compose chuẩn.
- ⚠️ Khi dịch vụ tăng nhiều, số file override tăng theo.
- ⚠️ Tự động hoá hostname vẫn theo convention “cứng”, chưa phải generator.

### Độ phức tạp
- **Thấp** (1–2 ngày refactor).

---

## Phương án B — “Template có bước render config” (khuyến nghị cân bằng)

### Ý tưởng
- Giữ compose template với placeholders (hoặc dùng `docker compose config` + envsubst).
- Thêm **script render** (`scripts/render-stack.sh` hoặc Node/Python) đọc 1 file khai báo ngắn, ví dụ `stack.env` + `services.yml`.
- Script sinh ra:
  - `generated/docker-compose.generated.yml`
  - `generated/cloudflared/config.yml`
- Mỗi dịch vụ chỉ cần khai báo 1 block:

```yaml
services:
  - name: browser
    image: ghcr.io/.../browser:latest
    port: 8080
    public: true
  - name: gitea
    image: gitea/gitea:latest
    port: 3000
    public: true
  - name: omniroute
    image: your/omniroute:latest
    port: 9000
    public: false
```

Script tự gán host:
- `browser.mainapp.gitea.dpdns.org`
- `gitea.mainapp.gitea.dpdns.org`
- `omniroute.mainapp.gitea.dpdns.org` (chỉ nội bộ nếu `public: false`)

### Cách bật/tắt
- Trong `stack.env`:
  - `ENABLE_TAILSCALE=false`
  - `ENABLE_FILEBROWSER=true`
  - `ENABLE_WEBSSH=true`

### Ưu / Nhược
- ✅ Giảm mạnh số env cần nhớ.
- ✅ Add service mới chỉ thêm 1 block, không chạm compose lõi.
- ✅ Dễ mở rộng CI (GitHub/Azure/self-host) vì chỉ cần chạy bước “render + up”.
- ⚠️ Cần thêm script + validate schema.

### Độ phức tạp
- **Trung bình** (2–5 ngày).

---

## Phương án C — “Đóng gói thành Docker image điều phối (stack launcher image)” (linh hoạt cao, code nhiều hơn)

### Ý tưởng
- Build 1 image riêng (vd `my-stack-launcher`) chứa:
  - template compose,
  - script render,
  - rule generate cloudflared/caddy labels,
  - health-check orchestration.
- Khi deploy chỉ cần mount file cấu hình vào launcher và chạy:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(pwd)/stack-config:/config \
  ghcr.io/your-org/my-stack-launcher:latest deploy
```

### Ưu / Nhược
- ✅ Trải nghiệm “template productized”, rất tiện nhân bản nhiều môi trường.
- ✅ Chuẩn hoá tuyệt đối giữa GitHub Actions, Azure, self-host.
- ⚠️ Phức tạp hơn (version launcher, backward compatibility, debug).
- ⚠️ Cần thiết kế tốt vấn đề bảo mật docker.sock.

### Độ phức tạp
- **Trung bình-cao** (1–2 tuần).

---

## Phương án D — “Helm/K8s-first” (không khuyến nghị nếu ưu tiên nhanh)

### Ý tưởng
- Chuyển toàn bộ sang chart Helm + values cho mỗi service.

### Ưu / Nhược
- ✅ Mạnh khi scale lớn.
- ⚠️ Overkill với mục tiêu hiện tại, tăng đáng kể chi phí vận hành.

### Độ phức tạp
- **Cao**.

---

## 4) So sánh nhanh

| Phương án | Code thêm | Tốc độ ra kết quả | Dễ dùng cho team | Mở rộng nhiều dịch vụ | Phù hợp hiện tại |
|---|---:|---:|---:|---:|---:|
| A. Env + override | Ít nhất | Nhanh nhất | Cao | Trung bình | Rất phù hợp |
| B. Render template | Vừa | Nhanh | Cao | Cao | **Phù hợp nhất** |
| C. Launcher image | Nhiều | Trung bình | Rất cao (sau khi xong) | Rất cao | Phù hợp giai đoạn 2 |
| D. Helm/K8s | Rất nhiều | Chậm | Trung bình | Rất cao | Chưa nên làm ngay |

---

## 5) Đề xuất lộ trình thực thi (khuyên dùng)

### Giai đoạn 1 (nhanh, ít rủi ro)
- Làm **Phương án A** để chuẩn hoá env + tách compose base/ops/services.
- Mục tiêu: giảm 30–50% số biến env phải chạm khi tạo stack mới.

### Giai đoạn 2 (template tái sử dụng thật sự)
- Nâng lên **Phương án B**:
  - thêm `services.yml` + script render,
  - tự sinh cloudflared mapping + caddy labels.
- Mục tiêu: add dịch vụ mới chỉ thêm 1 block config.

### Giai đoạn 3 (nếu cần scale nhiều team/môi trường)
- Productize theo **Phương án C** (launcher image).

---

## 6) Thiết kế env tối giản đề xuất (bản mục tiêu)

```dotenv
# bắt buộc
BASE_DOMAIN=gitea.dpdns.org
STACK_SLUG=mainapp
CADDY_EMAIL=admin@gitea.dpdns.org
CADDY_AUTH_USER=admin
CADDY_AUTH_HASH=...

# dịch vụ chính
PRIMARY_NAME=browser
PRIMARY_IMAGE=ghcr.io/your-org/browser:latest
PRIMARY_PORT=8080

# feature flags
ENABLE_CLOUDFLARE=true
ENABLE_TAILSCALE=false
ENABLE_FILEBROWSER=true
ENABLE_DOZZLE=true
ENABLE_WEBSSH=true

# tunnel
CF_TUNNEL_TOKEN=...
```

Từ đó tự suy ra:
- Public primary: `browser.mainapp.gitea.dpdns.org`
- Ops:
  - `files.mainapp.gitea.dpdns.org`
  - `logs.mainapp.gitea.dpdns.org`
  - `ssh.mainapp.gitea.dpdns.org`

---

## 7) Gợi ý CI/CD ít phụ thuộc nền tảng

- Chuẩn hoá 1 entry command dùng chung mọi nơi:

```bash
./scripts/deploy.sh
```

Bên trong gồm 3 bước cố định:
1. `render` config từ env + catalog.
2. `docker compose up -d --remove-orphans` với file generated.
3. `collect artifacts`.

Ánh xạ vào:
- GitHub Actions: chỉ gọi `./scripts/deploy.sh`.
- Azure Pipelines: chỉ gọi `./scripts/deploy.sh`.
- Self-host (cron/manual): gọi cùng command.

---

## 8) Khuyến nghị chốt

Nếu ưu tiên **ít sửa code và ra kết quả nhanh**: chọn **Phương án A** ngay.

Nếu ưu tiên **template tái sử dụng lâu dài, add dịch vụ cực nhanh**: đi theo **A → B** (khuyến nghị tốt nhất).

Nếu sau này nhiều team dùng chung và cần “1 lệnh deploy mọi nơi” rất chuẩn hoá: nâng lên **C**.

---

## 9) Checklist khi bắt đầu triển khai (để tránh breaking)

- [ ] Chuẩn hoá naming env (tránh trùng `TS_AUTHKEY` vs `TAILSCALE_CLIENT_SECRET`).
- [ ] Tách rõ service core và service ops.
- [ ] Viết validation script cho env bắt buộc.
- [ ] Thử matrix deploy: Linux + Windows + self-host.
- [ ] Có sample `services.yml` cho Omniroute + Gitea + Browser.
- [ ] Có rollback command rõ ràng (`docker compose down && up` theo version trước).

