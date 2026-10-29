<div align="center">

# 🛒 MIS Purchasing — Quản lý Mua hàng bán lẻ trên Odoo 19

**Bổ sung hàng cửa hàng · Yêu cầu mua hàng (PR) · Kiểm soát ngân sách · QC nhận hàng · Xử lý sai lệch · Khuyến mãi hàng cận hạn**

[![Odoo](https://img.shields.io/badge/Odoo-19.0-714B67?logo=odoo&logoColor=white)](https://www.odoo.com/)
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)](docker-compose.yml)
[![Hugging Face](https://img.shields.io/badge/Demo-Hugging%20Face%20Spaces-FFD21E?logo=huggingface&logoColor=black)](https://huggingface.co/spaces/IceMisha/OdooProject)
[![License](https://img.shields.io/badge/License-LGPL--3.0-blue)](LICENSE)

[English](README.md) · **Tiếng Việt**

</div>

> Đồ án môn **Đồ án thực tế Hệ thống thông tin quản lý** — Trường Đại học Giao thông Vận tải TP.HCM (UTH).
> Đề tài: *Phân tích quy trình nghiệp vụ quản lý mua hàng trong hệ thống ERP — Ứng dụng triển khai trên nền tảng
> Odoo tại Central Retail*. Phần phân tích dựa trên khảo sát quy trình thực tế; repository là sản phẩm học tập,
> không phải sản phẩm chính thức của Central Retail.

---

## 📑 Mục lục

- [Tổng quan](#-tổng-quan)
- [Từ AS-IS đến TO-BE](#-từ-as-is-đến-to-be)
- [Kiến trúc](#-kiến-trúc)
- [Tính năng theo vai trò](#-tính-năng-theo-vai-trò)
- [Luồng nghiệp vụ](#-luồng-nghiệp-vụ)
- [Ảnh giao diện](#-ảnh-giao-diện)
- [Phân tích GAP](#-phân-tích-gap)
- [Cài đặt](#-cài-đặt)
- [Triển khai](#-triển-khai)
- [Cấu trúc thư mục](#-cấu-trúc-thư-mục)
- [Đánh giá & hướng phát triển](#-đánh-giá--hướng-phát-triển)
- [Nhóm thực hiện](#-nhóm-thực-hiện)
- [Giấy phép](#-giấy-phép)

## 🔎 Tổng quan

Doanh nghiệp bán lẻ nhiều cửa hàng bổ sung hàng từ **hai nguồn**: kho tổng và nhà cung cấp (NCC). Khảo sát cho thấy
dữ liệu giữa cửa hàng, Merchandise, kho và NCC chưa đồng bộ theo thời gian thực, nhiều công đoạn còn thủ công qua
email, chưa có cảnh báo sớm cho thiếu hàng, dư hàng, hàng cận hạn hay sai lệch giao nhận.

Đề tài mô hình hóa quy trình cải tiến **TO-BE** bằng BPMN, phân tích **GAP** so với Odoo tiêu chuẩn và lấp các khoảng
trống bằng ba module tùy biến trên Odoo 19 Community. **Yêu cầu mua hàng (PR)** trở thành chứng từ trung tâm, liên kết
nhu cầu cửa hàng, quyết định của Merchandise, đơn mua hàng (PO), phiếu điều chuyển, kiểm tra chất lượng và bù hàng.

## 🔁 Từ AS-IS đến TO-BE

| Nội dung | AS-IS | TO-BE (đã triển khai) |
| --- | --- | --- |
| Tạo yêu cầu mua hàng | Thủ công, thiếu đồng bộ, phụ thuộc nhân viên | Đề xuất PR theo định mức min/max, phân luồng theo nguồn cung |
| Phát hiện thiếu hàng | Kiểm tra định kỳ thủ công | Hệ thống theo dõi tồn khả dụng + đang chờ bổ sung theo cửa hàng |
| Hàng cận hạn | Xử lý thủ công, dễ trễ | Tự phát hiện lô sắp hết hạn (số ngày cấu hình theo sản phẩm) |
| Khuyến mãi | Tạo tay, khó tra cứu | Scheduler tạo hằng ngày, tra cứu theo mã, sản phẩm, cửa hàng |
| Sai lệch nhận hàng | Trao đổi qua email | Báo cáo thiếu/dư/lỗi liên kết PR bù, PO bù hoặc phiếu thu hồi |
| Kiểm soát chi | Kiểm tra sau | Chặn phê duyệt khi vượt ngân sách ngành hàng |

## 🏗 Kiến trúc

```mermaid
flowchart LR
    subgraph Odoo["Odoo 19 Community"]
        purchase[purchase]
        stock[stock]
        sale[sale_stock]
        expiry[product_expiry]
        account[account]
    end
    MER["merchandise_management<br/><sub>PR · ngân sách · khuyến mãi · sai lệch · dashboard</sub>"]
    WH["warehouse_management<br/><sub>QC nhận hàng · trả hàng</sub>"]
    ST["store_management<br/><sub>cửa hàng · min/max · Kho tổng→Cửa hàng · bán hàng</sub>"]
    purchase --> MER
    stock --> MER
    MER --> WH
    MER --> ST
    WH --> ST
    sale --> ST
    expiry --> ST
    account --> ST
```

| Module | Chức năng |
| --- | --- |
| [`merchandise_management`](custom_addons/merchandise_management) | PR duyệt nhiều cấp, ngân sách theo ngành hàng, vòng đời SKU & phân loại ABC, dừng đặt hàng, khuyến mãi (kể cả hàng cận hạn), báo cáo sai lệch, đánh giá NCC, dashboard (OWL) |
| [`warehouse_management`](custom_addons/warehouse_management) | QC phiếu nhập (đạt / không đạt), ghi nhận hàng hư hỏng, tự tạo phiếu trả NCC hoặc hoàn trả nội bộ |
| [`store_management`](custom_addons/store_management) | Danh mục cửa hàng & định mức min/max, đề xuất bổ sung, luồng Kho tổng → Cửa hàng qua kho trung chuyển, kiểm hàng thực tế (thiếu/dư/lỗi), bán hàng theo cửa hàng áp giá KM, hóa đơn NCC đơn giản |

Cài **Store Management** sẽ tự cài kèm hai module còn lại.

## 👥 Tính năng theo vai trò

<details open>
<summary><b>🏪 Cửa hàng (Store)</b></summary>

- Danh mục cửa hàng với người phụ trách, mức ưu tiên bổ sung; kho cửa hàng được tạo tự động.
- Định mức **tồn tối thiểu / tối đa** theo sản phẩm: tồn hiện có, khả dụng, đang chờ bổ sung và **số lượng đề xuất**.
- Tốc độ bán 30 ngày, **số ngày tồn dự kiến**; trạng thái: cháy hàng, thiếu, bình thường, tồn cao, dư hàng.
- Tạo PR bổ sung một chạm; chặn PR trùng sản phẩm đang xử lý.
- Nhận hàng có bước kiểm đếm thực tế, ghi nhận thiếu / dư / hàng lỗi.
- Đơn bán chỉ chọn sản phẩm của cửa hàng, tự áp giá và hạn mức khuyến mãi.
- Chỉ số kinh doanh: doanh thu, lợi nhuận gộp, tỷ suất lợi nhuận.

</details>

<details open>
<summary><b>📋 Merchandise</b></summary>

- Quy trình PR: *Nháp → Đã gửi → Chờ quản lý duyệt → Được phê duyệt → Đang thực hiện → Hoàn tất* (hoặc *Từ chối / Hủy*).
- **Phương án đáp ứng theo từng dòng**: Kho tổng có sẵn, NCC giao về Kho tổng, NCC giao thẳng Cửa hàng; hiển thị tồn kho nguồn.
- **Ngân sách mua hàng** theo ngành hàng: đã thực chi (PO), đã cam kết (PR), còn lại; từ chối duyệt khi vượt ngân sách.
- Vòng đời SKU (mới, đang kinh doanh, bán chậm, xả hàng, ngừng kinh doanh), phân loại ABC, cờ *dừng đặt hàng*.
- Từ báo cáo sai lệch tạo nhanh PR bù / PO bù hoặc phiếu thu hồi.
- Khuyến mãi theo cửa hàng có giới hạn số lượng; **scheduler hằng ngày** tạo KM hàng cận hạn và cập nhật giá KM.
- Điểm uy tín, xếp hạng và lead-time trung bình của NCC; dashboard KPI, pipeline PR và hoạt động gần đây.

</details>

<details open>
<summary><b>🏭 Kho (Warehouse)</b></summary>

- **QC nhận hàng**: bắt đầu kiểm → đạt (xác nhận nhập kho) hoặc không đạt (bắt buộc ghi chú).
- Số lượng hư hỏng và ghi chú lỗi theo dòng; theo dõi chênh lệch dự kiến – thực nhận.
- Phiếu bị từ chối tự sinh phiếu trả NCC hoặc hoàn trả nội bộ.
- Kho tổng kiểm hàng cho yêu cầu cửa hàng, giao qua kho trung chuyển và giữ nguyên số lô.
- Tự tạo hóa đơn NCC sau khi nhận hàng.

</details>

## 🔄 Luồng nghiệp vụ

```mermaid
flowchart TD
    A([Cửa hàng: tồn dưới mức tối thiểu]) --> B[PR bổ sung hàng]
    B --> C{Merchandise xem xét<br/>+ kiểm tra ngân sách}
    C -- từ chối --> X([Từ chối])
    C -- phê duyệt --> D{Phương án đáp ứng}
    D -- Kho tổng có sẵn --> E[Kho tổng kiểm hàng]
    D -- NCC giao về Kho tổng --> F[PO gửi NCC]
    D -- NCC giao thẳng Cửa hàng --> G[PO gửi NCC]
    F --> H[Kho tổng nhận hàng + QC]
    H --> E
    E --> I[Điều chuyển về cửa hàng qua kho trung chuyển]
    G --> J[Cửa hàng nhận hàng + QC]
    I --> J
    J -- đạt --> K([Cập nhật tồn · hóa đơn NCC · PR hoàn tất])
    J -- thiếu / dư / lỗi --> L[Báo cáo sai lệch]
    L --> M[PR/PO bù hoặc phiếu thu hồi]
    M --> K
```

Sơ đồ BPMN đầy đủ (Bizagi): [`docs/SODOBPMN.pdf`](docs/SODOBPMN.pdf).

## 🖼 Ảnh giao diện

| Định mức min/max & bổ sung hàng | Phương án đáp ứng trên PR |
| :---: | :---: |
| ![Định mức cửa hàng](docs/images/store-replenishment.png) | ![Phương án đáp ứng](docs/images/pr-fulfillment-options.png) |
| **Ngân sách mua hàng theo ngành hàng** | **Tồn kho theo số lô & hạn sử dụng** |
| ![Ngân sách](docs/images/purchase-budget.png) | ![Tồn theo lô](docs/images/stock-by-lot-expiry.png) |
| **Khuyến mãi hàng cận hạn tự động** | **Bán hàng áp giá khuyến mãi** |
| ![Khuyến mãi cận hạn](docs/images/expiry-promotion.png) | ![Bán hàng](docs/images/store-sale-promo-price.png) |

## 🧩 Phân tích GAP

| Nhóm nghiệp vụ | Odoo tiêu chuẩn | Mức độ | Giải pháp trong repo |
| --- | --- | --- | --- |
| Nhu cầu cửa hàng & PR nội bộ | Chưa có luồng PR Store → Merchandise | Partial Fit | PR của `store_management` + `merchandise_management` |
| Phê duyệt PR | Cấu hình được nhưng chưa khớp quy trình | Partial Fit | Trạng thái nhiều cấp & kiểm tra theo vai trò |
| Tạo PO | Hỗ trợ đầy đủ | Full Fit | Purchase tiêu chuẩn, PO liên kết PR gốc |
| Tồn kho & điều chuyển | Hỗ trợ đầy đủ | Full / Config Fit | Cấu hình kho, tuyến trung chuyển, phiếu kho |
| Quyết định nguồn đáp ứng | Chưa có theo từng dòng PR | Gap / Partial | Tab *Phương án xử lý* trên PR |
| Nhận hàng & kiểm tra chất lượng | Chỉ nhận hàng, chưa có QC | Partial Fit | QC của `warehouse_management` |
| Xử lý sai lệch | Chỉ điều chỉnh tồn / trả hàng | Gap | Báo cáo sai lệch liên kết PR/PO bù, thu hồi |
| Khuyến mãi | Có nhưng chưa theo cửa hàng/lô | Partial Fit | KM theo cửa hàng & scheduler hàng cận hạn |
| Theo dõi trạng thái | Theo từng chứng từ | Partial Fit | Trạng thái luồng Kho tổng + dashboard Merchandise |

## 🚀 Cài đặt

**Yêu cầu:** [Docker Desktop](https://www.docker.com/products/docker-desktop/) (Docker Compose v2).

```bash
git clone https://github.com/HoangPhucTony/Mis_Purchasing_OdooProject.git
cd Mis_Purchasing_OdooProject
docker compose up -d --build
```

1. Mở <http://localhost:7860>, tạo database (master password local: `admin`).
2. Vào **Ứng dụng**, tìm **Store Management** và cài đặt.
3. Vào **Thiết lập → Người dùng**, cấp quyền cho từng người dùng ở mục *Bộ phận Merchandise*, *Cửa hàng* hoặc *Kho*.

Cập nhật module sau khi sửa code:

```bash
docker compose run --rm odoo odoo -u merchandise_management,warehouse_management,store_management -d <ten_db> --stop-after-init
```

## ☁️ Triển khai

Mỗi lần push lên `main`, [GitHub Actions](.github/workflows/sync_to_hf.yml) đồng bộ mã nguồn sang Hugging Face Space;
Space tự build lại từ [`Dockerfile`](Dockerfile).

Khai báo biến môi trường trong *Space → Settings → Variables and secrets* (mẫu: [`.env.example`](.env.example)):

| Biến | Ý nghĩa |
| --- | --- |
| `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD` | Kết nối PostgreSQL bên ngoài |
| `ODOO_MASTER_PASSWORD` | Mật khẩu Database Manager — **luôn đặt giá trị mạnh** |
| `DB_FILTER` | Regex giới hạn database hiển thị |
| `MAX_CRON_THREADS` | Số luồng scheduler (mặc định `1`, `0` để tắt) |

Secret của repository GitHub: `HF_DEPLOY_ODOO` — Hugging Face access token có quyền ghi.

## 📁 Cấu trúc thư mục

```text
.
├── custom_addons/
│   ├── merchandise_management/   # PR, ngân sách, khuyến mãi, sai lệch, dashboard
│   ├── store_management/         # cửa hàng, min/max, Kho tổng→Cửa hàng, bán hàng
│   └── warehouse_management/     # QC nhận hàng, trả hàng
├── docs/
│   ├── images/                   # ảnh minh họa README
│   └── SODOBPMN.pdf              # sơ đồ BPMN
├── .github/workflows/            # đồng bộ Hugging Face Spaces
├── Dockerfile                    # image chạy trên Space
├── docker-compose.yml            # môi trường phát triển local (Odoo + PostgreSQL)
└── .env.example
```

## 📊 Đánh giá & hướng phát triển

Giải pháp đáp ứng tốt các yêu cầu cốt lõi: PR là chứng từ trung tâm với quy trình trạng thái rõ ràng, Merchandise
giữ vai trò điều phối, xử lý rẽ nhánh theo hai nguồn đáp ứng (kho tổng hoặc NCC). Các nghiệp vụ nhận hàng, QC, xử lý
sai lệch, PR bù, thu hồi và khuyến mãi đã được mô phỏng ở mức prototype.

**Hướng phát triển**

- [ ] Kiểm thử với dữ liệu lớn, sát thực tế (nhiều cửa hàng, SKU, tình huống ngoại lệ đồng thời); thí điểm phạm vi nhỏ.
- [ ] Đồng bộ dữ liệu bán hàng POS để bổ sung hàng theo tốc độ tiêu thụ thực.
- [ ] Liên kết mua hàng, điều chuyển, khuyến mãi với kế toán (chi phí, công nợ, ngân sách).
- [ ] KPI nhà cung cấp: giao đúng hạn, tỷ lệ giao đủ, tỷ lệ hàng lỗi và sai lệch.
- [ ] Dashboard riêng cho cửa hàng, Merchandise, kho tổng và cấp quản lý.
- [ ] Phân quyền chi tiết và lưu vết thao tác giữa các bộ phận.

## 👨‍💻 Nhóm thực hiện

**Nhóm 1** — Viện Công nghệ Thông tin và Điện, Điện tử, Trường Đại học Giao thông Vận tải TP.HCM (UTH)

| Thành viên |
| --- |
| Huỳnh Thái Bảo Ngân |
| Nguyễn Thị Mỹ Trinh |
| Phạm Như Trâm |
| Hoàng Quốc Khánh |
| Đỗ Hoàng Phúc |
| Trần Nguyên Thịnh |

**Giảng viên hướng dẫn:** ThS. Lê Hữu Thanh Tùng

## 📄 Giấy phép

Phát hành theo giấy phép [GNU LGPL-3.0](LICENSE), tương thích với Odoo Community.

<div align="center"><sub>Thực hiện bởi Nhóm 1 · UTH · 2026</sub></div>
