---
title: MIS Odoo Project
emoji: 🚀
colorFrom: yellow
colorTo: blue
sdk: docker
app_port: 7860
pinned: false
license: lgpl-3.0
---

# MIS Purchasing — Hệ thống quản lý Mua hàng, Kho & Cửa hàng trên Odoo 19

Đồ án môn **Hệ thống thông tin quản lý (MIS)** — Trường Đại học Giao thông Vận tải TP.HCM (UTH).

Bộ module tùy biến trên **Odoo 19** mô phỏng chuỗi cung ứng của một doanh nghiệp bán lẻ nhiều cửa hàng:
cửa hàng lập yêu cầu bổ sung hàng → Merchandise phê duyệt theo ngân sách → mua từ nhà cung cấp hoặc điều chuyển
từ Kho tổng → kiểm tra chất lượng (QC) khi nhận → xử lý sai lệch và bù hàng.

![Odoo](https://img.shields.io/badge/Odoo-19.0-714B67?logo=odoo&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)
![License](https://img.shields.io/badge/License-LGPL--3.0-blue)

## Module

| Module | Vai trò | Phụ thuộc chính |
| --- | --- | --- |
| [`merchandise_management`](custom_addons/merchandise_management) | Yêu cầu mua hàng (PR) nhiều cấp duyệt, ngân sách theo ngành hàng, khuyến mãi (kể cả hàng cận hạn), báo cáo sai lệch, đánh giá nhà cung cấp, dashboard | `purchase`, `stock`, `mail` |
| [`warehouse_management`](custom_addons/warehouse_management) | QC phiếu nhập (đạt / không đạt), ghi nhận hàng hư hỏng, tự tạo phiếu trả NCC hoặc hoàn trả nội bộ | `purchase_stock`, `merchandise_management` |
| [`store_management`](custom_addons/store_management) | Danh mục cửa hàng & định mức tồn, đề xuất bổ sung, luồng Kho tổng → Cửa hàng, nhận hàng dư/thiếu/lỗi, bán hàng theo cửa hàng, hóa đơn NCC đơn giản | `sale_stock`, `product_expiry`, `account`, 2 module trên |

Thứ tự cài đặt: `merchandise_management` → `warehouse_management` → `store_management`
(cài `store_management` sẽ tự kéo theo hai module còn lại).

## Quy trình nghiệp vụ chính

```text
Cửa hàng (định mức tồn) ──► PR ──► Merchandise duyệt (kiểm tra ngân sách)
                                         │
              ┌──────────────────────────┼───────────────────────────┐
              ▼                          ▼                           ▼
     NCC giao thẳng Cửa hàng   NCC giao về Kho tổng          Kho tổng có sẵn
              │                          │                           │
              └──────► QC nhận hàng ◄────┴──── Kho tổng giao Cửa hàng ┘
                            │
             Đạt: nhập kho, tạo hóa đơn NCC
             Không đạt / sai lệch: báo cáo sai lệch → PR/PO bù, phiếu trả hàng
```

Sơ đồ BPMN và báo cáo chi tiết nằm trong thư mục [`docs/`](docs).

## Chạy trên máy (Docker)

Yêu cầu: [Docker Desktop](https://www.docker.com/products/docker-desktop/).

```bash
git clone https://github.com/HoangPhucTony/Mis_Purchasing_OdooProject.git
cd Mis_Purchasing_OdooProject
docker compose up -d --build
```

Mở <http://localhost:7860>, tạo database mới (master password mặc định cho local: `admin`),
sau đó vào **Ứng dụng** và cài **Store Management**.

Cập nhật module sau khi sửa code:

```bash
docker compose run --rm odoo odoo -u merchandise_management,warehouse_management,store_management -d <ten_db> --stop-after-init
```

## Triển khai (Hugging Face Spaces)

Mỗi lần push lên nhánh `main`, GitHub Actions ([`sync_to_hf.yml`](.github/workflows/sync_to_hf.yml))
đồng bộ mã nguồn sang Hugging Face Space; Space build lại từ [`Dockerfile`](Dockerfile).

Biến môi trường cần khai báo trong *Settings → Variables and secrets* của Space
(xem mẫu tại [`.env.example`](.env.example)):

| Biến | Ý nghĩa |
| --- | --- |
| `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD` | Kết nối PostgreSQL bên ngoài |
| `ODOO_MASTER_PASSWORD` | Mật khẩu Database Manager — **bắt buộc đặt giá trị mạnh** |
| `DB_FILTER` | Regex giới hạn database hiển thị |
| `MAX_CRON_THREADS` | Số luồng cron (mặc định `1`; `0` để tắt) |

Secret của repository GitHub: `HF_DEPLOY_ODOO` (Hugging Face access token có quyền ghi).

## Cấu trúc thư mục

```text
.
├── custom_addons/
│   ├── merchandise_management/
│   ├── store_management/
│   └── warehouse_management/
├── docs/                    # Báo cáo đồ án, sơ đồ BPMN
├── .github/workflows/       # CI/CD đồng bộ Hugging Face
├── Dockerfile               # Image chạy trên Hugging Face Spaces
├── docker-compose.yml       # Môi trường phát triển local (Odoo + PostgreSQL)
└── .env.example
```

## Giấy phép

Phát hành theo giấy phép [LGPL-3.0](LICENSE), tương thích với Odoo Community.
