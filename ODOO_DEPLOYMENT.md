# Hướng Dẫn Deploy Odoo 18 - erp.trecome.vn

## Thông tin hệ thống

| Mục | Giá trị |
|-----|---------|
| Domain | https://erp.trecome.vn |
| Server | ubuntu@150.230.108.164 |
| SSH Key | `C:\Users\Admin\.ssh\trecome\trecome.key` |
| GitHub Repo | https://github.com/trandinhnamuet/trecome-odoo |
| Odoo Version | 18.0 Community |
| Database | PostgreSQL |
| Odoo Port | 8069 (proxied qua Nginx) |

---

## 1. Lấy code về máy local (lần đầu)

```bash
# Clone từ GitHub của bạn về máy
git clone https://github.com/trandinhnamuet/trecome-odoo.git d:\Project\trecome-temp-server
cd d:\Project\trecome-temp-server

# Remote đã được cấu hình sẵn:
# origin           → GitHub của bạn (push/pull)
# odoo-upstream    → GitHub Odoo chính thức (để cập nhật bản mới)
```

---

## 2. Workflow hàng ngày: Sửa code → Đẩy lên GitHub

```bash
cd d:\Project\trecome-temp-server

# 1. Tạo/sửa module trong thư mục custom_addons/
# 2. Stage và commit thay đổi
git add .
git commit -m "feat: mô tả thay đổi"

# 3. Push lên GitHub
git push origin main
```

---

## 3. Deploy lên server (sau khi push GitHub)

```bash
# SSH vào server
ssh -i "C:\Users\Admin\.ssh\trecome\trecome.key" ubuntu@150.230.108.164

# Pull code mới nhất từ GitHub
cd /opt/odoo/odoo18
sudo -u odoo git pull origin main

# Restart Odoo để áp dụng thay đổi
sudo systemctl restart odoo

# Kiểm tra trạng thái
sudo systemctl status odoo
```

### Nếu thêm module mới cần update database:

```bash
ssh -i "C:\Users\Admin\.ssh\trecome\trecome.key" ubuntu@150.230.108.164

sudo systemctl stop odoo
sudo -u odoo /opt/odoo/venv/bin/python3 /opt/odoo/odoo18/odoo-bin \
  -c /etc/odoo/odoo.conf \
  -d <tên_database> \
  --update=<tên_module> \
  --stop-after-init
sudo systemctl start odoo
```

---

## 4. Cập nhật Odoo lên phiên bản mới hơn

```bash
cd d:\Project\trecome-temp-server

# Fetch code mới từ Odoo official
git fetch odoo-upstream 18.0

# Merge vào branch main
git merge odoo-upstream/18.0

# Giải quyết conflict nếu có, sau đó push
git push origin main

# Trên server: pull và restart
ssh -i "C:\Users\Admin\.ssh\trecome\trecome.key" ubuntu@150.230.108.164 "
  cd /opt/odoo/odoo18 && sudo -u odoo git pull origin main &&
  sudo -u odoo /opt/odoo/venv/bin/pip install -r requirements.txt &&
  sudo systemctl restart odoo
"
```

---

## 5. Cấu trúc thư mục trên server

```
/opt/odoo/
├── odoo18/          ← Source code (clone từ GitHub của bạn)
│   ├── addons/      ← Modules chuẩn của Odoo
│   └── odoo/        ← Core Odoo framework
├── custom_addons/   ← Modules tùy chỉnh của bạn
├── venv/            ← Python virtual environment
└── .cache/          ← Cache pip

/etc/odoo/odoo.conf  ← File cấu hình Odoo
/var/log/odoo/       ← Log files
```

---

## 6. Thêm custom module

```bash
# Trên máy local: tạo module trong custom_addons/
mkdir d:\Project\trecome-temp-server\custom_addons\my_module

# Tạo cấu trúc module cơ bản
# my_module/__init__.py
# my_module/__manifest__.py
# my_module/models/
# ...

# Push lên GitHub
git add custom_addons/
git commit -m "feat: add my_module"
git push origin main

# Trên server: pull và cập nhật
ssh -i "C:\Users\Admin\.ssh\trecome\trecome.key" ubuntu@150.230.108.164 "
  cd /opt/odoo/odoo18 && sudo -u odoo git pull &&
  sudo systemctl restart odoo
"
```

---

## 7. Trỏ tên miền

Để trỏ domain `erp.trecome.vn` về server:

1. Vào DNS manager của trecome.vn
2. Tạo record **A**:
   - Name: `erp`
   - Value: `150.230.108.164`
   - TTL: 300 (hoặc tùy chọn)
3. Đợi DNS propagate (5-30 phút)
4. Kiểm tra: `nslookup erp.trecome.vn` → phải trả về `150.230.108.164`

> SSL cert tự động cấp và renew qua Let's Encrypt (certbot). Hết hạn: 2026-09-07, tự gia hạn trước 30 ngày.

---

## 8. Các lệnh quản lý thường dùng

```bash
# SSH vào server
ssh -i "C:\Users\Admin\.ssh\trecome\trecome.key" ubuntu@150.230.108.164

# Xem log Odoo realtime
sudo journalctl -u odoo -f

# Restart/Stop/Start Odoo
sudo systemctl restart odoo
sudo systemctl stop odoo
sudo systemctl start odoo

# Xem trạng thái
sudo systemctl status odoo

# Xem log file
sudo tail -100 /var/log/odoo/odoo.log

# Backup database
sudo -u odoo /opt/odoo/venv/bin/python3 /opt/odoo/odoo18/odoo-bin \
  -c /etc/odoo/odoo.conf \
  --backup --database=<db_name> --backup-format=zip \
  --backup-dest=/opt/odoo/backups/
```

---

## 9. Thông tin đăng nhập

| Mục | Giá trị |
|-----|---------|
| URL | https://erp.trecome.vn |
| Admin password | `Trecome@2024!` (master password để tạo database) |
| Database | Tạo mới khi truy cập lần đầu tại https://erp.trecome.vn/web/database/manager |

> **Lưu ý bảo mật**: Đổi `admin_passwd` trong `/etc/odoo/odoo.conf` sau khi deploy xong.
