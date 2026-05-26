# Tài liệu Filebeat, Winlogbeat, Logstash

## 1. Bức tranh tổng quan

Trong một hệ thống giám sát log, thường có luồng xử lý như sau:

```text
Server / Application / Windows Machine
        ↓
Filebeat / Winlogbeat
        ↓
Logstash
        ↓
OpenSearch / Elasticsearch
        ↓
Dashboard / Alert / Search
```

Hiểu đơn giản:

* **Filebeat**: đọc log từ file, ví dụ `/var/log/nginx/access.log`, `/var/log/auth.log`, log application.
* **Winlogbeat**: đọc Windows Event Log, ví dụ Security, System, Application.
* **Logstash**: nhận log, parse log, đổi field, lọc dữ liệu, chuẩn hóa dữ liệu, rồi đẩy đi nơi khác.
* **OpenSearch / Elasticsearch**: nơi lưu trữ và tìm kiếm log.
* **Dashboard**: nơi xem log, filter, tạo biểu đồ, tạo cảnh báo.

---

## 2. Filebeat là gì?

**Filebeat** là một lightweight shipper dùng để đọc log từ file và gửi log đi.

Nói dễ hiểu: Filebeat giống như một “người giao hàng” chuyên đi lấy log từ các file trên server rồi chuyển đến Logstash hoặc Elasticsearch/OpenSearch.

### Filebeat thường dùng khi nào?

Dùng Filebeat khi muốn thu thập log từ:

* Application log
* Nginx log
* Apache log
* Docker log
* Linux system log
* Custom log file
* API service log
* Error log

Ví dụ:

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
/var/log/auth.log
/opt/app/logs/app.log
```

---

## 3. Winlogbeat là gì?

**Winlogbeat** là một lightweight shipper dùng để đọc log từ Windows Event Viewer.

Nói dễ hiểu: Winlogbeat giống Filebeat nhưng chuyên cho Windows Event Log.

### Winlogbeat thường dùng để lấy log nào?

Các log phổ biến:

* **Security**: đăng nhập, đăng xuất, tạo user, đổi quyền, failed login.
* **System**: service start/stop, lỗi hệ thống, driver, reboot.
* **Application**: lỗi ứng dụng, crash, service app.
* **PowerShell**: script execution, command activity.
* **Sysmon**: process creation, network connection, registry change.

Ví dụ các Event ID quan trọng:

| Event ID | Ý nghĩa                      |
| -------: | ---------------------------- |
|     4624 | Đăng nhập thành công         |
|     4625 | Đăng nhập thất bại           |
|     4634 | Đăng xuất                    |
|     4672 | Đăng nhập với quyền đặc biệt |
|     4688 | Process được tạo             |
|     4720 | User account được tạo        |
|     4722 | User account được enable     |
|     4726 | User account bị xóa          |
|     4740 | Account bị lock              |
|     7045 | Service mới được cài         |

---

## 4. Logstash là gì?

**Logstash** là công cụ xử lý log ở giữa.

Nói dễ hiểu: nếu Filebeat/Winlogbeat là người giao log, thì Logstash là người “biên tập log”. Nó nhận log thô, sau đó:

* Parse nội dung log
* Tách field
* Đổi tên field
* Xóa field không cần thiết
* Thêm field mới
* Chuẩn hóa timestamp
* Gắn tag
* Gửi log sang OpenSearch/Elasticsearch

### Cấu trúc pipeline Logstash

Một file config Logstash thường có 3 phần:

```conf
input {
  # Nhận dữ liệu từ đâu?
}

filter {
  # Xử lý dữ liệu thế nào?
}

output {
  # Gửi dữ liệu đi đâu?
}
```

Ví dụ dễ hiểu:

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  mutate {
    add_field => { "source_type" => "server_log" }
  }
}

output {
  stdout {
    codec => rubydebug
  }
}
```

Ý nghĩa:

* `input`: Logstash nhận log từ Beats qua port `5044`.
* `filter`: thêm field `source_type`.
* `output`: in log ra màn hình để debug.

---

# 5. Luồng hoạt động chi tiết

## 5.1. Filebeat → Logstash

```text
File log trên Linux server
        ↓
Filebeat đọc file log
        ↓
Filebeat gửi event sang Logstash port 5044
        ↓
Logstash parse / filter / enrich
        ↓
OpenSearch lưu dữ liệu
```

## 5.2. Winlogbeat → Logstash

```text
Windows Event Viewer
        ↓
Winlogbeat đọc Security/System/Application log
        ↓
Winlogbeat gửi event sang Logstash port 5044
        ↓
Logstash lọc Event ID, đổi field, thêm thông tin
        ↓
OpenSearch lưu dữ liệu
```

---

# 6. Filebeat cấu hình cơ bản

File cấu hình chính thường là:

```text
/etc/filebeat/filebeat.yml
```

Trên Windows có thể là:

```text
C:\Program Files\Filebeat\filebeat.yml
```

## 6.1. Ví dụ filebeat.yml cơ bản

```yaml
filebeat.inputs:
  - type: filestream
    id: app-log
    enabled: true
    paths:
      - /var/log/app/*.log

output.logstash:
  hosts: ["192.168.1.10:5044"]
```

Giải thích:

| Dòng               | Ý nghĩa                                                                      |
| ------------------ | ---------------------------------------------------------------------------- |
| `filebeat.inputs`  | Khai báo nguồn log Filebeat cần đọc                                          |
| `type: filestream` | Kiểu input đọc file log mới, nên dùng thay cho `log` trong nhiều version mới |
| `id: app-log`      | Tên định danh cho input                                                      |
| `enabled: true`    | Bật input này                                                                |
| `paths`            | Đường dẫn file log cần đọc                                                   |
| `output.logstash`  | Gửi log sang Logstash                                                        |
| `hosts`            | IP/domain và port của Logstash                                               |

---

## 6.2. Các biến hay dùng trong Filebeat

### `type`

Chỉ loại input.

Ví dụ:

```yaml
- type: filestream
```

Ý nghĩa: Filebeat sẽ đọc log từ file.

### `id`

Tên định danh input.

Ví dụ:

```yaml
id: nginx-access-log
```

Nên đặt rõ nghĩa để dễ debug.

### `enabled`

Bật hoặc tắt input.

```yaml
enabled: true
```

Nếu để `false`, Filebeat sẽ không đọc input đó.

### `paths`

Danh sách đường dẫn file log cần đọc.

```yaml
paths:
  - /var/log/nginx/access.log
  - /var/log/nginx/error.log
```

Có thể dùng wildcard:

```yaml
paths:
  - /var/log/nginx/*.log
```

### `fields`

Thêm field custom vào event.

```yaml
fields:
  environment: production
  service: nginx
```

Kết quả log sẽ có thêm:

```text
environment = production
service = nginx
```

### `fields_under_root`

Quyết định field custom nằm ở root hay nằm trong object `fields`.

```yaml
fields_under_root: true
```

Nếu `true`:

```json
{
  "environment": "production"
}
```

Nếu `false`:

```json
{
  "fields": {
    "environment": "production"
  }
}
```

Thực tế nên dùng:

```yaml
fields_under_root: true
```

Vì dễ filter hơn.

### `exclude_lines`

Bỏ qua những dòng log không cần đọc.

```yaml
exclude_lines: ['^DEBUG']
```

Ý nghĩa: bỏ qua dòng bắt đầu bằng `DEBUG`.

### `include_lines`

Chỉ lấy những dòng khớp điều kiện.

```yaml
include_lines: ['ERROR', 'WARN']
```

Ý nghĩa: chỉ lấy log có chữ `ERROR` hoặc `WARN`.

### `ignore_older`

Bỏ qua file log quá cũ.

```yaml
ignore_older: 72h
```

Ý nghĩa: nếu file không thay đổi trong 72 giờ thì bỏ qua.

### `multiline`

Dùng khi một log event bị trải dài nhiều dòng.

Ví dụ Java stack trace:

```text
Exception in thread "main" java.lang.NullPointerException
    at com.app.Service.run(Service.java:10)
    at com.app.Main.main(Main.java:5)
```

Nếu không dùng multiline, Filebeat có thể gửi mỗi dòng thành một event riêng.

Ví dụ cấu hình:

```yaml
parsers:
  - multiline:
      type: pattern
      pattern: '^\['
      negate: true
      match: after
```

Ý nghĩa đơn giản:

* Dòng nào không bắt đầu bằng `[` thì nối vào dòng trước đó.
* Phù hợp với log có format bắt đầu bằng timestamp như `[2026-05-24 10:00:00]`.

---

# 7. Winlogbeat cấu hình cơ bản

File cấu hình chính thường là:

```text
C:\Program Files\Winlogbeat\winlogbeat.yml
```

## 7.1. Ví dụ winlogbeat.yml cơ bản

```yaml
winlogbeat.event_logs:
  - name: Security
  - name: System
  - name: Application

output.logstash:
  hosts: ["192.168.1.10:5044"]
```

Giải thích:

| Dòng                    | Ý nghĩa                                |
| ----------------------- | -------------------------------------- |
| `winlogbeat.event_logs` | Khai báo các Windows Event Log cần đọc |
| `name: Security`        | Đọc Security log                       |
| `name: System`          | Đọc System log                         |
| `name: Application`     | Đọc Application log                    |
| `output.logstash`       | Gửi log sang Logstash                  |
| `hosts`                 | IP/domain và port Logstash             |

---

## 7.2. Các biến hay dùng trong Winlogbeat

### `name`

Tên channel log trong Windows Event Viewer.

Ví dụ:

```yaml
- name: Security
```

Các channel phổ biến:

```yaml
- name: Security
- name: System
- name: Application
- name: Windows PowerShell
- name: Microsoft-Windows-PowerShell/Operational
```

### `event_id`

Chỉ lấy một số Event ID nhất định.

```yaml
- name: Security
  event_id: 4624, 4625, 4672, 4720, 4722, 4726
```

Ý nghĩa: chỉ lấy các event đăng nhập, quyền đặc biệt, tạo user, enable user, xóa user.

Có thể loại trừ Event ID:

```yaml
- name: Security
  event_id: 4624, 4625, -4634
```

Ý nghĩa: lấy 4624, 4625 nhưng bỏ 4634.

### `ignore_older`

Không lấy log quá cũ.

```yaml
ignore_older: 72h
```

### `level`

Lọc theo mức độ log.

```yaml
level: critical, error, warning
```

### `provider`

Lọc theo provider.

Ví dụ với PowerShell:

```yaml
- name: Microsoft-Windows-PowerShell/Operational
  provider:
    - Microsoft-Windows-PowerShell
```

### `fields`

Thêm field custom.

```yaml
fields:
  log_source: windows
  environment: production
fields_under_root: true
```

---

# 8. Logstash cấu hình cơ bản

File pipeline Logstash thường nằm ở:

```text
/etc/logstash/conf.d/
```

Ví dụ:

```text
/etc/logstash/conf.d/beats-input.conf
/etc/logstash/conf.d/windows-security.conf
/etc/logstash/conf.d/output-opensearch.conf
```

Một file đơn giản:

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  mutate {
    add_field => { "processed_by" => "logstash" }
  }
}

output {
  opensearch {
    hosts => ["https://192.168.1.20:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    user => "admin"
    password => "your_password"
    ssl => true
    ssl_certificate_verification => false
  }
}
```

---

# 9. Các plugin quan trọng trong Logstash

## 9.1. input plugin

Input là nơi Logstash nhận dữ liệu.

### `beats`

Nhận dữ liệu từ Filebeat/Winlogbeat.

```conf
input {
  beats {
    port => 5044
  }
}
```

### `file`

Logstash tự đọc file log trực tiếp.

```conf
input {
  file {
    path => "/var/log/app.log"
    start_position => "beginning"
  }
}
```

Thực tế nên dùng Filebeat đọc file, rồi gửi Logstash. Không nên bắt Logstash đọc quá nhiều file trên nhiều server.

### `tcp`

Nhận log qua TCP.

```conf
input {
  tcp {
    port => 5000
  }
}
```

### `udp`

Nhận log qua UDP, thường dùng cho syslog.

```conf
input {
  udp {
    port => 514
  }
}
```

---

## 9.2. filter plugin

Filter là phần quan trọng nhất trong Logstash.

## `mutate`

Dùng để sửa field.

### Thêm field

```conf
filter {
  mutate {
    add_field => { "environment" => "production" }
  }
}
```

### Đổi tên field

```conf
filter {
  mutate {
    rename => { "host" => "source_host" }
  }
}
```

### Xóa field

```conf
filter {
  mutate {
    remove_field => ["agent", "ecs", "input"]
  }
}
```

### Chuyển kiểu dữ liệu

```conf
filter {
  mutate {
    convert => { "response_time" => "float" }
  }
}
```

Các kiểu thường gặp:

```text
integer
float
string
boolean
```

### Thay thế giá trị

```conf
filter {
  mutate {
    replace => { "event_category" => "authentication" }
  }
}
```

---

## `grok`

Dùng để parse log text thành field.

Ví dụ log:

```text
2026-05-24 10:15:30 INFO user=admin action=login status=success
```

Grok:

```conf
filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:log_time} %{LOGLEVEL:log_level} user=%{WORD:user} action=%{WORD:action} status=%{WORD:status}"
    }
  }
}
```

Kết quả:

```json
{
  "log_time": "2026-05-24 10:15:30",
  "log_level": "INFO",
  "user": "admin",
  "action": "login",
  "status": "success"
}
```

### Pattern grok hay dùng

| Pattern                     | Ý nghĩa                  |
| --------------------------- | ------------------------ |
| `%{WORD:name}`              | Một từ                   |
| `%{NUMBER:num}`             | Số                       |
| `%{INT:id}`                 | Số nguyên                |
| `%{DATA:data}`              | Dữ liệu ngắn, non-greedy |
| `%{GREEDYDATA:msg}`         | Lấy hết phần còn lại     |
| `%{IP:client_ip}`           | Địa chỉ IP               |
| `%{HOSTNAME:host}`          | Hostname                 |
| `%{TIMESTAMP_ISO8601:time}` | Timestamp                |
| `%{LOGLEVEL:level}`         | INFO, WARN, ERROR        |

---

## `date`

Dùng để set timestamp đúng cho event.

Nếu log có timestamp riêng, nên dùng `date` để Logstash hiểu đúng thời gian event.

```conf
filter {
  date {
    match => ["log_time", "yyyy-MM-dd HH:mm:ss"]
    target => "@timestamp"
    timezone => "Asia/Ho_Chi_Minh"
  }
}
```

Ý nghĩa:

* Lấy field `log_time`
* Parse theo format `yyyy-MM-dd HH:mm:ss`
* Gán vào `@timestamp`
* Dùng timezone Việt Nam

---

## `json`

Dùng khi message là JSON string.

Ví dụ log:

```json
{"user":"admin","action":"login","status":"success"}
```

Config:

```conf
filter {
  json {
    source => "message"
  }
}
```

Nếu muốn đưa JSON vào field con:

```conf
filter {
  json {
    source => "message"
    target => "parsed"
  }
}
```

---

## `if / else`

Dùng để xử lý log theo điều kiện.

Ví dụ:

```conf
filter {
  if [event][code] == "4625" {
    mutate {
      add_field => { "alert_type" => "failed_login" }
    }
  } else if [event][code] == "4624" {
    mutate {
      add_field => { "alert_type" => "successful_login" }
    }
  } else {
    mutate {
      add_field => { "alert_type" => "other" }
    }
  }
}
```

---

# 10. Field reference trong Logstash

Logstash dùng cú pháp field như sau:

```conf
[field]
[parent][child]
[parent][child][sub_child]
```

Ví dụ event:

```json
{
  "event": {
    "code": "4624"
  },
  "host": {
    "name": "WIN-SERVER-01"
  }
}
```

Muốn gọi Event ID:

```conf
[event][code]
```

Muốn gọi hostname:

```conf
[host][name]
```

Ví dụ điều kiện:

```conf
if [event][code] == "4624" {
  mutate {
    add_field => { "login_status" => "success" }
  }
}
```

---

# 11. Output trong Logstash

## 11.1. Output ra console để debug

```conf
output {
  stdout {
    codec => rubydebug
  }
}
```

Dùng khi muốn xem event sau khi filter.

## 11.2. Output sang OpenSearch

```conf
output {
  opensearch {
    hosts => ["https://localhost:9200"]
    index => "security-winlogbeat-%{+YYYY.MM.dd}"
    user => "admin"
    password => "admin"
    ssl => true
    ssl_certificate_verification => false
  }
}
```

## 11.3. Output sang Elasticsearch

```conf
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

---

# 12. Index naming nên đặt thế nào?

Index nên đặt theo source + ngày.

Ví dụ:

```text
filebeat-linux-%{+YYYY.MM.dd}
winlogbeat-security-%{+YYYY.MM.dd}
winlogbeat-system-%{+YYYY.MM.dd}
app-log-%{+YYYY.MM.dd}
```

Không nên để tất cả log vào một index duy nhất vì:

* Khó quản lý
* Khó search
* Khó phân quyền
* Khó set retention
* Khó tối ưu mapping

Ví dụ output tách index theo log source:

```conf
output {
  if [log_source] == "windows_security" {
    opensearch {
      hosts => ["https://localhost:9200"]
      index => "windows-security-%{+YYYY.MM.dd}"
      user => "admin"
      password => "admin"
      ssl => true
      ssl_certificate_verification => false
    }
  } else if [log_source] == "linux" {
    opensearch {
      hosts => ["https://localhost:9200"]
      index => "linux-log-%{+YYYY.MM.dd}"
      user => "admin"
      password => "admin"
      ssl => true
      ssl_certificate_verification => false
    }
  }
}
```

---

# 13. Ví dụ hoàn chỉnh: Winlogbeat Security → Logstash → OpenSearch

## 13.1. winlogbeat.yml

```yaml
winlogbeat.event_logs:
  - name: Security
    event_id: 4624, 4625, 4672, 4720, 4722, 4726, 4740
    ignore_older: 72h
    fields:
      log_source: windows_security
      environment: production
    fields_under_root: true

output.logstash:
  hosts: ["192.168.1.10:5044"]
```

## 13.2. logstash pipeline

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [log_source] == "windows_security" {
    mutate {
      add_field => { "data_type" => "windows_event_log" }
    }

    if [event][code] == "4624" {
      mutate {
        add_field => { "event_action_vi" => "Đăng nhập thành công" }
      }
    } else if [event][code] == "4625" {
      mutate {
        add_field => { "event_action_vi" => "Đăng nhập thất bại" }
      }
    } else if [event][code] == "4720" {
      mutate {
        add_field => { "event_action_vi" => "Tạo tài khoản người dùng" }
      }
    } else if [event][code] == "4722" {
      mutate {
        add_field => { "event_action_vi" => "Kích hoạt tài khoản người dùng" }
      }
    } else if [event][code] == "4726" {
      mutate {
        add_field => { "event_action_vi" => "Xóa tài khoản người dùng" }
      }
    }
  }
}

output {
  opensearch {
    hosts => ["https://192.168.1.20:9200"]
    index => "security-winlogbeat-%{+YYYY.MM.dd}"
    user => "admin"
    password => "your_password"
    ssl => true
    ssl_certificate_verification => false
  }
}
```

---

# 14. Ví dụ hoàn chỉnh: Filebeat app log → Logstash → OpenSearch

## 14.1. File log mẫu

```text
2026-05-24 10:15:30 INFO user=admin action=login status=success ip=192.168.1.5
2026-05-24 10:16:10 ERROR user=guest action=login status=failed ip=192.168.1.8
```

## 14.2. filebeat.yml

```yaml
filebeat.inputs:
  - type: filestream
    id: app-auth-log
    enabled: true
    paths:
      - /opt/app/logs/auth.log
    fields:
      log_source: app_auth
      environment: production
    fields_under_root: true

output.logstash:
  hosts: ["192.168.1.10:5044"]
```

## 14.3. logstash pipeline

```conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [log_source] == "app_auth" {
    grok {
      match => {
        "message" => "%{TIMESTAMP_ISO8601:log_time} %{LOGLEVEL:log_level} user=%{WORD:user} action=%{WORD:action} status=%{WORD:status} ip=%{IP:client_ip}"
      }
    }

    date {
      match => ["log_time", "yyyy-MM-dd HH:mm:ss"]
      target => "@timestamp"
      timezone => "Asia/Ho_Chi_Minh"
    }

    mutate {
      add_field => { "data_type" => "application_auth_log" }
    }
  }
}

output {
  opensearch {
    hosts => ["https://192.168.1.20:9200"]
    index => "app-auth-%{+YYYY.MM.dd}"
    user => "admin"
    password => "your_password"
    ssl => true
    ssl_certificate_verification => false
  }
}
```

---

# 15. Các lệnh thường dùng

## 15.1. Filebeat

Kiểm tra config:

```bash
sudo filebeat test config
```

Kiểm tra output:

```bash
sudo filebeat test output
```

Chạy Filebeat:

```bash
sudo systemctl start filebeat
```

Restart:

```bash
sudo systemctl restart filebeat
```

Xem trạng thái:

```bash
sudo systemctl status filebeat
```

Xem log:

```bash
sudo journalctl -u filebeat -f
```

---

## 15.2. Winlogbeat

Trên PowerShell, chạy bằng quyền Administrator.

Kiểm tra config:

```powershell
.\winlogbeat.exe test config -c .\winlogbeat.yml
```

Kiểm tra output:

```powershell
.\winlogbeat.exe test output -c .\winlogbeat.yml
```

Cài service:

```powershell
.\install-service-winlogbeat.ps1
```

Start service:

```powershell
Start-Service winlogbeat
```

Stop service:

```powershell
Stop-Service winlogbeat
```

Restart service:

```powershell
Restart-Service winlogbeat
```

Xem trạng thái:

```powershell
Get-Service winlogbeat
```

---

## 15.3. Logstash

Kiểm tra config:

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

Restart:

```bash
sudo systemctl restart logstash
```

Xem trạng thái:

```bash
sudo systemctl status logstash --no-pager
```

Xem log:

```bash
sudo journalctl -u logstash -f
```

Xem log file trực tiếp:

```bash
sudo tail -f /var/log/logstash/logstash-plain.log
```

---

# 16. Cách debug khi log không vào OpenSearch

## Bước 1: Kiểm tra Beat có chạy không

Filebeat:

```bash
sudo systemctl status filebeat
```

Winlogbeat:

```powershell
Get-Service winlogbeat
```

## Bước 2: Kiểm tra Beat có gửi được sang Logstash không

Filebeat:

```bash
sudo filebeat test output
```

Winlogbeat:

```powershell
.\winlogbeat.exe test output -c .\winlogbeat.yml
```

Nếu lỗi connection refused:

* Logstash chưa chạy
* Sai IP Logstash
* Sai port
* Firewall chặn port 5044
* Logstash không mở input beats

## Bước 3: Kiểm tra Logstash có listen port chưa

```bash
sudo ss -lntp | grep 5044
```

Nếu không thấy port 5044, nghĩa là Logstash chưa mở beats input.

## Bước 4: Test Logstash config

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

Nếu config sai syntax, Logstash sẽ không chạy pipeline.

## Bước 5: Output ra stdout để debug

Tạm thời đổi output thành:

```conf
output {
  stdout {
    codec => rubydebug
  }
}
```

Nếu thấy log in ra màn hình, nghĩa là input và filter đã chạy.

## Bước 6: Kiểm tra OpenSearch

Các lỗi thường gặp:

* Sai user/password
* Sai SSL setting
* Sai certificate
* Index permission không đủ
* OpenSearch disk full
* Mapping conflict

---

# 17. Lỗi thường gặp và cách hiểu

## 17.1. `connection refused`

Ý nghĩa: Beat không kết nối được Logstash.

Nguyên nhân thường gặp:

* Logstash chưa chạy
* Sai port
* Firewall block
* Logstash chưa có `beats { port => 5044 }`

## 17.2. `_grokparsefailure`

Ý nghĩa: grok pattern không match với log message.

Cách xử lý:

* Xem lại format log thật
* Test grok từng phần nhỏ
* Dùng `%{GREEDYDATA:raw_message}` tạm thời để tránh mất log
* Không nên viết grok quá phức tạp ngay từ đầu

## 17.3. `_dateparsefailure`

Ý nghĩa: date filter không parse được timestamp.

Nguyên nhân:

* Sai format ngày giờ
* Thiếu timezone
* Field timestamp không tồn tại

## 17.4. Logstash stuck ở `deactivating`

Ý nghĩa: service đang dừng nhưng process Java chưa thoát.

Có thể kiểm tra:

```bash
ps aux | grep logstash
```

Nếu cần dừng mạnh:

```bash
sudo kill -9 <PID>
```

Sau đó start lại:

```bash
sudo systemctl start logstash
```

## 17.5. Mapping conflict

Ý nghĩa: cùng một field nhưng lúc là string, lúc là number/object.

Ví dụ:

```text
user = "admin"
user = { "name": "admin" }
```

OpenSearch sẽ bị conflict.

Cách xử lý:

* Chuẩn hóa field name
* Rename field trước khi output
* Tách index theo source

---

# 18. Best practices

## 18.1. Không gom tất cả log vào một index

Nên tách theo source:

```text
windows-security-YYYY.MM.dd
windows-system-YYYY.MM.dd
linux-auth-YYYY.MM.dd
app-auth-YYYY.MM.dd
```

## 18.2. Thêm field `log_source`

Nên thêm ở Beat:

```yaml
fields:
  log_source: windows_security
fields_under_root: true
```

Vì Logstash sẽ dễ route:

```conf
if [log_source] == "windows_security" {
  # xử lý security log
}
```

## 18.3. Luôn test config trước khi restart

```bash
sudo /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

## 18.4. Filter càng đơn giản càng tốt

Không nên viết một pipeline quá dài, quá nhiều if else.

Nên chia pipeline theo nhóm:

```text
01-input.conf
10-filter-windows.conf
20-filter-linux.conf
30-output.conf
```

## 18.5. Không xóa field quá sớm

Khi mới debug, đừng `remove_field` quá nhiều.

Nên giữ field gốc để dễ kiểm tra.

Sau khi pipeline ổn mới tối ưu field.

---

# 19. Checklist triển khai nhanh

## Trên máy gửi log

* Cài Filebeat hoặc Winlogbeat
* Cấu hình input đúng
* Cấu hình output Logstash đúng IP/port
* Test config
* Test output
* Start service

## Trên máy Logstash

* Cài Logstash
* Tạo pipeline input beats port 5044
* Tạo filter phù hợp
* Tạo output OpenSearch
* Test config
* Restart Logstash
* Kiểm tra port 5044
* Xem log Logstash

## Trên OpenSearch

* Kiểm tra index có được tạo không
* Kiểm tra Discover/Search
* Kiểm tra timestamp
* Kiểm tra field có đúng không
* Tạo index pattern nếu cần

---

# 20. Cách học nhanh nhất

Thứ tự nên học:

1. Hiểu Filebeat/Winlogbeat dùng để lấy log từ đâu.
2. Hiểu Logstash có 3 phần: input, filter, output.
3. Biết đọc event JSON sau khi Beat gửi qua.
4. Biết gọi field bằng cú pháp `[field][child]`.
5. Biết dùng `mutate` để thêm/sửa/xóa field.
6. Biết dùng `if` để phân loại log.
7. Biết dùng `grok` khi log là text thô.
8. Biết dùng `date` để chuẩn hóa thời gian.
9. Biết debug bằng `stdout { codec => rubydebug }`.
10. Biết xem log service bằng `journalctl` hoặc PowerShell.

---

# 21. ELK Stack là gì?

## 21.1. ELK là viết tắt của gì?

**ELK** là viết tắt của:

```text
E = Elasticsearch
L = Logstash
K = Kibana
```

Ban đầu ELK chỉ gồm 3 thành phần chính: Elasticsearch, Logstash, Kibana.

Sau này Elastic bổ sung thêm **Beats** và **Elastic Agent**, nên tên gọi chính thức rộng hơn là **Elastic Stack**.

Tuy nhiên, trong thực tế nhiều người vẫn gọi chung là **ELK Stack**.

---

## 21.2. ELK dùng để làm gì?

ELK dùng để:

* Thu thập log từ nhiều server/app khác nhau.
* Tập trung log về một nơi.
* Tìm kiếm log nhanh.
* Phân tích lỗi hệ thống.
* Theo dõi đăng nhập, bảo mật, hành vi user.
* Tạo dashboard giám sát.
* Tạo alert khi có dấu hiệu bất thường.

Ví dụ thực tế:

```text
Có 10 server khác nhau.
Mỗi server có log riêng.
Nếu không có ELK, phải SSH từng server để xem log.
Nếu có ELK, tất cả log được gom về một nơi để search trên Kibana.
```

---

# 22. Các thành phần chính trong ELK

## 22.1. Elasticsearch

**Elasticsearch** là nơi lưu trữ và tìm kiếm dữ liệu.

Nói dễ hiểu: Elasticsearch giống như “database cho log”, nhưng mạnh hơn database thường ở khả năng search text rất nhanh.

Elasticsearch lưu dữ liệu dưới dạng JSON document.

Ví dụ một log event trong Elasticsearch:

```json
{
  "@timestamp": "2026-05-26T10:00:00Z",
  "host": {
    "name": "WIN-SERVER-01"
  },
  "event": {
    "code": "4625",
    "action": "failed-login"
  },
  "user": {
    "name": "admin"
  },
  "source": {
    "ip": "192.168.1.10"
  }
}
```

Elasticsearch phù hợp với:

* Log search
* Full-text search
* Metrics
* Security event
* Application monitoring
* Audit trail
* Business event tracking

---

## 22.2. Logstash

**Logstash** là nơi xử lý dữ liệu trước khi lưu vào Elasticsearch.

Vai trò chính:

* Nhận log từ Beats, app, syslog, file, TCP/UDP.
* Parse log text thành field.
* Chuẩn hóa field.
* Thêm field mới.
* Xóa field thừa.
* Đổi format timestamp.
* Gửi dữ liệu sang Elasticsearch.

Ví dụ:

Log thô:

```text
2026-05-26 10:00:00 ERROR user=admin action=login status=failed
```

Sau khi qua Logstash:

```json
{
  "log_time": "2026-05-26 10:00:00",
  "log_level": "ERROR",
  "user": "admin",
  "action": "login",
  "status": "failed"
}
```

---

## 22.3. Kibana

**Kibana** là giao diện để xem, search, visualize dữ liệu trong Elasticsearch.

Nói dễ hiểu: Kibana là “màn hình dashboard” của ELK.

Kibana dùng để:

* Search log.
* Filter log theo thời gian.
* Tạo dashboard.
* Tạo chart.
* Tạo table.
* Theo dõi security event.
* Quản lý index pattern / data view.
* Tạo alert.

Ví dụ các dashboard thường làm:

| Dashboard                   | Mục đích                                      |
| --------------------------- | --------------------------------------------- |
| Windows Security Dashboard  | Theo dõi login success/failed, account change |
| Linux Auth Dashboard        | Theo dõi SSH login, sudo, failed password     |
| Application Error Dashboard | Theo dõi lỗi app theo service                 |
| Nginx Access Dashboard      | Theo dõi traffic, status code, IP             |
| System Health Dashboard     | Theo dõi CPU, RAM, disk, service              |

---

## 22.4. Beats

**Beats** là các lightweight agent dùng để ship dữ liệu.

Các Beats phổ biến:

| Beat       | Chức năng                                |
| ---------- | ---------------------------------------- |
| Filebeat   | Đọc log từ file                          |
| Winlogbeat | Đọc Windows Event Log                    |
| Metricbeat | Thu thập CPU, RAM, disk, network         |
| Packetbeat | Thu thập network packet metadata         |
| Auditbeat  | Thu thập audit/security event trên Linux |
| Heartbeat  | Kiểm tra service có sống không           |

Trong project log monitoring cơ bản, thường gặp nhất là:

```text
Filebeat + Winlogbeat + Logstash + Elasticsearch + Kibana
```

---

# 23. Luồng ELK đầy đủ

## 23.1. Luồng cơ bản

```text
Application / Server / Windows Event Log
        ↓
Filebeat / Winlogbeat
        ↓
Logstash
        ↓
Elasticsearch
        ↓
Kibana
```

Ý nghĩa từng bước:

| Bước | Công cụ             | Vai trò               |
| ---: | ------------------- | --------------------- |
|    1 | App/Server          | Sinh ra log           |
|    2 | Filebeat/Winlogbeat | Thu thập log          |
|    3 | Logstash            | Parse, filter, enrich |
|    4 | Elasticsearch       | Lưu và search         |
|    5 | Kibana              | Hiển thị dashboard    |

---

## 23.2. Luồng không dùng Logstash

Có thể gửi Beat trực tiếp vào Elasticsearch:

```text
Filebeat / Winlogbeat
        ↓
Elasticsearch
        ↓
Kibana
```

Dùng khi:

* Log đã có format chuẩn.
* Không cần parse phức tạp.
* Muốn giảm tài nguyên vận hành.
* Dùng module có sẵn của Elastic.

Không nên dùng khi:

* Log cần xử lý phức tạp.
* Cần route nhiều index khác nhau.
* Cần enrich dữ liệu.
* Cần custom logic nhiều.

---

## 23.3. Luồng có Kafka ở giữa

Hệ thống lớn có thể thêm Kafka:

```text
Filebeat / Winlogbeat
        ↓
Kafka
        ↓
Logstash
        ↓
Elasticsearch
        ↓
Kibana
```

Dùng Kafka khi:

* Log volume lớn.
* Cần buffer để tránh mất log.
* Elasticsearch/Logstash có thể downtime tạm thời.
* Có nhiều consumer cùng đọc log.

Với hệ thống nhỏ hoặc lab, chưa cần Kafka.

---

# 24. Elasticsearch concepts cần biết

## 24.1. Document

**Document** là một bản ghi dữ liệu trong Elasticsearch.

Ví dụ: một dòng log = một document.

```json
{
  "user": "admin",
  "action": "login",
  "status": "failed"
}
```

---

## 24.2. Index

**Index** là nơi chứa nhiều document cùng loại.

Ví dụ:

```text
windows-security-2026.05.26
linux-auth-2026.05.26
nginx-access-2026.05.26
```

Có thể hiểu gần giống “table” trong database, nhưng không hoàn toàn giống.

---

## 24.3. Field

**Field** là từng thuộc tính trong document.

Ví dụ:

```json
{
  "user": "admin",
  "status": "failed"
}
```

Ở đây có 2 field:

```text
user
status
```

---

## 24.4. Mapping

**Mapping** là định nghĩa kiểu dữ liệu của field.

Ví dụ:

```text
user.name     → keyword
message       → text
status_code   → integer
@timestamp    → date
source.ip     → ip
```

Mapping rất quan trọng vì nếu sai mapping thì search/filter/dashboard có thể lỗi.

Ví dụ lỗi phổ biến:

```text
status_code lúc đầu được lưu là string: "200"
Sau đó lại gửi number: 200
```

Có thể gây mapping conflict.

---

## 24.5. Kiểu field thường gặp

| Type      | Dùng cho                                   | Ví dụ                      |
| --------- | ------------------------------------------ | -------------------------- |
| `keyword` | Giá trị chính xác, dùng filter/aggregation | username, hostname, status |
| `text`    | Nội dung cần full-text search              | message, description       |
| `integer` | Số nguyên                                  | status_code, event_id      |
| `float`   | Số thập phân                               | response_time              |
| `date`    | Thời gian                                  | @timestamp                 |
| `ip`      | Địa chỉ IP                                 | source.ip                  |
| `boolean` | true/false                                 | is_success                 |
| `object`  | Field cha chứa field con                   | user.name, host.name       |

Cách nhớ nhanh:

```text
keyword = filter chính xác
text = search nội dung dài
date = thời gian
ip = địa chỉ IP
number = tính toán/biểu đồ
```

---

## 24.6. Shard

**Shard** là phần chia nhỏ của index.

Elasticsearch chia index thành nhiều shard để phân tán dữ liệu trên nhiều node.

Ví dụ:

```text
Index: windows-security-2026.05.26
Primary shard: 3
Replica shard: 1
```

Ý nghĩa:

* Primary shard: nơi chứa dữ liệu chính.
* Replica shard: bản sao để tăng độ an toàn và khả năng search.

Không nên tạo quá nhiều shard cho index nhỏ, vì shard cũng tốn tài nguyên.

---

## 24.7. Replica

**Replica** là bản sao của primary shard.

Tác dụng:

* Tăng khả năng chịu lỗi.
* Nếu một node chết, dữ liệu vẫn còn.
* Tăng khả năng search song song.

Ví dụ:

```text
number_of_replicas: 1
```

Nghĩa là mỗi primary shard có 1 bản sao.

---

## 24.8. Cluster

**Cluster** là toàn bộ cụm Elasticsearch.

Một cluster có thể gồm một hoặc nhiều node.

```text
Elasticsearch Cluster
  ├── Node 1
  ├── Node 2
  └── Node 3
```

---

## 24.9. Node

**Node** là một server chạy Elasticsearch.

Một node có thể giữ nhiều vai trò:

| Node role        | Ý nghĩa                         |
| ---------------- | ------------------------------- |
| master           | Quản lý cluster state           |
| data             | Lưu dữ liệu                     |
| ingest           | Xử lý ingest pipeline           |
| coordinating     | Nhận request và phân phối query |
| machine_learning | Chạy ML feature nếu có license  |

Với lab nhỏ, một node có thể làm tất cả vai trò.

Với production, nên tách role nếu hệ thống lớn.

---

# 25. Kibana concepts cần biết

## 25.1. Data View / Index Pattern

Trong Kibana, muốn xem dữ liệu thì cần tạo **Data View**.

Ví dụ index trong Elasticsearch:

```text
windows-security-2026.05.26
windows-security-2026.05.27
windows-security-2026.05.28
```

Data View có thể đặt:

```text
windows-security-*
```

Dấu `*` nghĩa là match nhiều index.

---

## 25.2. Discover

**Discover** là nơi search log thô.

Dùng Discover để:

* Xem log mới nhất.
* Filter theo field.
* Search keyword.
* Kiểm tra field có đúng không.
* Debug xem log có vào chưa.

Ví dụ query trong Kibana:

```text
event.code: "4625"
```

Ý nghĩa: tìm failed login.

```text
host.name: "WIN-SERVER-01"
```

Ý nghĩa: tìm log từ host cụ thể.

```text
event.code: "4625" and user.name: "admin"
```

Ý nghĩa: tìm failed login của user admin.

---

## 25.3. Dashboard

**Dashboard** là nơi gom nhiều chart/table lại để theo dõi.

Ví dụ dashboard Security:

* Tổng số login failed theo thời gian.
* Top user bị login failed.
* Top source IP.
* Số account bị tạo mới.
* Số account bị enable/disable.
* Event đặc biệt như 4672, 4720, 4722.

---

## 25.4. Lens

**Lens** là công cụ kéo-thả để tạo chart trong Kibana.

Có thể tạo:

* Bar chart
* Line chart
* Pie chart
* Metric card
* Table
* Heatmap

Ví dụ:

```text
Chart: Failed login over time
X-axis: @timestamp
Y-axis: Count
Filter: event.code = 4625
```

---

## 25.5. Alert

Kibana có thể tạo alert khi dữ liệu match điều kiện.

Ví dụ:

```text
Nếu trong 5 phút có hơn 10 lần failed login từ cùng một user → cảnh báo.
```

Hoặc:

```text
Nếu có Event ID 4720 tạo user mới → cảnh báo.
```

Các kênh alert có thể là:

* Email
* Slack
* Webhook
* Microsoft Teams
* Jira

Tùy license và setup.

---

# 26. ILM - Index Lifecycle Management

## 26.1. ILM là gì?

**ILM** là cơ chế quản lý vòng đời index.

Log sinh ra mỗi ngày rất nhiều. Nếu không có chính sách xóa hoặc rollover, disk sẽ đầy.

ILM giúp tự động:

* Tạo index mới khi index cũ quá lớn.
* Chuyển index sang phase ít tốn tài nguyên hơn.
* Xóa index cũ sau một khoảng thời gian.

---

## 26.2. Các phase trong ILM

| Phase  | Ý nghĩa                              |
| ------ | ------------------------------------ |
| Hot    | Dữ liệu mới, ghi/search thường xuyên |
| Warm   | Dữ liệu cũ hơn, ít ghi, vẫn search   |
| Cold   | Dữ liệu lâu hơn, ít search           |
| Frozen | Dữ liệu rất cũ, search chậm hơn      |
| Delete | Xóa dữ liệu                          |

Với lab hoặc hệ thống nhỏ, có thể chỉ cần:

```text
Hot → Delete
```

Ví dụ:

```text
Giữ log 30 ngày, sau đó xóa.
```

---

## 26.3. Vì sao ILM quan trọng?

Nếu không có ILM:

* Disk có thể đầy.
* Elasticsearch chậm.
* Query lâu.
* Cluster dễ yellow/red.
* Phải xóa index thủ công.

Best practice:

```text
Log security quan trọng: giữ lâu hơn.
Log debug/app tạm thời: giữ ngắn hơn.
```

Ví dụ:

| Loại log              | Retention gợi ý |
| --------------------- | --------------: |
| Security log          |   90 - 180 ngày |
| Application error log |    30 - 90 ngày |
| Debug log             |     7 - 14 ngày |
| Access log volume lớn |    14 - 30 ngày |

---

# 27. Data Stream là gì?

**Data Stream** là cách quản lý dữ liệu time-series mới hơn, phù hợp cho log/metrics.

Thay vì tự tạo index mỗi ngày như:

```text
logs-2026.05.26
logs-2026.05.27
logs-2026.05.28
```

Data Stream cho bạn một tên logic ổn định:

```text
logs-windows-security
```

Bên dưới, Elasticsearch tự quản lý backing indices.

Dùng Data Stream khi:

* Dữ liệu có `@timestamp`.
* Dữ liệu dạng log/metric/event.
* Muốn rollover tự động.
* Muốn dùng ILM/data lifecycle gọn hơn.

Với người mới học, có thể bắt đầu bằng index theo ngày trước. Khi hiểu rõ hơn thì học Data Stream.

---

# 28. ELK vs OpenSearch

## 28.1. Khác nhau ngắn gọn

| Tiêu chí      | ELK / Elastic Stack                   | OpenSearch Stack                            |
| ------------- | ------------------------------------- | ------------------------------------------- |
| Search engine | Elasticsearch                         | OpenSearch                                  |
| Dashboard     | Kibana                                | OpenSearch Dashboards                       |
| Ingest        | Logstash / Beats / Elastic Agent      | Logstash / Beats-compatible / Fluent Bit... |
| Công ty chính | Elastic                               | OpenSearch project, khởi nguồn từ AWS fork  |
| Use case      | Search, logs, observability, security | Search, logs, observability, security       |

Nói đơn giản:

```text
ELK dùng Elasticsearch + Kibana.
OpenSearch dùng OpenSearch + OpenSearch Dashboards.
```

Về concept thì rất giống nhau:

```text
Index
Document
Field
Mapping
Shard
Replica
Dashboard
Query
Alert
```

Nhưng về version, plugin, license, API chi tiết có thể khác.

---

## 28.2. Khi nào dùng ELK?

Dùng ELK khi:

* Công ty đang dùng Elastic chính thức.
* Cần hệ sinh thái Elastic đầy đủ.
* Có license phù hợp.
* Cần tính năng mới của Elastic.
* Muốn dùng Elastic Agent/Fleet mạnh hơn.

## 28.3. Khi nào dùng OpenSearch?

Dùng OpenSearch khi:

* Muốn open-source oriented hơn.
* Đang dùng AWS/OpenSearch.
* Muốn tránh phụ thuộc license Elastic.
* Đã có OpenSearch Dashboards.
* Hệ thống hiện tại đã dùng OpenSearch.

---

# 29. ELK architecture mẫu

## 29.1. Architecture nhỏ cho lab

```text
[Linux Server] ---- Filebeat ----\
                                \
[Windows Server] -- Winlogbeat ----> [Logstash] ---> [Elasticsearch] ---> [Kibana]
                                /
[App Server] ----- Filebeat ----/
```

Một server có thể chạy cả:

```text
Logstash + Elasticsearch + Kibana
```

Phù hợp để học/lab, không tối ưu cho production.

---

## 29.2. Architecture production cơ bản

```text
[Servers with Beats]
        ↓
[Logstash Nodes]
        ↓
[Elasticsearch Cluster]
        ↓
[Kibana]
```

Elasticsearch cluster có thể gồm:

```text
3 master nodes
3 data nodes
1-2 coordinating nodes
```

Tùy quy mô log.

---

## 29.3. Architecture production có buffer

```text
[Beats]
   ↓
[Kafka]
   ↓
[Logstash Consumer]
   ↓
[Elasticsearch Cluster]
   ↓
[Kibana]
```

Kafka giúp chống mất log khi Elasticsearch hoặc Logstash bị nghẽn.

---

# 30. Query cơ bản trong Kibana

## 30.1. Tìm theo field

```text
host.name: "WIN-SERVER-01"
```

## 30.2. Tìm failed login

```text
event.code: "4625"
```

## 30.3. Tìm successful login

```text
event.code: "4624"
```

## 30.4. Tìm log có chữ error

```text
message: "error"
```

## 30.5. Kết hợp điều kiện

```text
event.code: "4625" and user.name: "admin"
```

## 30.6. Loại trừ điều kiện

```text
event.code: "4625" and not source.ip: "127.0.0.1"
```

## 30.7. Tìm nhiều giá trị

```text
event.code: ("4624" or "4625")
```

---

# 31. Dashboard nên làm cho log monitoring

## 31.1. Windows Security Dashboard

Nên có:

| Chart                  | Field / Logic         |
| ---------------------- | --------------------- |
| Total failed login     | `event.code: 4625`    |
| Failed login over time | count by `@timestamp` |
| Top failed users       | terms `user.name`     |
| Top source IP          | terms `source.ip`     |
| Successful login       | `event.code: 4624`    |
| Privileged login       | `event.code: 4672`    |
| User created           | `event.code: 4720`    |
| User enabled           | `event.code: 4722`    |
| User deleted           | `event.code: 4726`    |
| Account locked         | `event.code: 4740`    |

---

## 31.2. Linux Auth Dashboard

Nên có:

| Chart             | Logic                                        |
| ----------------- | -------------------------------------------- |
| SSH failed login  | message contains failed password             |
| SSH success login | message contains accepted password/publickey |
| Top usernames     | user field                                   |
| Top source IP     | source.ip                                    |
| Sudo command      | process.command_line or message              |
| Root login        | user.name = root                             |

---

## 31.3. Application Dashboard

Nên có:

| Chart                 | Logic                        |
| --------------------- | ---------------------------- |
| Error count           | log_level = ERROR            |
| Error over time       | count by @timestamp          |
| Top service error     | terms service.name           |
| Top endpoint error    | terms url.path               |
| Average response time | avg response_time            |
| P95 response time     | percentile response_time     |
| HTTP 5xx              | status_code >= 500           |
| HTTP 4xx              | status_code >= 400 and < 500 |

---

# 32. Best practices cho ELK

## 32.1. Chuẩn hóa field name

Nên theo Elastic Common Schema nếu có thể.

Ví dụ nên dùng:

```text
host.name
user.name
source.ip
event.code
event.action
log.level
service.name
```

Không nên mỗi source đặt một kiểu:

```text
username
user_name
UserName
account_name
```

Vì dashboard sẽ khó gom dữ liệu.

---

## 32.2. Tách index theo source

Nên tách:

```text
windows-security-*
linux-auth-*
app-payment-*
nginx-access-*
```

Không nên gom hết vào:

```text
logs-*
```

Nếu hệ thống nhỏ thì `logs-*` vẫn dùng được, nhưng về lâu dài sẽ khó quản lý mapping và retention.

---

## 32.3. Quản lý retention ngay từ đầu

Ngay khi triển khai nên hỏi:

```text
Log này cần giữ bao lâu?
```

Không có retention thì sớm muộn disk sẽ đầy.

---

## 32.4. Không parse quá mức

Không phải field nào cũng cần tách.

Nên tách các field dùng cho:

* Search
* Filter
* Alert
* Dashboard
* Report

Ví dụ nên tách:

```text
user.name
source.ip
event.code
status_code
log.level
service.name
```

Không nhất thiết tách mọi chữ trong message.

---

## 32.5. Dùng stdout để debug trước khi output thật

Khi viết pipeline mới, nên test:

```conf
output {
  stdout {
    codec => rubydebug
  }
}
```

Sau khi field đúng rồi mới output sang Elasticsearch.

---

# 33. Checklist học ELK

## Level 1: Cơ bản

* Hiểu ELK là gì.
* Hiểu Elasticsearch, Logstash, Kibana làm gì.
* Biết Filebeat/Winlogbeat gửi log.
* Biết tạo Data View trong Kibana.
* Biết search log trong Discover.

## Level 2: Vận hành

* Biết viết Logstash pipeline.
* Biết dùng grok/mutate/date/json.
* Biết debug `_grokparsefailure`.
* Biết kiểm tra service.
* Biết kiểm tra index có data chưa.

## Level 3: Thiết kế hệ thống

* Biết tách index.
* Biết mapping cơ bản.
* Biết shard/replica.
* Biết ILM/retention.
* Biết dashboard và alert.

## Level 4: Production

* Biết cluster sizing.
* Biết node role.
* Biết backup/snapshot.
* Biết security/user/role.
* Biết Kafka/buffer nếu log volume lớn.
* Biết tối ưu query và storage.

---

# 34. Tóm tắt cực ngắn

| Công cụ       | Vai trò                       | Dùng cho                             |
| ------------- | ----------------------------- | ------------------------------------ |
| Filebeat      | Đọc file log                  | Linux/app/nginx/apache/custom log    |
| Winlogbeat    | Đọc Windows Event Log         | Security/System/Application/Sysmon   |
| Logstash      | Xử lý và chuyển log           | Parse, filter, enrich, route, output |
| Elasticsearch | Lưu và search dữ liệu         | Index, document, query, aggregation  |
| Kibana        | Hiển thị và phân tích         | Discover, dashboard, chart, alert    |
| OpenSearch    | Alternative cho Elasticsearch | Search/log/dashboard tương tự        |

Câu dễ nhớ:

```text
Beat lấy log → Logstash xử lý log → Elasticsearch/OpenSearch lưu log → Kibana/Dashboard hiển thị log
```
