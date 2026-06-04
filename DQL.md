# OpenSearch Query Languages: DSL, DQL, Vega & PPL

> Tài liệu tham khảo toàn diện về các ngôn ngữ truy vấn trong OpenSearch — từ cơ bản đến nâng cao.

---

## Mục lục

1. [Tổng quan về OpenSearch](#1-tổng-quan-về-opensearch)
2. [So sánh các ngôn ngữ truy vấn](#2-so-sánh-các-ngôn-ngữ-truy-vấn)
3. [Query DSL](#3-query-dsl)
   - [Cấu trúc cơ bản](#31-cấu-trúc-cơ-bản)
   - [Leaf Queries](#32-leaf-queries)
   - [Compound Queries](#33-compound-queries)
   - [Full-Text Queries](#34-full-text-queries)
   - [Aggregations](#35-aggregations)
   - [Sorting & Pagination](#36-sorting--pagination)
4. [DQL — Dashboards Query Language](#4-dql--dashboards-query-language)
   - [Cú pháp cơ bản](#41-cú-pháp-cơ-bản)
   - [Tìm kiếm theo field](#42-tìm-kiếm-theo-field)
   - [Boolean operators](#43-boolean-operators)
   - [Range queries](#44-range-queries)
   - [Wildcards & Phrases](#45-wildcards--phrases)
   - [Nested & Object fields](#46-nested--object-fields)
5. [Vega & Vega-Lite](#5-vega--vega-lite)
   - [Tổng quan](#51-tổng-quan)
   - [Cấu trúc Vega Spec](#52-cấu-trúc-vega-spec)
   - [Kết nối dữ liệu OpenSearch](#53-kết-nối-dữ-liệu-opensearch)
   - [Các loại biểu đồ phổ biến](#54-các-loại-biểu-đồ-phổ-biến)
   - [Vega-Lite (cú pháp rút gọn)](#55-vega-lite-cú-pháp-rút-gọn)
6. [PPL — Piped Processing Language](#6-ppl--piped-processing-language)
   - [Cú pháp và Pipeline](#61-cú-pháp-và-pipeline)
   - [Các lệnh cốt lõi](#62-các-lệnh-cốt-lõi)
   - [Functions](#63-functions)
   - [Ví dụ thực tế](#64-ví-dụ-thực-tế)
7. [Khi nào dùng ngôn ngữ nào?](#7-khi-nào-dùng-ngôn-ngữ-nào)
8. [Tài nguyên tham khảo](#8-tài-nguyên-tham-khảo)

---

## 1. Tổng quan về OpenSearch

OpenSearch là một bộ phần mềm mã nguồn mở dựa trên nền tảng Apache Lucene, bao gồm:

- **OpenSearch** — Search engine lưu trữ và tìm kiếm dữ liệu dạng document JSON
- **OpenSearch Dashboards** — Giao diện trực quan hóa và phân tích dữ liệu

OpenSearch lưu dữ liệu theo mô hình **Index → Shards → Documents**. Mỗi document là một JSON object được đánh index theo các field. OpenSearch phiên bản mới nhất (3.6.0, tháng 4/2026) đang được phát triển bởi OpenSearch Software Foundation (thuộc Linux Foundation).

**Các ngôn ngữ truy vấn chính:**

| Ngôn ngữ | Dùng ở đâu | Đặc điểm |
|---|---|---|
| **Query DSL** | REST API | JSON, mạnh nhất, dùng cho lập trình |
| **DQL** | Dashboards search bar | Text-based, nhanh, dành cho UI |
| **Vega / Vega-Lite** | Dashboards Visualize | Declarative grammar, custom charts |
| **PPL** | Query Workbench / Observability | Pipe syntax, thân thiện với log analytics |
| **SQL** | Query Workbench | Cú pháp SQL truyền thống |

---

## 2. So sánh các ngôn ngữ truy vấn

```
┌─────────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│             │  Query DSL   │     DQL      │    Vega      │     PPL      │
├─────────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ Format      │ JSON         │ Text         │ JSON spec    │ Pipe syntax  │
│ Dùng qua    │ REST API     │ Search bar   │ Visualize UI │ Workbench/API│
│ Độ phức tạp │ Cao          │ Thấp         │ Cao          │ Trung bình   │
│ Mục đích    │ Search+Agg   │ Filter UI    │ Custom chart │ Log Analysis │
│ Read-only?  │ Không        │ Có           │ Có           │ Có           │
│ Phù hợp     │ Developer    │ Analyst/User │ Developer    │ DevOps/SRE   │
└─────────────┴──────────────┴──────────────┴──────────────┴──────────────┘
```

---

## 3. Query DSL

**Query DSL (Domain-Specific Language)** là ngôn ngữ truy vấn chính của OpenSearch, dùng qua HTTP REST API với body dạng JSON. Đây là ngôn ngữ mạnh nhất và linh hoạt nhất trong hệ sinh thái OpenSearch.

### 3.1 Cấu trúc cơ bản

Mọi request đều gửi đến endpoint `GET /<index>/_search` với body JSON:

```json
GET /my-index/_search
{
  "query": { ... },
  "size": 10,
  "from": 0,
  "sort": [ ... ],
  "aggs": { ... },
  "_source": ["field1", "field2"]
}
```

**Phân loại query:**

- **Leaf queries**: Tìm kiếm một giá trị trong một hoặc nhiều field cụ thể (`match`, `term`, `range`, ...)
- **Compound queries**: Kết hợp nhiều query lại với nhau (`bool`, `dis_max`, ...)

---

### 3.2 Leaf Queries

#### `match_all` — Lấy tất cả documents

```json
GET /logs/_search
{
  "query": {
    "match_all": {}
  }
}
```

#### `term` — Tìm giá trị chính xác (không phân tích)

Dùng cho các field kiểu `keyword`, `boolean`, `integer`.

```json
GET /logs/_search
{
  "query": {
    "term": {
      "status.keyword": "ERROR"
    }
  }
}
```

> **Lưu ý:** Không dùng `term` với field kiểu `text` vì text đã bị tokenize. Dùng `term` với `field.keyword`.

#### `terms` — Tìm nhiều giá trị

```json
GET /logs/_search
{
  "query": {
    "terms": {
      "http_method.keyword": ["GET", "POST", "DELETE"]
    }
  }
}
```

#### `range` — Tìm theo khoảng giá trị

```json
GET /logs/_search
{
  "query": {
    "range": {
      "response_time_ms": {
        "gte": 500,
        "lte": 5000
      }
    }
  }
}
```

Với kiểu ngày giờ:

```json
GET /logs/_search
{
  "query": {
    "range": {
      "@timestamp": {
        "gte": "2025-01-01T00:00:00",
        "lte": "2025-12-31T23:59:59",
        "format": "yyyy-MM-dd'T'HH:mm:ss"
      }
    }
  }
}
```

Toán tử range: `gt` (>), `gte` (>=), `lt` (<), `lte` (<=).

#### `exists` — Kiểm tra field tồn tại

```json
GET /logs/_search
{
  "query": {
    "exists": {
      "field": "user_id"
    }
  }
}
```

#### `wildcard` — Tìm kiếm với ký tự đại diện

- `*` — 0 hoặc nhiều ký tự
- `?` — đúng 1 ký tự

```json
GET /logs/_search
{
  "query": {
    "wildcard": {
      "service_name.keyword": {
        "value": "payment-*",
        "case_insensitive": true
      }
    }
  }
}
```

#### `regexp` — Biểu thức chính quy

```json
GET /logs/_search
{
  "query": {
    "regexp": {
      "ip_address.keyword": "192\\.168\\.[0-9]+\\.[0-9]+"
    }
  }
}
```

#### `ids` — Tìm theo document ID

```json
GET /logs/_search
{
  "query": {
    "ids": {
      "values": ["doc-001", "doc-002", "doc-003"]
    }
  }
}
```

#### `fuzzy` — Tìm kiếm gần đúng (cho phép sai chính tả)

```json
GET /products/_search
{
  "query": {
    "fuzzy": {
      "name": {
        "value": "iPhon",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

---

### 3.3 Compound Queries

#### `bool` — Kết hợp logic

Đây là compound query quan trọng nhất, với 4 clause:

| Clause | Ý nghĩa | Ảnh hưởng score |
|---|---|---|
| `must` | Bắt buộc phải khớp | Có |
| `filter` | Bắt buộc, nhưng không tính score | Không |
| `should` | Nên khớp (tăng score nếu khớp) | Có |
| `must_not` | Không được khớp | Không |

```json
GET /logs/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "service": "payment" } }
      ],
      "filter": [
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h",
              "lte": "now"
            }
          }
        }
      ],
      "should": [
        { "term": { "level": "ERROR" } },
        { "term": { "level": "WARN" } }
      ],
      "must_not": [
        { "term": { "endpoint.keyword": "/health" } }
      ],
      "minimum_should_match": 1
    }
  }
}
```

> `minimum_should_match`: chỉ định ít nhất bao nhiêu `should` clause phải khớp. Mặc định là 0 nếu có `must` hoặc `filter`, là 1 nếu không có.

#### `dis_max` — Chọn score cao nhất trong các sub-query

```json
GET /products/_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "title": "laptop gaming" } },
        { "match": { "description": "laptop gaming" } }
      ],
      "tie_breaker": 0.3
    }
  }
}
```

#### `nested` — Tìm kiếm trong nested objects

```json
GET /orders/_search
{
  "query": {
    "nested": {
      "path": "items",
      "query": {
        "bool": {
          "must": [
            { "term": { "items.product_id": "SKU-123" } },
            { "range": { "items.quantity": { "gte": 2 } } }
          ]
        }
      }
    }
  }
}
```

---

### 3.4 Full-Text Queries

#### `match` — Tìm kiếm full-text

Phân tích text đầu vào (tokenize, lowercase, ...) trước khi tìm.

```json
GET /articles/_search
{
  "query": {
    "match": {
      "content": {
        "query": "opensearch log analytics",
        "operator": "AND"
      }
    }
  }
}
```

- `operator: "OR"` (mặc định): khớp bất kỳ từ nào
- `operator: "AND"`: tất cả các từ phải xuất hiện

#### `match_phrase` — Tìm cụm từ chính xác

```json
GET /articles/_search
{
  "query": {
    "match_phrase": {
      "content": "distributed search engine"
    }
  }
}
```

#### `multi_match` — Tìm trên nhiều field

```json
GET /articles/_search
{
  "query": {
    "multi_match": {
      "query": "opensearch tutorial",
      "fields": ["title^3", "content", "tags^2"],
      "type": "best_fields"
    }
  }
}
```

> `^3` là **field boosting** — field `title` được tính điểm cao gấp 3 lần.

Các `type` của multi_match:

| Type | Mô tả |
|---|---|
| `best_fields` | Lấy score cao nhất từ 1 field (mặc định) |
| `most_fields` | Cộng score từ nhiều field |
| `cross_fields` | Xem tất cả field như 1 field lớn |
| `phrase` | Tìm cụm từ |

#### `query_string` — Cú pháp Lucene đầy đủ

```json
GET /logs/_search
{
  "query": {
    "query_string": {
      "query": "level:ERROR AND service:(payment OR order) AND NOT endpoint:/health",
      "default_field": "message"
    }
  }
}
```

---

### 3.5 Aggregations

Aggregations cho phép phân tích và thống kê dữ liệu.

#### Bucket Aggregations — Nhóm dữ liệu

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "by_service": {
      "terms": {
        "field": "service.keyword",
        "size": 10
      },
      "aggs": {
        "by_level": {
          "terms": {
            "field": "level.keyword"
          }
        }
      }
    }
  }
}
```

#### Date Histogram — Phân tích theo thời gian

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "errors_over_time": {
      "date_histogram": {
        "field": "@timestamp",
        "calendar_interval": "1h",
        "time_zone": "Asia/Ho_Chi_Minh"
      },
      "aggs": {
        "error_count": {
          "filter": { "term": { "level.keyword": "ERROR" } }
        }
      }
    }
  }
}
```

#### Metric Aggregations — Tính toán thống kê

```json
GET /logs/_search
{
  "size": 0,
  "aggs": {
    "response_stats": {
      "stats": {
        "field": "response_time_ms"
      }
    },
    "p95_latency": {
      "percentiles": {
        "field": "response_time_ms",
        "percents": [50, 90, 95, 99]
      }
    },
    "avg_response": {
      "avg": { "field": "response_time_ms" }
    }
  }
}
```

#### Pipeline Aggregations

```json
GET /sales/_search
{
  "size": 0,
  "aggs": {
    "monthly_sales": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "total": { "sum": { "field": "amount" } }
      }
    },
    "sales_derivative": {
      "derivative": {
        "buckets_path": "monthly_sales>total"
      }
    }
  }
}
```

---

### 3.6 Sorting & Pagination

```json
GET /logs/_search
{
  "query": { "match_all": {} },
  "sort": [
    { "@timestamp": { "order": "desc" } },
    { "response_time_ms": { "order": "asc" } },
    "_score"
  ],
  "from": 0,
  "size": 20,
  "_source": ["@timestamp", "level", "message", "service"]
}
```

**Pagination sâu (deep pagination):** Dùng `search_after` thay vì `from/size` để tránh quá tải bộ nhớ:

```json
GET /logs/_search
{
  "query": { "match_all": {} },
  "sort": [{ "@timestamp": "desc" }, { "_id": "asc" }],
  "size": 100,
  "search_after": ["2025-06-01T12:00:00.000Z", "abc123"]
}
```

---

## 4. DQL — Dashboards Query Language

**DQL (Dashboards Query Language)** là ngôn ngữ tìm kiếm dạng text được dùng trực tiếp trên thanh tìm kiếm trong **OpenSearch Dashboards** (Discover và Dashboard panels). DQL được thiết kế đơn giản, không cần viết JSON.

Mặc định Dashboards dùng DQL. Để chuyển sang Lucene query string, nhấn nút **DQL** bên cạnh search box và toggle off.

### 4.1 Cú pháp cơ bản

```
<search term>                         # Tìm toàn văn bản
<field>:<value>                       # Tìm trong field cụ thể
<field>:<value1> or <field>:<value2>  # Boolean OR
<field>:<value1> and <field>:<value2> # Boolean AND
not <field>:<value>                   # Boolean NOT
```

Tìm kiếm đơn giản nhất — chỉ gõ từ khóa:

```
error
payment timeout
```

DQL không phân biệt hoa/thường — `ERROR` và `error` cho cùng kết quả.

### 4.2 Tìm kiếm theo field

```
# Syntax: field:value
level:ERROR
service:payment
http_status:500
```

Với `keyword` field (exact match):
```
status.keyword:active
```

Với `text` field (analyzed):
```
message:connection refused
```

> **Phân biệt text vs keyword:** Field `text` được phân tích (tokenize, lowercase), field `keyword` lưu giá trị chính xác. Ví dụ field `service` kiểu `keyword` thì `service:Payment` khác `service:payment`, nhưng field `message` kiểu `text` thì không phân biệt hoa thường.

### 4.3 Boolean operators

```
# AND — cả hai điều kiện phải đúng
level:ERROR and service:payment

# OR — ít nhất một điều kiện đúng
level:ERROR or level:WARN

# NOT — loại trừ
level:ERROR and not service:health-check

# Kết hợp phức tạp — dùng ngoặc đơn
(level:ERROR or level:WARN) and service:payment and not endpoint:/metrics
```

**Thứ tự ưu tiên:** `NOT` > `AND` > `OR`

```
# Không nên viết thế này vì dễ nhầm:
service:payment or level:ERROR and response_time > 1000

# Nên dùng ngoặc để rõ ràng:
service:payment or (level:ERROR and response_time > 1000)
```

### 4.4 Range queries

```
# Số
response_time_ms >= 1000
response_time_ms > 500 and response_time_ms <= 2000
bytes_sent < 1024

# Ngày giờ
@timestamp > "2025-01-01"
@timestamp >= "2025-06-01T00:00:00" and @timestamp < "2025-07-01T00:00:00"
```

### 4.5 Wildcards & Phrases

**Wildcard** — chỉ dùng `*` (không hỗ trợ `?`):

```
# Tìm tất cả service bắt đầu bằng "payment"
service:payment*

# Tìm tất cả field chứa "error"
*error*: true

# Kiểm tra field tồn tại
user_id:*
```

**Phrase search** — tìm cụm từ chính xác bằng dấu nháy kép:

```
message:"connection timeout"
message:"failed to connect to database"
```

**Escape ký tự đặc biệt** — dùng dấu backslash `\`:

```
# Tìm chuỗi "2*3" (dấu * là literal)
value:2\*3
```

Danh sách ký tự cần escape trong DQL: `\`, `*`, `(`, `)`, `:`, `<`, `>`, `"`, `/`

### 4.6 Nested & Object fields

```
# Object field — dùng dấu chấm
geo.country:Vietnam
geo.coordinates.lat > 10

# Nested field — tương tự
items.product_id:SKU-001
user.address.city:Hanoi
```

---

## 5. Vega & Vega-Lite

### 5.1 Tổng quan

**Vega** và **Vega-Lite** là các ngôn ngữ khai báo (declarative grammar) mã nguồn mở để tạo các biểu đồ tùy chỉnh trong OpenSearch Dashboards. Chúng cho phép xây dựng các visualization phức tạp mà các loại chart có sẵn không đáp ứng được.

| | Vega | Vega-Lite |
|---|---|---|
| Độ phức tạp | Cao | Thấp hơn |
| Kiểm soát | Toàn bộ mọi chi tiết | Tự động hóa nhiều thứ |
| Code lượng | Nhiều | Ít hơn |
| Use case | Biểu đồ tùy biến sâu | Biểu đồ nhanh, thông dụng |

Để tạo Vega visualization trong Dashboards: **Visualize → Create Visualization → Vega**

### 5.2 Cấu trúc Vega Spec

Một Vega specification JSON bao gồm các thành phần chính:

```json
{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "width": 600,
  "height": 300,
  "padding": 5,

  "data": [ ... ],          // Nguồn dữ liệu
  "scales": [ ... ],        // Ánh xạ dữ liệu → tọa độ/màu sắc
  "axes": [ ... ],          // Trục x, y
  "marks": [ ... ],         // Hình vẽ thực tế (bar, line, point...)
  "legends": [ ... ],       // Chú thích
  "signals": [ ... ]        // Biến tương tác
}
```

**Thành phần `marks` (các loại hình):**

| Mark Type | Mô tả |
|---|---|
| `rect` | Hình chữ nhật (bar chart) |
| `line` | Đường kẻ (line chart) |
| `symbol` | Điểm/hình (scatter plot) |
| `area` | Vùng tô màu (area chart) |
| `text` | Văn bản nhãn |
| `arc` | Cung tròn (pie/donut) |
| `rule` | Đường thẳng (reference line) |
| `image` | Hình ảnh |

### 5.3 Kết nối dữ liệu OpenSearch

OpenSearch Dashboards mở rộng Vega với các biến đặc biệt để truy vấn dữ liệu trực tiếp từ OpenSearch:

| Biến đặc biệt | Ý nghĩa |
|---|---|
| `%context%` | Áp dụng filter từ Dashboard hiện tại |
| `%timefield%` | Field thời gian để lọc theo time picker |
| `%timefilter%` | Filter thời gian từ Dashboard |

#### Ví dụ: Lấy dữ liệu từ OpenSearch bằng DSL aggregation

```json
{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "data": [
    {
      "name": "table",
      "url": {
        "%context%": true,
        "%timefield%": "@timestamp",
        "index": "logs-*",
        "body": {
          "size": 0,
          "aggs": {
            "over_time": {
              "date_histogram": {
                "field": "@timestamp",
                "calendar_interval": "1h"
              },
              "aggs": {
                "error_count": {
                  "filter": {
                    "term": { "level.keyword": "ERROR" }
                  }
                }
              }
            }
          }
        }
      },
      "format": {
        "property": "aggregations.over_time.buckets"
      },
      "transform": [
        {
          "type": "formula",
          "as": "time",
          "expr": "toDate(datum.key)"
        },
        {
          "type": "formula",
          "as": "count",
          "expr": "datum.error_count.doc_count"
        }
      ]
    }
  ]
}
```

#### Ví dụ: Lấy dữ liệu bằng PPL (OpenSearch 2.x+)

```json
{
  "data": [
    {
      "name": "ppl_data",
      "url": {
        "method": "POST",
        "path": "/_plugins/_ppl",
        "body": {
          "query": "source=logs-* | where level='ERROR' | stats count() by service"
        }
      },
      "format": {
        "type": "json",
        "property": "datarows"
      }
    }
  ]
}
```

### 5.4 Các loại biểu đồ phổ biến

#### Bar Chart — Biểu đồ cột

```json
{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "width": 500,
  "height": 300,
  "data": [
    {
      "name": "table",
      "url": {
        "%context%": true,
        "index": "logs-*",
        "body": {
          "size": 0,
          "aggs": {
            "by_service": {
              "terms": { "field": "service.keyword", "size": 10 }
            }
          }
        }
      },
      "format": { "property": "aggregations.by_service.buckets" }
    }
  ],
  "scales": [
    {
      "name": "xscale",
      "type": "band",
      "domain": { "data": "table", "field": "key" },
      "range": "width",
      "padding": 0.1
    },
    {
      "name": "yscale",
      "type": "linear",
      "domain": { "data": "table", "field": "doc_count" },
      "range": "height",
      "nice": true
    }
  ],
  "axes": [
    { "orient": "bottom", "scale": "xscale", "labelAngle": -45 },
    { "orient": "left", "scale": "yscale", "title": "Count" }
  ],
  "marks": [
    {
      "type": "rect",
      "from": { "data": "table" },
      "encode": {
        "enter": {
          "x": { "scale": "xscale", "field": "key" },
          "width": { "scale": "xscale", "band": 1 },
          "y": { "scale": "yscale", "field": "doc_count" },
          "y2": { "scale": "yscale", "value": 0 },
          "fill": { "value": "steelblue" }
        },
        "hover": {
          "fill": { "value": "orange" }
        }
      }
    }
  ]
}
```

#### Line Chart — Biểu đồ đường thời gian

```json
{
  "$schema": "https://vega.github.io/schema/vega/v5.json",
  "width": 700,
  "height": 300,
  "data": [
    {
      "name": "timeseries",
      "url": {
        "%context%": true,
        "%timefield%": "@timestamp",
        "index": "metrics-*",
        "body": {
          "size": 0,
          "aggs": {
            "over_time": {
              "date_histogram": {
                "field": "@timestamp",
                "calendar_interval": "5m"
              },
              "aggs": {
                "avg_latency": { "avg": { "field": "response_time_ms" } }
              }
            }
          }
        }
      },
      "format": { "property": "aggregations.over_time.buckets" },
      "transform": [
        { "type": "formula", "as": "ts", "expr": "toDate(datum.key)" },
        { "type": "formula", "as": "latency", "expr": "datum.avg_latency.value" }
      ]
    }
  ],
  "scales": [
    {
      "name": "x",
      "type": "time",
      "domain": { "data": "timeseries", "field": "ts" },
      "range": "width"
    },
    {
      "name": "y",
      "type": "linear",
      "domain": { "data": "timeseries", "field": "latency" },
      "range": "height",
      "nice": true,
      "zero": true
    }
  ],
  "axes": [
    { "orient": "bottom", "scale": "x", "format": "%H:%M", "title": "Time" },
    { "orient": "left", "scale": "y", "title": "Avg Latency (ms)" }
  ],
  "marks": [
    {
      "type": "line",
      "from": { "data": "timeseries" },
      "encode": {
        "enter": {
          "x": { "scale": "x", "field": "ts" },
          "y": { "scale": "y", "field": "latency" },
          "stroke": { "value": "#1f77b4" },
          "strokeWidth": { "value": 2 }
        }
      }
    }
  ]
}
```

### 5.5 Vega-Lite (cú pháp rút gọn)

Vega-Lite đơn giản hơn Vega nhiều, phù hợp cho phần lớn nhu cầu thông thường:

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "url": {
      "%context%": true,
      "%timefield%": "@timestamp",
      "index": "logs-*",
      "body": {
        "size": 0,
        "aggs": {
          "by_level": {
            "terms": { "field": "level.keyword" }
          }
        }
      }
    },
    "format": { "property": "aggregations.by_level.buckets" }
  },
  "mark": "bar",
  "encoding": {
    "x": {
      "field": "key",
      "type": "nominal",
      "title": "Log Level"
    },
    "y": {
      "field": "doc_count",
      "type": "quantitative",
      "title": "Count"
    },
    "color": {
      "field": "key",
      "type": "nominal",
      "scale": {
        "domain": ["ERROR", "WARN", "INFO", "DEBUG"],
        "range": ["#e41a1c", "#ff7f00", "#4daf4a", "#377eb8"]
      }
    },
    "tooltip": [
      { "field": "key", "title": "Level" },
      { "field": "doc_count", "title": "Count" }
    ]
  }
}
```

---

## 6. PPL — Piped Processing Language

**PPL (Piped Processing Language)** là ngôn ngữ truy vấn dạng pipe, lấy cảm hứng từ Unix pipe và Splunk SPL. PPL được thiết kế đặc biệt cho việc phân tích **observability data** — logs, metrics, traces — với cú pháp đơn giản, dễ đọc.

PPL được dùng trong:
- **Query Workbench** trong OpenSearch Dashboards
- **Observability → Event Analytics**
- REST API endpoint `POST /_plugins/_ppl`
- Amazon CloudWatch Logs (phiên bản giới hạn)

**Đặc điểm nổi bật:**
- Read-only — không thể sửa/xóa dữ liệu
- Không có learning curve dốc như DSL
- Mỗi lệnh nhận đầu vào và truyền kết quả sang lệnh tiếp theo qua `|`
- Xử lý semi-structured data tốt (logs, JSON lồng nhau)

### 6.1 Cú pháp và Pipeline

```
search source=<index> [<filter>]
| command1 [options]
| command2 [options]
| ...
```

Dữ liệu chạy **từ trái sang phải** qua mỗi stage trong pipeline.

```
source=logs-*                     # Nguồn dữ liệu (bắt buộc)
| where level = 'ERROR'           # Lọc
| fields timestamp, service, message  # Chọn field
| sort - timestamp                # Sắp xếp
| head 100                        # Giới hạn kết quả
```

### 6.2 Các lệnh cốt lõi

#### `search` / `source` — Xác định nguồn dữ liệu

```ppl
search source=logs-*
search source=logs-* level = 'ERROR'

# Dùng source thay search (tương đương)
source=logs-* | where level = 'ERROR'
```

#### `where` — Lọc dữ liệu

```ppl
source=logs-*
| where level = 'ERROR'
| where response_time_ms > 1000
| where service = 'payment' and method = 'POST'
| where timestamp >= '2025-01-01 00:00:00'
```

**Toán tử so sánh:** `=`, `!=`, `>`, `>=`, `<`, `<=`, `LIKE`, `NOT LIKE`, `IN`, `NOT IN`, `BETWEEN`, `IS NULL`, `IS NOT NULL`

```ppl
source=logs-*
| where message LIKE '%timeout%'
| where service IN ('payment', 'order', 'user')
| where response_time_ms BETWEEN 500 AND 2000
| where error_code IS NOT NULL
```

#### `fields` — Chọn/loại trừ fields

```ppl
source=logs-*
| fields timestamp, service, level, message

# Loại trừ field
source=logs-*
| fields - password, secret_key, internal_id
```

#### `stats` — Thống kê và aggregation

```ppl
# Đếm tổng
source=logs-* | stats count()

# Đếm theo nhóm
source=logs-*
| stats count() by service

# Nhiều metric
source=logs-*
| stats count() as total,
         count() as errors where level = 'ERROR',
         avg(response_time_ms) as avg_latency,
         max(response_time_ms) as max_latency,
         min(response_time_ms) as min_latency
  by service

# Groupby theo thời gian
source=metrics-*
| stats avg(cpu_usage) as avg_cpu by span(timestamp, 5m), host
```

**Các hàm stats:**

| Hàm | Mô tả |
|---|---|
| `count()` | Đếm số document |
| `count(field)` | Đếm số document có field không null |
| `sum(field)` | Tổng |
| `avg(field)` | Trung bình |
| `min(field)` | Giá trị nhỏ nhất |
| `max(field)` | Giá trị lớn nhất |
| `stddev(field)` | Độ lệch chuẩn |
| `percentile(field, N)` | Phân vị thứ N |
| `dc(field)` | Distinct count |
| `first(field)` | Giá trị đầu tiên |
| `last(field)` | Giá trị cuối cùng |
| `list(field)` | Danh sách các giá trị |

#### `sort` — Sắp xếp

```ppl
source=logs-*
| sort timestamp             # Tăng dần
| sort - timestamp           # Giảm dần
| sort - response_time_ms, + service  # Nhiều field
```

#### `head` — Giới hạn số kết quả

```ppl
source=logs-*
| where level = 'ERROR'
| sort - timestamp
| head 50                    # Lấy 50 kết quả đầu
```

#### `dedup` — Loại bỏ trùng lặp

```ppl
source=logs-*
| dedup service              # Giữ lại 1 document cho mỗi service

# Giữ lại N bản ghi đầu tiên của mỗi nhóm
source=logs-*
| dedup 3 service            # Giữ 3 document đầu tiên cho mỗi service

# Giữ empty events
source=logs-*
| dedup service keepempty=true
```

#### `rename` — Đổi tên field

```ppl
source=logs-*
| rename response_time_ms as latency, @timestamp as time
```

#### `eval` — Tính toán, tạo field mới

```ppl
source=logs-*
| eval latency_seconds = response_time_ms / 1000
| eval is_slow = if(response_time_ms > 1000, true, false)
| eval log_type = upper(level)
| eval full_path = concat(host, ':', tostring(port), uri)
```

#### `top` — Tìm giá trị phổ biến nhất

```ppl
source=logs-*
| top 10 service              # Top 10 service xuất hiện nhiều nhất

source=logs-*
| where level = 'ERROR'
| top 5 error_code by service  # Top 5 error_code theo mỗi service
```

#### `rare` — Tìm giá trị hiếm nhất

```ppl
source=logs-*
| rare user_agent              # User agent ít gặp nhất

source=logs-*
| rare 5 ip_address by country  # 5 IP hiếm nhất theo quốc gia
```

#### `parse` — Trích xuất dữ liệu từ text bằng Grok / Regex

```ppl
# Dùng Grok pattern
source=logs-*
| parse message "%{IP:client_ip} - - \[%{HTTPDATE:timestamp}\] \"%{WORD:method} %{URIPATHPARAM:path}\""

# Dùng regex
source=logs-*
| parse message "user=(?P<username>\w+)"
```

#### `grok` — Pattern extraction nâng cao

```ppl
source=access-logs-*
| grok message "%{COMBINEDAPACHELOG}"
| fields client_ip, verb, request, response, bytes
```

### 6.3 Functions

#### String functions

```ppl
source=logs-*
| eval
    upper_service = upper(service),
    lower_service = lower(service),
    msg_length    = length(message),
    trimmed_msg   = trim(message),
    sub_msg       = substring(message, 1, 100),
    has_error     = like(message, '%error%'),
    replaced_msg  = replace(message, 'ERROR', '[ERR]'),
    split_parts   = split(service, '-'),
    concat_info   = concat(service, ':', level)
```

#### Date/Time functions

```ppl
source=logs-*
| eval
    hour_of_day    = hour(timestamp),
    day_of_week    = dayofweek(timestamp),
    day_name       = date_format(timestamp, 'EEEE'),
    unix_ts        = unix_timestamp(timestamp),
    date_only      = date(timestamp),
    relative_time  = timestampdiff(SECOND, timestamp, now())
```

#### Math functions

```ppl
source=metrics-*
| eval
    rounded    = round(cpu_usage, 2),
    absolute   = abs(delta),
    ceiling    = ceil(memory_gb),
    floored    = floor(memory_gb),
    log_value  = log(requests_per_sec),
    sqrt_val   = sqrt(variance),
    powered    = pow(base, 2)
```

#### Conditional functions

```ppl
source=logs-*
| eval severity = case(
    level = 'ERROR',    'HIGH',
    level = 'WARN',     'MEDIUM',
    level = 'INFO',     'LOW',
    true,               'UNKNOWN'
  )
| eval is_error = isnull(error_code) = false
```

### 6.4 Ví dụ thực tế

#### Phân tích log lỗi theo giờ

```ppl
source=application-logs-*
| where level = 'ERROR' and timestamp >= '2025-06-01 00:00:00'
| eval hour = hour(timestamp)
| stats count() as error_count by hour, service
| sort + hour
```

#### Top 10 IP có nhiều request 4xx nhất

```ppl
source=access-logs-*
| where status_code >= 400 and status_code < 500
| stats count() as request_count by client_ip
| sort - request_count
| head 10
| fields client_ip, request_count
```

#### Tính tỷ lệ lỗi (error rate) theo service

```ppl
source=api-logs-*
| stats count() as total,
         count() as errors where status_code >= 500
  by service
| eval error_rate = round((errors / total) * 100, 2)
| where total > 100
| sort - error_rate
| fields service, total, errors, error_rate
```

#### Phát hiện bất thường — latency đột biến

```ppl
source=metrics-*
| stats avg(response_time_ms) as avg_latency,
         stddev(response_time_ms) as stddev_latency
  by service
| eval threshold = avg_latency + (3 * stddev_latency)
| eval is_anomaly = avg_latency > 2000 or stddev_latency > 500
| where is_anomaly = true
| sort - avg_latency
```

#### Parse Apache access log

```ppl
source=raw-access-logs-*
| grok message "%{COMBINEDAPACHELOG}"
| where response != "200"
| stats count() as count by response, verb
| sort - count
| fields verb, response, count
```

#### Tìm session user dài nhất

```ppl
source=user-events-*
| where event_type IN ('login', 'logout')
| stats min(timestamp) as session_start,
         max(timestamp) as session_end
  by session_id, user_id
| eval duration_min = timestampdiff(MINUTE, session_start, session_end)
| where duration_min > 0
| sort - duration_min
| head 20
| fields user_id, session_id, session_start, session_end, duration_min
```

---

## 7. Khi nào dùng ngôn ngữ nào?

```
Tôi cần...
│
├── Tìm kiếm / lọc dữ liệu bằng giao diện Dashboards
│   └── Dùng DQL (thanh search bar)
│
├── Xây dựng ứng dụng / tích hợp API
│   └── Dùng Query DSL (REST API với JSON body)
│
├── Phân tích log / metrics / traces
│   ├── Muốn cú pháp đơn giản, pipe-based → PPL
│   └── Cần aggregation phức tạp → Query DSL
│
├── Tạo visualization tùy chỉnh
│   ├── Chart đơn giản → Vega-Lite
│   └── Chart phức tạp, tương tác cao → Vega
│
└── Chạy ad-hoc query nhanh
    ├── Quen SQL → dùng SQL trong Query Workbench
    └── Quen Unix pipe / Splunk → dùng PPL
```

**Bảng tổng hợp:**

| Tình huống | Ngôn ngữ khuyến nghị |
|---|---|
| Filter realtime trên Dashboard | DQL |
| Search & relevance scoring | Query DSL (`match`, `multi_match`) |
| Exact match, aggregation | Query DSL (`term`, `terms`, `aggs`) |
| Log analysis, pipe workflow | PPL |
| Observability, DevOps troubleshoot | PPL |
| Custom chart, data visualization | Vega / Vega-Lite |
| API integration, programmatic search | Query DSL |
| Vector / semantic search | Query DSL (`knn`, `neural`) |

---

## 8. Tài nguyên tham khảo

**Tài liệu chính thức:**
- Query DSL: https://docs.opensearch.org/latest/query-dsl/
- DQL: https://docs.opensearch.org/latest/dashboards/dql/
- PPL: https://docs.opensearch.org/latest/sql-and-ppl/ppl/index/
- Vega: https://docs.opensearch.org/latest/dashboards/visualize/vega/

**Vega ecosystem:**
- Vega docs: https://vega.github.io/vega/docs/
- Vega-Lite docs: https://vega.github.io/vega-lite/docs/
- Vega Editor online: https://vega.github.io/editor/

**OpenSearch community:**
- Forum: https://forum.opensearch.org/
- GitHub: https://github.com/opensearch-project
- Playground: https://playground.opensearch.org/

---

*Tài liệu này được soạn thảo dựa trên OpenSearch phiên bản 3.x. Cập nhật lần cuối: tháng 6/2026.*
