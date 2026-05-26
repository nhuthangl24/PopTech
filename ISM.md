# OpenSearch ISM Policy — Hướng dẫn toàn diện

> Tài liệu này tổng hợp lý thuyết, cách cấu hình, và các bước thực hiện ISM Policy trên OpenSearch cho hệ thống security log monitoring.

---

## 1. ISM là gì?

**ISM (Index State Management)** là tính năng của OpenSearch cho phép tự động hóa vòng đời của index theo các rule định nghĩa sẵn — thay vì phải thao tác thủ công từng lệnh.

Không có ISM, bạn phải tự nhớ: *"index này 7 ngày rồi, vào xóa đi"*, *"index này to quá rồi, tạo cái mới đi"*. ISM làm hết việc đó tự động.

OpenSearch chạy một **ISM job nền mỗi 5 phút**, quét tất cả index đang được quản lý, kiểm tra điều kiện transition, nếu thỏa thì chuyển state và chạy action tương ứng.

---

## 2. Các khái niệm cốt lõi

### Policy
Bản kế hoạch vòng đời của index. Viết một lần, gắn vào index, OpenSearch tự thực thi.

### State
Trạng thái mà index đang ở. Ví dụ: `hot`, `warm`, `live`, `delete`. Index đi từng state theo thứ tự định nghĩa trong policy.

### Action
Việc cần làm khi index vào một state. Một state có thể có nhiều action, chạy lần lượt theo thứ tự.

### Transition
Điều kiện để chuyển sang state tiếp theo. Không thỏa điều kiện thì index đứng yên ở state hiện tại.

---

## 3. Các action thường dùng

| Action | Mục đích | Khi nào dùng |
|---|---|---|
| `rollover` | Tạo index mới khi đủ điều kiện | Index ghi liên tục, cần chia nhỏ |
| `force_merge` | Gộp segments lại, giảm disk, tăng tốc đọc | Sau khi index không còn ghi nữa |
| `replica_count` | Thay đổi số replica | Giảm về 0 ở warm/cold để tiết kiệm disk |
| `read_only` | Khóa không cho ghi | Cold state |
| `snapshot` | Backup ra S3 hoặc repo khác | Trước khi xóa nếu cần lưu trữ dài hạn |
| `delete` | Xóa index vĩnh viễn | Cuối vòng đời |

---

## 4. Hai kiểu index và cách xử lý khác nhau

### Date-based index (kiểu này của hệ thống)
Index được tạo tự động mỗi ngày theo tên ngày tháng, ví dụ:
```
security-ubuntu-monitor-2026.05.26
security-winlogbeat-monitor-2026.05.26
```
Logstash/Filebeat/Winlogbeat tự tạo index mới mỗi ngày — **không cần rollover**.
Policy chỉ cần: live → warm (merge) → delete.

### Rollover-based index
Index tạo thủ công một lần với alias, rollover khi đủ điều kiện:
```
app-logs-000001 → app-logs-000002 → ...
```
Yêu cầu bắt buộc phải có **write alias**. App ghi vào alias, OpenSearch tự điều hướng.

---

## 5. Policy đã cấu hình cho hệ thống này

### Kịch bản
- Index date-based, tạo tự động mỗi ngày
- Giữ data 7 ngày rồi xóa
- Merge sau 1 ngày để tối ưu disk và tốc độ đọc

### Sơ đồ vòng đời
```
[Tạo index] → live (chờ 1 ngày) → warm (merge + giảm replica) → delete (sau 7 ngày)
```

### Giải thích chi tiết từng dòng lệnh

```json
PUT _plugins/_ism/policies/ubuntu-monitor-policy
```
- `PUT` = tạo mới hoặc ghi đè nếu đã tồn tại
- `_plugins/_ism/policies/` = endpoint API của ISM trên OpenSearch
- `ubuntu-monitor-policy` = tên policy, đặt gì cũng được, dùng để gắn vào index sau

```json
"description": "Merge sau 1 ngay, xoa sau 7 ngay"
```
- Chỉ là ghi chú mô tả, không ảnh hưởng logic

```json
"default_state": "live"
```
- Index mới được gắn policy này sẽ bắt đầu ở state `live`
- Nếu không khai báo, ISM không biết index phải bắt đầu từ đâu → lỗi

```json
"states": [ ... ]
```
- Mảng chứa toàn bộ các state của policy, khai báo theo thứ tự vòng đời

---

**State `live`:**
```json
{
  "name": "live",
  "actions": [],
  "transitions": [{
    "state_name": "warm",
    "conditions": { "min_index_age": "1d" }
  }]
}
```
- `"actions": []` = không làm gì cả khi ở state này, chỉ chờ
- Tại sao không làm gì? Vì index đang trong ngày hôm đó, Winlogbeat/Filebeat vẫn đang ghi vào — không được đụng vào
- `"min_index_age": "1d"` = sau 1 ngày kể từ lúc index được **tạo ra**, chuyển sang state warm
- Lưu ý: thời gian tính từ lúc **tạo index**, không phải lúc gắn policy

---

**State `warm`:**
```json
{
  "name": "warm",
  "actions": [
    { "replica_count": { "number_of_replicas": 0 } },
    { "force_merge": { "max_num_segments": 1 } }
  ],
  "transitions": [{
    "state_name": "delete",
    "conditions": { "min_index_age": "7d" }
  }]
}
```
- Index đã qua ngày hôm đó, không còn ghi nữa → an toàn để tối ưu
- **Action 1 — `replica_count: 0`**: Giảm số replica xuống 0. Replica là bản sao của index để đảm bảo tính sẵn sàng. Index cũ không cần ghi nữa, ít đọc → không cần replica → tiết kiệm disk đáng kể (ví dụ index 5GB, replica 1 → tiết kiệm 5GB)
- **Action 2 — `force_merge: 1`**: Lucene (engine bên dưới OpenSearch) lưu dữ liệu thành nhiều file nhỏ gọi là segment. Càng nhiều segment → truy vấn càng chậm vì phải quét nhiều file. `force_merge` gộp tất cả lại thành 1 segment duy nhất. Lý do làm sau khi không ghi nữa: nếu đang ghi mà force_merge thì vừa gộp xong lại có segment mới tạo ra → vô nghĩa và tốn I/O
- 2 action chạy **lần lượt**: giảm replica trước, rồi mới merge
- `"min_index_age": "7d"` = sau 7 ngày kể từ lúc tạo index → chuyển sang delete

---

**State `delete`:**
```json
{
  "name": "delete",
  "actions": [{ "delete": {} }],
  "transitions": []
}
```
- `"delete": {}` = xóa index vĩnh viễn, không rollback được
- `"transitions": []` = không có điều kiện chuyển tiếp → đây là điểm cuối của vòng đời

---

### Policy ubuntu (đầy đủ)

```json
PUT _plugins/_ism/policies/ubuntu-monitor-policy
{
  "policy": {
    "description": "Merge sau 1 ngay, xoa sau 7 ngay",
    "default_state": "live",
    "states": [
      {
        "name": "live",
        "actions": [],
        "transitions": [{
          "state_name": "warm",
          "conditions": { "min_index_age": "1d" }
        }]
      },
      {
        "name": "warm",
        "actions": [
          { "replica_count": { "number_of_replicas": 0 } },
          { "force_merge": { "max_num_segments": 1 } }
        ],
        "transitions": [{
          "state_name": "delete",
          "conditions": { "min_index_age": "7d" }
        }]
      },
      {
        "name": "delete",
        "actions": [{ "delete": {} }],
        "transitions": []
      }
    ]
  }
}
```

### Policy winlogbeat (đầy đủ)

```json
PUT _plugins/_ism/policies/winlogbeat-monitor-policy
{
  "policy": {
    "description": "Merge sau 1 ngay, xoa sau 7 ngay",
    "default_state": "live",
    "states": [
      {
        "name": "live",
        "actions": [],
        "transitions": [{
          "state_name": "warm",
          "conditions": { "min_index_age": "1d" }
        }]
      },
      {
        "name": "warm",
        "actions": [
          { "replica_count": { "number_of_replicas": 0 } },
          { "force_merge": { "max_num_segments": 1 } }
        ],
        "transitions": [{
          "state_name": "delete",
          "conditions": { "min_index_age": "7d" }
        }]
      },
      {
        "name": "delete",
        "actions": [{ "delete": {} }],
        "transitions": []
      }
    ]
  }
}
```

---

## 6. Các bước đã thực hiện

### Bước 1 — Kiểm tra index tồn tại
```
GET _cat/indices/security-ubuntu-monitor-*?v
GET _cat/indices/security-winlogbeat-monitor-*?v
```
Mục đích: xác nhận tên index chính xác trước khi làm bất cứ thứ gì.

### Bước 2 — Tạo policy
```
PUT _plugins/_ism/policies/ubuntu-monitor-policy     { ... }
PUT _plugins/_ism/policies/winlogbeat-monitor-policy { ... }
```

### Bước 3 — Gắn policy vào index

Với index **chưa có policy** dùng `add`:
```json
POST _plugins/_ism/add/security-ubuntu-monitor-*
{ "policy_id": "ubuntu-monitor-policy" }
```

Với index **đã có policy cũ** dùng `change_policy`:
```json
POST _plugins/_ism/change_policy/security-winlogbeat-monitor-*
{ "policy_id": "winlogbeat-monitor-policy" }
```

> Tại sao có 2 API khác nhau? `add` chỉ dùng được khi index chưa có policy nào. Nếu index đã có policy (dù là policy cũ hay sai) thì phải dùng `change_policy` để ghi đè.

### Bước 4 — Verify
```
GET _plugins/_ism/explain/security-ubuntu-monitor-*
GET _plugins/_ism/explain/security-winlogbeat-monitor-*
```

Kết quả đúng:
```json
{
  "policy_id": "ubuntu-monitor-policy",
  "enabled": true,
  "state": { "name": "live" },
  "step_status": "completed"
}
```

---

## 7. Kết quả hiện tại

| Index pattern | Policy | Trạng thái |
|---|---|---|
| `security-ubuntu-monitor-*` | `ubuntu-monitor-policy` | enabled ✓ |
| `security-winlogbeat-monitor-*` | `winlogbeat-monitor-policy` | enabled ✓ |

> ISM job chạy mỗi 5 phút. Sau lần chạy đầu tiên sẽ thấy `state` và `step_status` trong kết quả explain.

---

## 8. Việc còn lại (khuyến nghị)

### Tạo index template
Hiện tại mỗi ngày Winlogbeat/Filebeat tạo ra index mới, bạn phải chạy `add` tay. Index template giải quyết việc đó: mọi index mới khớp pattern sẽ tự động có policy luôn.

```json
PUT _index_template/ubuntu-monitor-template
{
  "index_patterns": ["security-ubuntu-monitor-*"],
  "template": {
    "settings": {
      "plugins.index_state_management.policy_id": "ubuntu-monitor-policy",
      "number_of_shards": 1,
      "number_of_replicas": 1
    }
  }
}
```

```json
PUT _index_template/winlogbeat-monitor-template
{
  "index_patterns": ["security-winlogbeat-monitor-*"],
  "template": {
    "settings": {
      "plugins.index_state_management.policy_id": "winlogbeat-monitor-policy",
      "number_of_shards": 1,
      "number_of_replicas": 1
    }
  }
}
```

---

## 9. Xử lý lỗi thường gặp

| Lỗi | Nguyên nhân | Cách xử lý |
|---|---|---|
| `This index already has a policy` | Dùng `add` cho index đã có policy | Dùng `change_policy` thay thế |
| `This index is not being managed` | Dùng `change_policy` cho index chưa có policy | Dùng `add` thay thế |
| `step_status: failed` | Action bị lỗi | Chạy `POST _plugins/_ism/retry/<index>` |
| Index bị xóa sớm hơn dự kiến | `min_index_age` tính từ lúc tạo index, không phải lúc gắn policy | Kiểm tra ngày tạo index trước khi gắn |
| Policy không tự gắn vào index mới | Chưa tạo index template | Tạo template theo mục 8 |

---

## 10. Các lệnh quản lý thường dùng

```bash
# Xem tất cả policy đang có
GET _plugins/_ism/policies

# Xem chi tiết một policy
GET _plugins/_ism/policies/ubuntu-monitor-policy

# Xem trạng thái tất cả index được quản lý
GET _plugins/_ism/explain

# Retry index bị failed
POST _plugins/_ism/retry/security-ubuntu-monitor-2026.05.26

# Xóa policy khỏi index (index vẫn còn, chỉ bỏ quản lý)
POST _plugins/_ism/remove/security-ubuntu-monitor-*

# Xóa policy hoàn toàn
DELETE _plugins/_ism/policies/ubuntu-monitor-policy
```

---

*Tài liệu được tạo ngày 26/05/2026*
