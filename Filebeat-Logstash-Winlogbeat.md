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

# 21. Tóm tắt cực ngắn

| Công cụ    | Vai trò               | Dùng cho                             |
| ---------- | --------------------- | ------------------------------------ |
| Filebeat   | Đọc file log          | Linux/app/nginx/apache/custom log    |
| Winlogbeat | Đọc Windows Event Log | Security/System/Application/Sysmon   |
| Logstash   | Xử lý và chuyển log   | Parse, filter, enrich, route, output |
| OpenSearch | Lưu và search log     | Index, dashboard, alert              |

Câu dễ nhớ:

```text
Beat lấy log → Logstash xử lý log → OpenSearch lưu log → Dashboard hiển thị log
```
