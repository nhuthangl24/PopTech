# Các khái niệm cơ bản, field/biến thường gặp trong Filebeat & Logstash

Phần này giải thích các thành phần nền tảng thường gặp khi làm pipeline **Filebeat → Logstash → OpenSearch/Elasticsearch**. Nội dung không gắn với một project cụ thể, mà tập trung vào những thứ gần như pipeline nào cũng dùng.

---

## 1. Event là gì?

Trong Logstash, mỗi dòng log hoặc mỗi bản ghi được gọi là **event**.

Ví dụ một dòng log gốc:

```text
2026-05-24 10:15:20 INFO User login success
```

Khi vào Logstash, nó thường được xem như một event dạng JSON:

```json
{
  "@timestamp": "2026-05-24T10:15:20.000Z",
  "message": "2026-05-24 10:15:20 INFO User login success",
  "host": {
    "name": "server-01"
  },
  "log": {
    "file": {
      "path": "/var/log/app.log"
    }
  }
}
```

Hiểu đơn giản:

```text
1 dòng log = 1 event
1 event = 1 document sau khi đẩy lên OpenSearch/Elasticsearch
```

---

## 2. Cấu trúc cơ bản của Logstash pipeline

Một file cấu hình Logstash thường có 3 phần:

```ruby
input {
  # Nhận log từ đâu
}

filter {
  # Xử lý, parse, thêm/xóa/sửa field
}

output {
  # Gửi log đi đâu
}
```

### Ví dụ đơn giản nhất

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  mutate {
    add_field => { "source_system" => "filebeat" }
  }
}

output {
  stdout { codec => rubydebug }
}
```

Ý nghĩa:

- `input`: nhận log từ Filebeat qua port `5044`.
- `filter`: thêm field mới tên `source_system`.
- `output`: in log ra màn hình để debug.

---

## 3. Các field mặc định thường gặp

Filebeat thường tự thêm một số field mặc định vào event.

| Field | Ý nghĩa | Ví dụ |
|---|---|---|
| `@timestamp` | Thời gian chính của event | `2026-05-24T10:15:20Z` |
| `message` | Nội dung log gốc | `INFO User login success` |
| `host.name` | Tên máy gửi log | `server-01` |
| `host.ip` | IP máy gửi log | `192.168.1.10` |
| `log.file.path` | Đường dẫn file log nguồn | `/var/log/app.log` |
| `log.offset` | Vị trí Filebeat đọc trong file | `123456` |
| `agent.type` | Loại beat | `filebeat`, `winlogbeat` |
| `agent.version` | Version beat | `8.17.0` |
| `input.type` | Loại input | `log`, `filestream` |
| `tags` | Nhãn gắn cho event | `["app", "prod"]` |
| `ecs.version` | Version Elastic Common Schema | `8.0.0` |

---

## 4. `message` — field quan trọng nhất

`message` chứa nội dung log gốc. Đây là field thường được dùng để parse bằng `grok`, `json`, `kv`, hoặc dùng điều kiện `if`.

### Ví dụ kiểm tra nội dung message

```ruby
filter {
  if "ERROR" in [message] {
    mutate {
      add_field => { "level" => "error" }
    }
  }
}
```

Nếu message là:

```text
2026-05-24 10:15:20 ERROR Cannot connect to database
```

Thì Logstash thêm field:

```json
{
  "level": "error"
}
```

---

## 5. `@timestamp` — thời gian của event

`@timestamp` là field thời gian chính. OpenSearch/Elasticsearch thường dùng field này để filter theo thời gian.

### Dùng `@timestamp` để tạo index theo ngày

```ruby
output {
  elasticsearch {
    index => "app-logs-%{+YYYY.MM.dd}"
  }
}
```

Nếu ngày là 2026-05-24, index sẽ là:

```text
app-logs-2026.05.24
```

### Lưu ý

`@timestamp` thường ở UTC. Nếu log gốc là giờ Việt Nam, nên dùng `date` filter và set timezone rõ ràng.

```ruby
filter {
  date {
    match => ["log_time", "yyyy-MM-dd HH:mm:ss"]
    target => "@timestamp"
    timezone => "Asia/Ho_Chi_Minh"
  }
}
```

---

## 6. Cách gọi field trong Logstash

### Field bình thường

```ruby
[message]
[level]
[status]
```

Ví dụ:

```ruby
if [level] == "ERROR" {
  mutate { add_field => { "severity" => "high" } }
}
```

---

### Field lồng nhau

Nếu event có dạng:

```json
{
  "host": {
    "name": "server-01"
  }
}
```

Trong Logstash phải gọi là:

```ruby
[host][name]
```

Ví dụ:

```ruby
mutate {
  add_field => {
    "server_name" => "%{[host][name]}"
  }
}
```

Output:

```json
{
  "server_name": "server-01"
}
```

---

## 7. `%{field}` — lấy giá trị field chèn vào chuỗi

Cú pháp `%{field}` dùng để lấy giá trị của field và đưa vào một chuỗi.

### Ví dụ

```ruby
mutate {
  add_field => {
    "summary" => "Log from %{[host][name]}: %{message}"
  }
}
```

Nếu event có:

```json
{
  "host": { "name": "server-01" },
  "message": "Service started"
}
```

Output:

```json
{
  "summary": "Log from server-01: Service started"
}
```

---

## 8. `%{+YYYY.MM.dd}` — format ngày từ `@timestamp`

Cú pháp này thường dùng để tạo index theo ngày.

```ruby
index => "logs-%{+YYYY.MM.dd}"
```

Ví dụ output:

```text
logs-2026.05.24
```

Một số format hay dùng:

| Cú pháp | Output ví dụ | Công dụng |
|---|---|---|
| `%{+YYYY.MM.dd}` | `2026.05.24` | Index theo ngày |
| `%{+YYYY.MM}` | `2026.05` | Index theo tháng |
| `%{+YYYY}` | `2026` | Index theo năm |
| `%{+HH}` | `10` | Tách theo giờ, ít dùng hơn |

---

## 9. `tags` — nhãn của event

`tags` là một mảng dùng để đánh dấu event.

### Thêm tag trong Filebeat

```yaml
tags: ["app", "production"]
```

### Thêm tag trong Logstash

```ruby
mutate {
  add_tag => ["important"]
}
```

### Kiểm tra tag

```ruby
if "important" in [tags] {
  mutate {
    add_field => { "priority" => "high" }
  }
}
```

### Tags thường gặp

| Tag | Ý nghĩa thường dùng |
|---|---|
| `_grokparsefailure` | Grok parse lỗi |
| `_dateparsefailure` | Date parse lỗi |
| `_jsonparsefailure` | JSON parse lỗi |
| `debug` | Log dùng để test/debug |
| `production` | Log từ môi trường production |
| `app` | Log ứng dụng |
| `security` | Log bảo mật |

---

## 10. `@metadata` — biến tạm không lưu vào output

`@metadata` là field đặc biệt của Logstash. Nó dùng để lưu thông tin tạm trong pipeline, nhưng không được gửi ra output.

Dùng khi muốn:

- Tạo index động.
- Lưu biến tạm.
- Route output.
- Không muốn field đó xuất hiện trong OpenSearch.

### Ví dụ

```ruby
filter {
  mutate {
    add_field => {
      "[@metadata][target_index]" => "logs-%{+YYYY.MM.dd}"
    }
  }
}

output {
  elasticsearch {
    index => "%{[@metadata][target_index]}"
  }
}
```

Ý nghĩa:

- `[@metadata][target_index]` chỉ dùng trong Logstash.
- Field này không xuất hiện trong document cuối.

---

## 11. Field tự thêm trong Filebeat `fields`

Trong Filebeat, `fields` dùng để thêm thông tin riêng cho mỗi input.

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/app.log
    fields:
      log_type: "application"
      environment: "production"
      service: "payment"
    fields_under_root: true
```

### Các field custom thường gặp

| Field | Ý nghĩa | Ví dụ |
|---|---|---|
| `log_type` | Loại log | `application`, `system`, `security` |
| `environment` | Môi trường | `dev`, `staging`, `production` |
| `service` | Tên service/app | `payment`, `api`, `web` |
| `team` | Team sở hữu log | `backend`, `infra`, `security` |
| `region` | Khu vực | `hcm`, `hn`, `sg` |
| `source` | Nguồn log | `nginx`, `app`, `syslog` |

### `fields_under_root: true` nghĩa là gì?

Nếu dùng:

```yaml
fields_under_root: true
```

Output:

```json
{
  "log_type": "application",
  "environment": "production",
  "message": "..."
}
```

Nếu dùng:

```yaml
fields_under_root: false
```

Output:

```json
{
  "fields": {
    "log_type": "application",
    "environment": "production"
  },
  "message": "..."
}
```

Khuyến nghị cơ bản:

```yaml
fields_under_root: true
```

Vì sau này viết điều kiện dễ hơn:

```ruby
if [environment] == "production" {
  # xử lý log production
}
```

---

## 12. `mutate` — filter hay dùng nhất

`mutate` dùng để thêm, sửa, xóa, đổi tên, đổi kiểu field.

### 12.1 `add_field` — thêm field

```ruby
mutate {
  add_field => {
    "processed_by" => "logstash"
  }
}
```

Output:

```json
{
  "processed_by": "logstash"
}
```

---

### 12.2 `remove_field` — xóa field

```ruby
mutate {
  remove_field => ["ecs", "@version", "agent"]
}
```

Dùng khi không muốn lưu field thừa vào OpenSearch.

---

### 12.3 `rename` — đổi tên field

```ruby
mutate {
  rename => {
    "clientip" => "client_ip"
  }
}
```

Trước:

```json
{ "clientip": "192.168.1.10" }
```

Sau:

```json
{ "client_ip": "192.168.1.10" }
```

---

### 12.4 `convert` — đổi kiểu dữ liệu

```ruby
mutate {
  convert => {
    "status_code" => "integer"
    "response_time" => "float"
    "is_error" => "boolean"
  }
}
```

Dùng khi field đang là string nhưng cần tính toán/filter đúng kiểu.

Ví dụ:

```json
{
  "status_code": "200"
}
```

Sau convert:

```json
{
  "status_code": 200
}
```

---

### 12.5 `lowercase` / `uppercase`

```ruby
mutate {
  lowercase => ["username", "service"]
  uppercase => ["level"]
}
```

Ví dụ:

```json
{
  "level": "error"
}
```

Sau uppercase:

```json
{
  "level": "ERROR"
}
```

---

### 12.6 `strip` — xóa khoảng trắng đầu/cuối

```ruby
mutate {
  strip => ["username", "service"]
}
```

Trước:

```json
{ "username": " admin " }
```

Sau:

```json
{ "username": "admin" }
```

---

### 12.7 `gsub` — thay thế chuỗi bằng regex

```ruby
mutate {
  gsub => [
    "message", "
", " ",
    "path", "\\", "/"
  ]
}
```

Ý nghĩa:

- Thay newline trong `message` bằng khoảng trắng.
- Thay dấu `\` trong đường dẫn Windows bằng `/`.

---

## 13. `grok` — parse log text thành field

`grok` dùng để tách một dòng log text thành nhiều field có cấu trúc.

### Ví dụ log

```text
2026-05-24 10:15:20 ERROR payment-service Cannot connect to database
```

### Grok filter

```ruby
grok {
  match => {
    "message" => "%{TIMESTAMP_ISO8601:log_time} %{LOGLEVEL:level} %{NOTSPACE:service} %{GREEDYDATA:log_detail}"
  }
}
```

### Output

```json
{
  "log_time": "2026-05-24 10:15:20",
  "level": "ERROR",
  "service": "payment-service",
  "log_detail": "Cannot connect to database"
}
```

---

## 14. Grok pattern thường gặp

| Pattern | Bắt được gì | Ví dụ |
|---|---|---|
| `%{WORD}` | Một từ | `ERROR`, `admin` |
| `%{NUMBER}` | Số | `200`, `10.5` |
| `%{INT}` | Số nguyên | `404`, `-1` |
| `%{IP}` | IP address | `192.168.1.10` |
| `%{HOSTNAME}` | Hostname/domain | `server-01` |
| `%{NOTSPACE}` | Chuỗi không có khoảng trắng | `/api/users` |
| `%{DATA}` | Chuỗi bất kỳ, bắt ngắn | `abc def` |
| `%{GREEDYDATA}` | Chuỗi bất kỳ, bắt hết phần còn lại | `full error message` |
| `%{LOGLEVEL}` | Level log | `INFO`, `ERROR`, `WARN` |
| `%{TIMESTAMP_ISO8601}` | Thời gian ISO | `2026-05-24 10:15:20` |
| `%{SYSLOGTIMESTAMP}` | Thời gian syslog | `May 24 10:15:20` |
| `%{HTTPDATE}` | Thời gian access log | `24/May/2026:10:15:20 +0700` |

### Cách nhớ nhanh

```text
WORD        = một từ
NUMBER      = số
IP          = địa chỉ IP
NOTSPACE    = chuỗi không có space
DATA        = lấy ít nhất có thể
GREEDYDATA  = lấy hết phần còn lại
```

---

## 15. `date` — parse thời gian log

Dùng khi log có thời gian riêng trong `message`, và muốn gán thời gian đó vào `@timestamp`.

### Ví dụ

Sau khi grok có field:

```json
{
  "log_time": "2026-05-24 10:15:20"
}
```

Dùng date filter:

```ruby
date {
  match => ["log_time", "yyyy-MM-dd HH:mm:ss"]
  target => "@timestamp"
  timezone => "Asia/Ho_Chi_Minh"
}
```

Ý nghĩa:

- Lấy `log_time` làm thời gian chính.
- Ghi vào `@timestamp`.
- Hiểu thời gian gốc theo múi giờ Việt Nam.

---

## 16. `json` — parse log dạng JSON

Nếu message là JSON:

```json
{"level":"ERROR","service":"api","msg":"timeout","status":500}
```

Dùng:

```ruby
json {
  source => "message"
}
```

Output:

```json
{
  "level": "ERROR",
  "service": "api",
  "msg": "timeout",
  "status": 500
}
```

Nếu muốn parse vào object riêng:

```ruby
json {
  source => "message"
  target => "app"
}
```

Output:

```json
{
  "app": {
    "level": "ERROR",
    "service": "api",
    "msg": "timeout",
    "status": 500
  }
}
```

---

## 17. `kv` — parse key=value

Dùng khi log có dạng:

```text
level=ERROR service=api status=500 msg=timeout
```

Config:

```ruby
kv {
  source => "message"
  field_split => " "
  value_split => "="
}
```

Output:

```json
{
  "level": "ERROR",
  "service": "api",
  "status": "500",
  "msg": "timeout"
}
```

Có thể thêm convert:

```ruby
mutate {
  convert => {
    "status" => "integer"
  }
}
```

---

## 18. `drop` — bỏ event không cần thiết

Dùng để loại log rác, log debug, health check, hoặc log không cần lưu.

```ruby
filter {
  if "healthcheck" in [message] {
    drop {}
  }
}
```

Ví dụ:

```text
GET /healthcheck 200
```

Event này sẽ bị bỏ, không gửi ra output.

---

## 19. `if / else` — điều kiện trong Logstash

### So sánh bằng

```ruby
if [level] == "ERROR" {
  mutate { add_field => { "severity" => "high" } }
}
```

### Kiểm tra chứa chuỗi

```ruby
if "timeout" in [message] {
  mutate { add_tag => ["timeout"] }
}
```

### Regex

```ruby
if [message] =~ /error|failed|timeout/i {
  mutate { add_field => { "needs_review" => "true" } }
}
```

### And / Or

```ruby
if [level] == "ERROR" and [environment] == "production" {
  mutate { add_field => { "priority" => "high" } }
}
```

```ruby
if [status] == 500 or [status] == 503 {
  mutate { add_tag => ["server_error"] }
}
```

### Field tồn tại

```ruby
if [user_id] {
  mutate { add_field => { "has_user" => "true" } }
}
```

---

## 20. `ruby` — xử lý logic phức tạp

Ruby filter dùng khi mutate/if thông thường không đủ.

### Lệnh Ruby hay dùng

| Cú pháp | Công dụng |
|---|---|
| `event.get("field")` | Lấy giá trị field |
| `event.get("[a][b]")` | Lấy nested field |
| `event.set("field", value)` | Tạo hoặc cập nhật field |
| `event.remove("field")` | Xóa field |
| `event.tag("tag_name")` | Thêm tag |

### Ví dụ tạo field dựa trên status code

```ruby
ruby {
  code => '
    status = event.get("status")

    if status && status.to_i >= 500
      event.set("error_type", "server_error")
    elsif status && status.to_i >= 400
      event.set("error_type", "client_error")
    else
      event.set("error_type", "normal")
    end
  '
}
```

---

## 21. `translate` — map giá trị

Dùng để đổi một giá trị code thành ý nghĩa dễ hiểu.

Ví dụ event có:

```json
{ "status": 404 }
```

Config:

```ruby
translate {
  field => "status"
  destination => "status_text"
  dictionary => {
    "200" => "OK"
    "400" => "Bad Request"
    "401" => "Unauthorized"
    "403" => "Forbidden"
    "404" => "Not Found"
    "500" => "Internal Server Error"
  }
  fallback => "Unknown"
}
```

Output:

```json
{
  "status": 404,
  "status_text": "Not Found"
}
```

---

## 22. `geoip` — enrich vị trí theo IP

Dùng khi có field IP và muốn lấy quốc gia/thành phố/toạ độ.

```ruby
geoip {
  source => "client_ip"
  target => "geoip"
}
```

Input:

```json
{
  "client_ip": "8.8.8.8"
}
```

Output có thể có:

```json
{
  "geoip": {
    "country_name": "United States",
    "city_name": "Mountain View",
    "location": {
      "lat": 37.4056,
      "lon": -122.0775
    }
  }
}
```

---

## 23. Output thường gặp

### 23.1 `stdout` — debug ra màn hình

```ruby
output {
  stdout { codec => rubydebug }
}
```

Dùng khi test config.

---

### 23.2 `file` — ghi ra file

```ruby
output {
  file {
    path => "/tmp/logstash-output.log"
  }
}
```

Dùng khi muốn kiểm tra output trước khi gửi vào OpenSearch.

---

### 23.3 `elasticsearch` hoặc `opensearch`

```ruby
output {
  opensearch {
    hosts => ["https://localhost:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

Dùng để gửi document vào OpenSearch.

---

## 24. Các field nên giữ / nên xóa

### Nên giữ

| Field | Lý do |
|---|---|
| `@timestamp` | Search theo thời gian |
| `message` | Giữ log gốc để debug |
| `level` | Filter theo log level |
| `service` | Biết log thuộc service nào |
| `environment` | Phân biệt dev/staging/prod |
| `host.name` hoặc `server_name` | Biết log từ máy nào |
| `log.file.path` | Biết log từ file nào |
| `tags` | Filter nhanh |
| `status` | Dùng cho HTTP/API log |
| `client_ip` | Dùng cho web/network log |

### Có thể xóa

| Field | Lý do |
|---|---|
| `ecs` | Thường không cần khi mới học |
| `@version` | Ít giá trị phân tích |
| `agent.ephemeral_id` | Noise, thay đổi theo lần chạy |
| `log.offset` | Chủ yếu dùng để debug Filebeat |
| `input.type` | Ít dùng nếu đã có `log_type` |

Ví dụ cleanup:

```ruby
mutate {
  remove_field => [
    "ecs",
    "@version",
    "[agent][ephemeral_id]",
    "[log][offset]"
  ]
}
```

---

## 25. Những lỗi parse thường gặp

| Tag lỗi | Nguyên nhân thường gặp | Cách xử lý |
|---|---|---|
| `_grokparsefailure` | Grok pattern không khớp log | Xem lại `message`, sửa pattern |
| `_dateparsefailure` | Format date không đúng | Sửa `match` trong date filter |
| `_jsonparsefailure` | Message không phải JSON hợp lệ | Check format JSON |

### Debug nhanh grok lỗi

```ruby
output {
  stdout { codec => rubydebug }
}
```

Hoặc ghi log lỗi ra file:

```ruby
if "_grokparsefailure" in [tags] {
  file {
    path => "/tmp/grok-failed.log"
    codec => line { format => "%{message}" }
  }
}
```

---

## 26. Mini pipeline cơ bản dễ học

```ruby
input {
  beats {
    port => 5044
  }
}

filter {
  grok {
    match => {
      "message" => "%{TIMESTAMP_ISO8601:log_time} %{LOGLEVEL:level} %{NOTSPACE:service} %{GREEDYDATA:detail}"
    }
  }

  date {
    match => ["log_time", "yyyy-MM-dd HH:mm:ss"]
    target => "@timestamp"
    timezone => "Asia/Ho_Chi_Minh"
  }

  mutate {
    lowercase => ["service"]
    uppercase => ["level"]
    add_field => {
      "processed_by" => "logstash"
    }
  }

  if [level] == "ERROR" {
    mutate {
      add_field => { "severity" => "high" }
      add_tag => ["need_check"]
    }
  }

  mutate {
    remove_field => ["ecs", "@version"]
  }
}

output {
  stdout { codec => rubydebug }

  opensearch {
    hosts => ["https://localhost:9200"]
    index => "basic-logs-%{+YYYY.MM.dd}"
  }
}
```

---

## 27. Công thức nhớ nhanh

```text
input      = nhận log từ đâu
filter     = xử lý log như thế nào
output     = gửi log đi đâu

message    = log gốc
@timestamp = thời gian chính
host.name  = máy gửi log
log.file.path = file log nguồn
tags       = nhãn của event
@metadata  = biến tạm không lưu ra output

mutate     = thêm/sửa/xóa/đổi kiểu field
grok       = parse text log
json       = parse JSON log
kv         = parse key=value log
date       = parse thời gian
drop       = bỏ event
ruby       = logic nâng cao
translate  = map code sang text
geoip      = enrich IP thành vị trí

%{field}        = lấy giá trị field
%{+YYYY.MM.dd}  = lấy ngày từ @timestamp
[field][child]  = gọi field lồng nhau
```

---

## 28. Thứ tự học nên đi theo

```text
1. Hiểu event, message, @timestamp
2. Hiểu input/filter/output
3. Học mutate
4. Học if/else
5. Học grok cơ bản
6. Học date filter
7. Học json và kv
8. Học tags và @metadata
9. Học output index động
10. Học debug lỗi _grokparsefailure / _dateparsefailure
```

Nếu mới học, nên tập trung 80% vào:

```text
message
@timestamp
mutate
grok
date
if/else
tags
stdout rubydebug
```

Vì đây là những thứ dùng nhiều nhất trong thực tế.

