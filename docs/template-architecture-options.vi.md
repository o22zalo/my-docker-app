# Đề xuất chuyển `my-docker-app` thành template triển khai đa dịch vụ

## Mục tiêu bạn đặt ra (diễn giải lại)

1. **Giảm biến env + tự động hóa domain/subdomain**:
   - Ví dụ chỉ cần khai báo một domain gốc (`gitea.dpdns.org`) và tự map ra:
     - `mainapp.gitea.dpdns.org`
     - `files.gitea.dpdns.org`
     - `logs.gitea.dpdns.org`
     - `webssh.gitea.dpdns.org`
2. **Dùng lại stack như một template**:
   - Chỉ chỉnh env là chạy.
   - Dễ thay `app` Node.js bằng image dịch vụ khác (OmniRoute, Gitea, ...).
3. **Bật/tắt thành phần theo nhu cầu**:
   - Ví dụ disable Tailscale.
4. **Ít phụ thuộc môi trường triển khai**:
   - Chạy được với GitHub Actions, Azure Pipelines, self-hosted.
5. **Mở rộng nhiều dịch vụ dễ dàng**.

---

## Hiện trạng codebase (điểm mạnh/yếu nhanh)

### Điểm mạnh đang có
- Đã có reverse proxy bằng **Caddy + labels**, hợp lý cho template động.
- Đã có Cloudflared/Tailscale/WebSSH/Dozzle/Filebrowser để phục vụ vận hành.
- Đã có skeleton CI cho GitHub Actions + Azure Pipelines.

### Điểm gây “nặng tay” khi scale dịch vụ
- `docker-compose.yml` đang hardcode khá nhiều service + labels lặp lại.
- Env đặt tên chưa đồng nhất (`TS_AUTHKEY` trong `.env.example` nhưng compose dùng `TAILSCALE_CLIENT_SECRET`).
- Logic subdomain đang phân tán theo từng biến/service.
- Chưa có cơ chế “khai báo 1 nơi -> generate route/service tự động”.

---

## Các phương án (sắp theo mức **ít phải thêm code** trước)

## Phương án A — Chuẩn hóa Compose base + override profile + naming convention (ít code nhất)

## Ý tưởng
- Giữ Docker Compose làm trung tâm.
- Tách thành:
  - `compose.base.yml`: caddy, cloudflared, filebrowser, dozzle, webssh (optional)
  - `compose.apps.yml`: các app service (nodejs, gitea, omniroute...)
  - `compose.features.yml`: tailscale/webssh/windows-linux split
- Dùng `profiles` để bật/tắt (vd: `--profile tailscale`, `--profile observability`).
- Chuẩn hóa env cực gọn:
  - `BASE_DOMAIN=gitea.dpdns.org`
  - `MAIN_APP_NAME=mainapp`
  - `SERVICES_ENABLED=filebrowser,dozzle,webssh`
  - `MAIN_IMAGE=gitea/gitea:latest`
  - `MAIN_PORT=3000`
- Dùng naming convention cho host:
  - `${MAIN_APP_NAME}.${BASE_DOMAIN}`
  - `files.${BASE_DOMAIN}`
  - `logs.${BASE_DOMAIN}`
  - `webssh.${BASE_DOMAIN}`

## Ưu điểm
- Nhanh triển khai, gần cấu trúc hiện tại.
- Team dễ học vì vẫn là Docker Compose thuần.
- CI hiện tại sửa ít.

## Nhược điểm
- Khi số lượng dịch vụ tăng cao, compose sẽ dài.
- Vẫn còn mức độ lặp labels nếu không có bước generate.

## Độ phức tạp
- **Thấp**.

## Khi nên chọn
- Muốn đi nhanh trong 1–2 vòng refactor.
- Muốn ít thay đổi quy trình vận hành hiện tại.

---

## Phương án B — Compose + “manifest dịch vụ” + script generate compose/env (cân bằng tốt nhất)

## Ý tưởng
- Giữ runtime là Docker Compose, nhưng thêm lớp cấu hình trung gian:
  - `stack.services.yml` (manifest): khai báo các service muốn bật.
- Viết script (Node.js/Python) để generate:
  - `docker-compose.generated.yml`
  - `.env.generated`
  - (tuỳ chọn) `cloudflared/config.generated.yml`
- Mỗi service khai báo ngắn gọn:
```yaml
services:
  mainapp:
    image: gitea/gitea:latest
    port: 3000
    host_prefix: mainapp
    protected: true
  files:
    image: filebrowser/filebrowser:latest
    port: 80
    host_prefix: files
    enabled: true
```
- Script tự render Caddy labels + hostname theo `BASE_DOMAIN`.
- Bật/tắt tailscale bằng flag duy nhất (`ENABLE_TAILSCALE=true/false`).

## Ưu điểm
- Env cực gọn, không còn “mỗi service một biến host/subdomain”.
- Scale thêm dịch vụ nhanh (thêm 1 block YAML).
- Giảm sai sót do copy-paste labels.

## Nhược điểm
- Cần thêm script generate + validate schema.
- Team cần học bước “generate trước khi up”.

## Độ phức tạp
- **Trung bình**.

## Khi nên chọn
- Muốn template dùng lâu dài, thêm dịch vụ thường xuyên.
- Muốn tái sử dụng cho nhiều dự án/khách hàng.

---

## Phương án C — Helm/K8s hoặc Nomad (linh hoạt lớn, nhiều công nhất)

## Ý tưởng
- Chuyển template sang Helm chart (hoặc Terraform + Nomad/K8s).
- Mỗi service là values block, ingress/domain routing do ingress controller xử lý.

## Ưu điểm
- Chuẩn enterprise, scale lớn, multi-environment bài bản.
- Tái sử dụng rất tốt nếu tổ chức đã có hạ tầng orchestration.

## Nhược điểm
- Overkill với nhu cầu hiện tại.
- Chi phí chuyển đổi + vận hành cao.

## Độ phức tạp
- **Cao**.

## Khi nên chọn
- Đã có cluster/K8s team chuyên trách.

---

## Khuyến nghị thực tế theo mục tiêu của bạn

- **Khuyến nghị chính: chọn Phương án B** (manifest + generate) vì cân bằng nhất giữa:
  - ít env,
  - thêm dịch vụ linh hoạt,
  - vẫn giữ Docker Compose dễ deploy ở self-hosted/GitHub/Azure.
- Nếu muốn đi siêu nhanh trong tuần này: làm **A trước**, rồi nâng dần sang B.

---

## Thiết kế env tinh gọn đề xuất

## Bộ env lõi (ít biến)
```env
# bắt buộc
BASE_DOMAIN=gitea.dpdns.org
MAIN_APP_NAME=mainapp
MAIN_IMAGE=gitea/gitea:latest
MAIN_PORT=3000

# auth chung
BASIC_AUTH_USER=admin
BASIC_AUTH_HASH=...

# feature flags
ENABLE_CLOUDFLARED=true
ENABLE_TAILSCALE=false
ENABLE_WEBSSH=true
ENABLE_FILEBROWSER=true
ENABLE_DOZZLE=true

# optional
TAILSCALE_AUTHKEY=
```

## Quy tắc tự sinh host
- Main app: `${MAIN_APP_NAME}.${BASE_DOMAIN}` → `mainapp.gitea.dpdns.org`
- Filebrowser: `files.${BASE_DOMAIN}`
- Dozzle: `logs.${BASE_DOMAIN}`
- WebSSH: `webssh.${BASE_DOMAIN}`
- Service mới `foo`: `foo.${BASE_DOMAIN}` (mặc định)

=> Không cần mỗi service giữ một env subdomain riêng trừ khi override.

---

## Lộ trình triển khai gợi ý (ít rủi ro)

1. **Chuẩn hóa naming/env hiện tại** (không đổi kiến trúc):
   - Đồng bộ `TS_AUTHKEY` vs `TAILSCALE_CLIENT_SECRET`.
   - Đổi toàn bộ về prefix thống nhất (`ENABLE_*`, `BASE_*`, `MAIN_*`).
2. **Tách compose thành base + overrides + profiles**.
3. **Thêm manifest + generator** để loại lặp labels/host.
4. **Cập nhật CI**:
   - step generate
   - step validate config
   - step compose up từ file generated.
5. **Đóng gói template**:
   - `scripts/init-template.sh` (hoặc `node scripts/init-template.mjs`)
   - tạo nhanh `.env` + sample manifest theo domain.

---

## “Build thành một docker image để sử dụng” — có nên không?

Bạn có thể làm, nhưng nên hiểu rõ:

- Nếu ý là **đóng gói toàn bộ stack thành 1 image duy nhất**: **không khuyến nghị** (khó vận hành, đi ngược best practice 1-process/1-container).
- Nếu ý là **đóng gói tool generate/deploy thành 1 image CLI** (ví dụ `my-stackctl`): **khuyến nghị**.
  - Image này chạy lệnh generate + validate + deploy compose.
  - Giúp CI/CD môi trường nào cũng dùng giống nhau.

Kết luận: nên container hóa **deployment tool**, không container hóa toàn bộ services vào một container.

---

## So sánh nhanh

| Tiêu chí | PA A | PA B | PA C |
|---|---:|---:|---:|
| Lượng code thêm | Ít nhất | Trung bình | Nhiều |
| Tốc độ áp dụng | Nhanh nhất | Nhanh-vừa | Chậm |
| Khả năng mở rộng dịch vụ | Trung bình | Cao | Rất cao |
| Phù hợp self-hosted đơn giản | Rất phù hợp | Rất phù hợp | Kém phù hợp |
| Rủi ro chuyển đổi | Thấp | Trung bình thấp | Cao |

---

## Đề xuất bước tiếp theo (nếu bạn đồng ý)

Mình có thể triển khai theo 2 nhịp:

1. **Nhịp 1 (nhanh):** refactor theo PA A để chạy ổn ngay, env gọn hơn.
2. **Nhịp 2 (nâng cấp):** thêm manifest + script generate theo PA B.

Nếu bạn muốn, ở bước kế tiếp mình sẽ tạo luôn:
- skeleton `compose.base.yml`, `compose.features.yml`, `compose.apps.yml`
- mẫu `stack.services.yml`
- script generate ban đầu (MVP).
