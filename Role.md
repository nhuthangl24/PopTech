# OpenSearch Security — Roles & Tenants

> Tài liệu giải thích từ đầu về phân quyền user, role, và tenants trên OpenSearch — kèm lý thuyết, ví dụ thực tế, và lệnh cấu hình.

---

## 1. Tổng quan — 3 lớp bảo mật

Khi một user đăng nhập vào OpenSearch Dashboards, hệ thống kiểm tra 3 lớp theo thứ tự:

```
[User đăng nhập] → Authentication (mày là ai?) 
                 → Authorization (mày được làm gì?) 
                 → Tenant (mày thấy dashboard nào?)
```

### Lớp 1 — Authentication: Mày là ai?

Xác minh danh tính. OpenSearch hỗ trợ 3 cách:

| Cách | Mô tả | Khi nào dùng |
|---|---|---|
| **Internal user** | Tạo username/password thẳng trên OpenSearch | Hệ thống nhỏ, không có AD/LDAP |
| **LDAP/Active Directory** | Dùng tài khoản Windows domain của công ty | Doanh nghiệp đã có AD |
| **SAML/SSO** | Đăng nhập một lần qua Google, Okta, Azure AD | Muốn SSO nhiều hệ thống |

→ Hệ thống của bạn đang dùng **Internal user**.

---

### Lớp 2 — Authorization: Mày được làm gì?

Sau khi biết mày là ai, OpenSearch kiểm tra mày có quyền gì. Cấu trúc:

```
User ──map──→ Role ──chứa──→ Permissions
```

- **User** — tài khoản đăng nhập
- **Role** — tập hợp quyền. Một user có thể có nhiều role. Nhiều user có thể dùng chung một role
- **Permission** — quyền cụ thể: đọc index nào, ghi index nào, xóa được không, search được không

Ví dụ thực tế:
```
analyst  ──→ security-readonly  ──→ chỉ GET/search index security-*
ubuntu-viewer ──→ ubuntu-readonly ──→ chỉ GET/search index security-ubuntu-monitor-*
admin    ──→ all_access         ──→ làm được mọi thứ
```

---

### Lớp 3 — Tenant: Mày thấy dashboard nào?

Tenant là **không gian làm việc riêng** trên OpenSearch Dashboards. Mỗi tenant có:
- Index pattern riêng
- Dashboard riêng
- Visualization riêng
- Saved search riêng

OpenSearch có 2 tenant mặc định sẵn:
- **Global** — tất cả user đều thấy, dùng chung
- **Private** — chỉ mình user đó thấy, không chia sẻ được

Custom tenant — bạn tự tạo, ví dụ `ubuntu-team`, `winlogbeat-team`, chỉ user được gắn role mới vào được.

---

## 2. Permissions — quyền cụ thể là gì?

### Cluster permissions — quyền trên toàn cluster

| Permission | Nghĩa |
|---|---|
| `cluster_composite_ops_ro` | Đọc metadata cluster, dùng cho user chỉ đọc |
| `cluster_composite_ops` | Đọc + một số write operation |
| `cluster:monitor/*` | Xem health, stats của cluster |

### Index permissions — quyền trên index cụ thể

| Permission | Nghĩa |
|---|---|
| `read` | Đọc document |
| `search` | Search document |
| `get` | Lấy document theo ID |
| `write` | Ghi document |
| `delete` | Xóa document |
| `indices:data/read*` | Toàn bộ quyền đọc |
| `indices:admin/mappings/get` | Xem mapping của index (cần thiết để Dashboards hoạt động) |

### Tenant permissions — quyền trên tenant

| Permission | Nghĩa |
|---|---|
| `kibana_all_read` | Chỉ xem dashboard, không sửa được |
| `kibana_all_write` | Xem và sửa dashboard, tạo visualization |

---

## 3. Lý do phải có đủ 3 thứ: User + Role + Role Mapping

Nhiều người nhầm tưởng tạo user xong là xong. Thực ra phải đủ 3 bước:

**Bước 1 — Tạo user**: Chỉ tạo tài khoản, chưa có quyền gì cả.

**Bước 2 — Tạo role**: Định nghĩa quyền — được đọc index nào, thấy tenant nào. Nhưng chưa gắn cho ai.

**Bước 3 — Role mapping**: Gắn user vào role. Bước này mới thực sự cấp quyền cho user.

```
Thiếu bước 3 → user đăng nhập được nhưng không thấy gì cả (403 Forbidden)
```

---

## 4. Cấu hình thực tế cho hệ thống này

### Mục tiêu

| User | Thấy index nào | Quyền | Tenant |
|---|---|---|---|
| `admin` | Tất cả | Mọi thứ | Global |
| `analyst` | `security-*` | Chỉ đọc | security-team |
| `ubuntu-viewer` | `security-ubuntu-monitor-*` | Chỉ đọc | ubuntu-team |
| `winlogbeat-viewer` | `security-winlogbeat-monitor-*` | Chỉ đọc | winlogbeat-team |

---

### Bước 1 — Tạo internal user

```json
PUT _plugins/_security/api/internalusers/analyst
{
  "password": "Analyst@123",
  "backend_roles": [],
  "attributes": {}
}
```

```json
PUT _plugins/_security/api/internalusers/ubuntu-viewer
{
  "password": "Ubuntu@123",
  "backend_roles": [],
  "attributes": {}
}
```

```json
PUT _plugins/_security/api/internalusers/winlogbeat-viewer
{
  "password": "Winlogbeat@123",
  "backend_roles": [],
  "attributes": {}
}
```

> `admin` đã có sẵn mặc định, không cần tạo.

---

### Bước 2 — Tạo tenant

```json
PUT _plugins/_security/api/tenants/security-team
{
  "description": "Tenant chung cho security team"
}
```

```json
PUT _plugins/_security/api/tenants/ubuntu-team
{
  "description": "Tenant rieng cho ubuntu team"
}
```

```json
PUT _plugins/_security/api/tenants/winlogbeat-team
{
  "description": "Tenant rieng cho winlogbeat team"
}
```

---

### Bước 3 — Tạo role

**Role analyst — đọc toàn bộ security index:**

```json
PUT _plugins/_security/api/roles/security-readonly
{
  "description": "Chi doc, khong xoa",
  "cluster_permissions": [
    "cluster_composite_ops_ro"
  ],
  "index_permissions": [{
    "index_patterns": ["security-*"],
    "allowed_actions": [
      "read",
      "search",
      "get",
      "indices:data/read*",
      "indices:admin/mappings/get"
    ]
  }],
  "tenant_permissions": [{
    "tenant_patterns": ["security-team"],
    "allowed_actions": ["kibana_all_read"]
  }]
}
```

Giải thích:
- `cluster_composite_ops_ro` — cho phép user đọc metadata cluster, cần thiết để Dashboards load được
- `index_patterns: ["security-*"]` — áp dụng cho tất cả index bắt đầu bằng `security-`
- `indices:admin/mappings/get` — cho phép đọc mapping, thiếu cái này Dashboards báo lỗi khi tạo index pattern
- `kibana_all_read` — chỉ xem dashboard, không tạo/sửa/xóa được

**Role ubuntu-viewer — chỉ đọc ubuntu index:**

```json
PUT _plugins/_security/api/roles/ubuntu-readonly
{
  "description": "Chi doc index ubuntu",
  "cluster_permissions": [
    "cluster_composite_ops_ro"
  ],
  "index_permissions": [{
    "index_patterns": ["security-ubuntu-monitor-*"],
    "allowed_actions": [
      "read",
      "search",
      "get",
      "indices:data/read*",
      "indices:admin/mappings/get"
    ]
  }],
  "tenant_permissions": [{
    "tenant_patterns": ["ubuntu-team"],
    "allowed_actions": ["kibana_all_read"]
  }]
}
```

**Role winlogbeat-viewer — chỉ đọc winlogbeat index:**

```json
PUT _plugins/_security/api/roles/winlogbeat-readonly
{
  "description": "Chi doc index winlogbeat",
  "cluster_permissions": [
    "cluster_composite_ops_ro"
  ],
  "index_permissions": [{
    "index_patterns": ["security-winlogbeat-monitor-*"],
    "allowed_actions": [
      "read",
      "search",
      "get",
      "indices:data/read*",
      "indices:admin/mappings/get"
    ]
  }],
  "tenant_permissions": [{
    "tenant_patterns": ["winlogbeat-team"],
    "allowed_actions": ["kibana_all_read"]
  }]
}
```

---

### Bước 4 — Role mapping (gắn user vào role)

```json
PUT _plugins/_security/api/rolesmapping/security-readonly
{
  "users": ["analyst"]
}
```

```json
PUT _plugins/_security/api/rolesmapping/ubuntu-readonly
{
  "users": ["ubuntu-viewer"]
}
```

```json
PUT _plugins/_security/api/rolesmapping/winlogbeat-readonly
{
  "users": ["winlogbeat-viewer"]
}
```

> Ngoài ra mỗi user cần được map vào role `kibana_user` để đăng nhập được Dashboards:

```json
PUT _plugins/_security/api/rolesmapping/kibana_user
{
  "users": ["analyst", "ubuntu-viewer", "winlogbeat-viewer"]
}
```

---

### Bước 5 — Verify

```json
# Kiểm tra user đã tạo chưa
GET _plugins/_security/api/internalusers/analyst

# Kiểm tra role đã đúng chưa
GET _plugins/_security/api/roles/security-readonly

# Kiểm tra mapping đã đúng chưa
GET _plugins/_security/api/rolesmapping/security-readonly

# Kiểm tra tenant đã tạo chưa
GET _plugins/_security/api/tenants/security-team
```

---

## 5. Lỗi hay gặp

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| User đăng nhập được nhưng không thấy index nào | Thiếu role mapping, hoặc index_patterns sai | Kiểm tra rolesmapping, thêm `kibana_user` role |
| Dashboards báo lỗi khi tạo index pattern | Thiếu permission `indices:admin/mappings/get` | Thêm vào allowed_actions của role |
| User thấy dashboard nhưng không thấy data | Thiếu cluster_permissions | Thêm `cluster_composite_ops_ro` |
| User vào Dashboards bị redirect ra ngoài | Thiếu role `kibana_user` | Map user vào `kibana_user` role |
| Muốn thêm user vào role đã có mapping | Dùng PATCH thay vì PUT (PUT ghi đè toàn bộ) | `PATCH _plugins/_security/api/rolesmapping/<role>` |

---

## 6. Lệnh quản lý thường dùng

```bash
# Xem tất cả user
GET _plugins/_security/api/internalusers

# Xem tất cả role
GET _plugins/_security/api/roles

# Xem tất cả role mapping
GET _plugins/_security/api/rolesmapping

# Xem tất cả tenant
GET _plugins/_security/api/tenants

# Xóa user
DELETE _plugins/_security/api/internalusers/analyst

# Xóa role
DELETE _plugins/_security/api/roles/security-readonly

# Đổi password user
PUT _plugins/_security/api/internalusers/analyst
{
  "password": "NewPassword@456"
}

# Thêm user vào role mapping có sẵn (không ghi đè)
PATCH _plugins/_security/api/rolesmapping/security-readonly
[{
  "op": "add",
  "path": "/users/-",
  "value": "new-analyst"
}]
```

---

## 7. Sơ đồ tổng quan

```
                    ┌─────────────┐
                    │   OpenSearch │
                    │  Dashboards  │
                    └──────┬──────┘
                           │ đăng nhập
                    ┌──────▼──────┐
                    │    User      │
                    │  (analyst)   │
                    └──────┬──────┘
                           │ map tới
                    ┌──────▼──────┐
                    │    Role      │
                    │security-only │
                    └──────┬──────┘
              ┌────────────┼────────────┐
              │            │            │
    ┌─────────▼──┐  ┌──────▼─────┐  ┌──▼──────────┐
    │  Cluster   │  │   Index    │  │   Tenant    │
    │ Permission │  │ Permission │  │ Permission  │
    │(read meta) │  │(read sec-*)│  │(read-only   │
    │            │  │            │  │ security-   │
    │            │  │            │  │ team)       │
    └────────────┘  └────────────┘  └─────────────┘
```

---

*Tài liệu được tạo ngày 26/05/2026*
*Docs chính thức: https://docs.opensearch.org/latest/security/access-control/users-roles/*
