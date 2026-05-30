## Part 5 — Anonymous writes

Phần tiếp theo là **Anonymous writes**: ghi dữ liệu vào API nhưng **không tạo resource có ID riêng**. Pattern này dùng cho dữ liệu kiểu log, analytics, time-series, event tracking, metric, audit event… tức là dữ liệu được ghi vào hệ thống để xử lý hoặc aggregate về sau, chứ không phải để user gọi `GET /dataPoints/{id}` từng bản ghi. 

---

## 20.1 Motivation

Từ đầu sách đến giờ, mỗi khi “ghi dữ liệu” vào API, ta thường nghĩ đến `Create<Resource>`:

```http
POST /chatRooms
POST /chatRooms/1/messages
```

Sau khi create, resource có ID, có thể get, update, delete:

```http
GET /chatRooms/1/messages/abc
PATCH /chatRooms/1/messages/abc
DELETE /chatRooms/1/messages/abc
```

Nhưng không phải dữ liệu nào cũng nên là resource. Ví dụ tracking số lần user mở chat room:

```json
{
  "name": "room_opened",
  "value": 1
}
```

Ta không thật sự cần ID cho từng event này. User cũng không cần gọi:

```http
GET /chatRooms/1/statEntries/entry-123
```

Thứ họ cần thường là aggregate:

```text
Hôm nay chat room này được mở bao nhiêu lần?
Trung bình mỗi ngày có bao nhiêu lượt mở?
Tỷ lệ user active theo tuần là bao nhiêu?
```

Sách ví việc biến từng datapoint thành resource giống như “bọc và dán nhãn từng hạt gạo”, thay vì mua cả bao gạo. Với analytics/time-series/data warehouse, user quan tâm tổng hợp dữ liệu hơn là từng datapoint riêng lẻ. Nhiều hệ thống analytics thậm chí không có unique identifier cho từng datapoint. 

---

## 20.2 Overview

Pattern này giới thiệu custom method tên là **write**.

Nó giống `Create` ở chỗ đều đưa dữ liệu mới vào API, nhưng khác ở chỗ dữ liệu được ghi vào là **anonymous entry**, không phải resource có identifier.

Ví dụ:

```http
POST /chatRooms/1/statEntries:write
```

Body:

```json
{
  "entry": {
    "name": "room_opened",
    "value": 1
  }
}
```

Sau khi write xong, entry này không có ID để get/update/delete. Nó chỉ đi vào pipeline hoặc storage để sau này query dạng aggregate. Sách gọi dữ liệu kiểu này là **entry**, không gọi là **resource**, để tránh hiểu nhầm rằng nó có full resource lifecycle. 

Có thể hiểu nhanh:

```text
Create = tạo resource có ID, sau này còn tương tác riêng được.
Write  = ghi entry không có ID, sau này chỉ đọc qua aggregate.
```

---

# 20.3 Implementation

## Return type: không trả resource

Với standard `Create`, response thường trả resource vừa tạo:

```ts
CreateMessage(req): Message
```

Nhưng với `write`, không có resource để trả. Entry vừa ghi không có ID, không được address riêng, không update/delete riêng. Vì vậy response đúng nhất là **không có body**, chỉ trả status thành công hoặc lỗi.

```ts
WriteChatRoomStatEntry(req): void
```

Không nên trả entry vừa ghi, cũng không nên trả fake ID. Nếu API trả ID thì user sẽ tưởng entry đó có thể `GET` lại được. 

---

## Field name: dùng `entry`, không dùng `resource`

Vì method này không xử lý resource, request body không nên dùng field `resource`.

Không nên:

```ts
interface WriteChatRoomStatEntryRequest {
  parent: string;
  resource: ChatRoomStatEntry;
}
```

Nên:

```ts
interface WriteChatRoomStatEntryRequest {
  parent: string;
  entry: ChatRoomStatEntry;
}
```

Lý do: `resource` mang nghĩa “thứ có ID, có lifecycle, có thể get/update/delete”. Còn ở đây ta chỉ ghi một entry vào stream/analytics pipeline. Sách khuyến nghị coi write request giống create request, nhưng đổi field từ `resource` sang `entry`. 

---

## URL: target collection, không target parent resource

Nếu ghi stat entry cho chat room, có 2 kiểu URL có thể nghĩ tới:

Không nên:

```http
POST /chatRooms/1:writeStatEntry
```

Nên:

```http
POST /chatRooms/1/statEntries:write
```

Dù `statEntries` không hẳn là collection resource đầy đủ, sách vẫn khuyên target vào collection-like path. Lý do là về mặt ý nghĩa, ta đang thêm entry vào một tập entry thuộc chat room. Sau này ta cũng có thể thêm các method khác trên collection này như batch write hoặc purge. 

---

## Batch write

Vì dữ liệu analytics/log thường được gửi nhiều record một lúc, API nên hỗ trợ batch write.

```http
POST /chatRooms/1/statEntries:batchWrite
```

Ví dụ body:

```json
{
  "requests": [
    {
      "parent": "chatRooms/1",
      "entry": {
        "name": "room_opened",
        "value": 1
      }
    },
    {
      "parent": "chatRooms/1",
      "entry": {
        "name": "message_sent",
        "value": 1
      }
    }
  ]
}
```

Response vẫn nên là `void`, không trả danh sách resource, vì mỗi entry vẫn không có ID riêng.

---

## 20.3.1 Consistency

Đây là điểm khác lớn nhất giữa `Create` và `Write`.

Với `Create`, nếu API trả success, user thường kỳ vọng có thể đọc lại ngay:

```http
POST /messages
=> 200 OK

GET /messages/{id}
=> thấy resource vừa tạo
```

Nhưng với `Write`, dữ liệu thường đi qua analytics pipeline. Nó có thể được nhận thành công, nhưng chưa xuất hiện ngay trong aggregate. Ví dụ vừa ghi event `room_opened`, nhưng query count ngay sau đó có thể chưa tăng. Sách nói điều này chấp nhận được, vì dữ liệu write thường được đọc qua aggregate, và rất khó biết aggregate đó đã bao gồm chính entry của mình hay chưa. 

Vì vậy `Write` có thể trả thành công ngay khi API nhận dữ liệu, trước khi dữ liệu thật sự visible trong analytics result. Trường hợp muốn nói rõ “đã nhận nhưng chưa xử lý xong”, có thể trả HTTP `202 Accepted` thay vì `200 OK`. 

Không nên dùng LRO cho từng write. Lý do là nếu mỗi event analytics đều trả một `Operation`, hệ thống sẽ phải track tiến độ từng datapoint, rất nặng và thường không có ích. Muốn biết event đã “vào aggregate chưa” cũng không đơn giản, vì aggregate được tính từ nhiều entry của nhiều user khác nhau. Sách khuyên nếu lo duplicate write thì dùng pattern **Request deduplication** ở chương 26, không dùng LRO. 

---

## 20.3.2 Final API definition

Ví dụ trong sách là ghi stat entry cho chat room:

```ts
abstract class ChatRoomApi {
  @post("/{parent=chatRooms/*}/statEntries:write")
  WriteChatRoomStatEntry(
    req: WriteChatRoomStatEntryRequest
  ): void;

  @post("/{parent=chatRooms/*}/statEntries:batchWrite")
  BatchWriteChatRoomStatEntry(
    req: BatchWriteChatRoomStatEntryRequest
  ): void;
}

interface ChatRoomStatEntry {
  name: string;
  value: number | string | boolean | null;
}

interface WriteChatRoomStatEntryRequest {
  parent: string;
  entry: ChatRoomStatEntry;
}

interface BatchWriteChatRoomStatEntryRequest {
  parent: string;
  requests: WriteChatRoomStatEntryRequest[];
}
```

Điểm cần nhớ trong API definition này:

```text
Method name:
WriteChatRoomStatEntry
BatchWriteChatRoomStatEntry

URL:
collection:write
collection:batchWrite

Request field:
entry, không phải resource

Response:
void, không trả resource
```

Sách nhấn mạnh cả write và batch write đều target collection, không target parent resource, và đều không trả resource trong response. 

---

## 20.4 Trade-offs

Pattern này khá hẹp use case. Nó gần như chỉ phù hợp với **analytical data**, không phải transactional data. Transactional data là kiểu `User`, `Order`, `Message`, `Payment` — cần ID, cần đọc lại, update, delete, audit rõ ràng. Analytical data là kiểu event/log/metric — ghi vào để tính toán, thống kê, aggregate. 

Bạn vẫn có thể dùng `CreateDataPoint` và coi từng datapoint là resource, nhưng thiết kế đó thường không hợp với analytics storage/data processing system. Dữ liệu tăng rất nhanh, số lượng cực lớn, và việc quản lý từng datapoint như resource riêng sẽ làm API nặng và khó dùng. Với trường hợp cần ingest analytics data bên cạnh resource-oriented API truyền thống, `write` thường là lựa chọn hợp lý hơn. 

Nhưng cần nhớ: dữ liệu ghi qua `write` là **one-way street**. Sau khi ghi, thường không có cách xóa từng entry riêng lẻ, vì nó không có ID. Nếu domain yêu cầu sửa/xóa từng record, hoặc cần user review từng record, thì nó không nên là anonymous write; nó nên là resource thật. 

---

## 20.5 Exercises

Các bài tập trong phần này xoay quanh 3 câu hỏi:

```text
1. Nếu sợ duplicate data khi write thì xử lý thế nào?
2. Vì sao write method không nên trả response body?
3. Nếu muốn báo “data đã nhận nhưng chưa xử lý”, nên làm gì?
```

Câu trả lời thực dụng:

```text
1. Dùng request deduplication / idempotency key.
2. Vì không có resource để trả; trả resource hoặc LRO sẽ làm user hiểu sai.
3. Dùng HTTP 202 Accepted là hợp lý nhất trong đa số API.
```

---

## Tóm tắt thực dụng

Dùng anonymous writes khi bạn có dữ liệu kiểu:

```text
analytics event
log entry
metric datapoint
time-series datapoint
tracking event
stat entry
```

Thiết kế nên là:

```http
POST /parents/{parent}/entries:write
POST /parents/{parent}/entries:batchWrite
```

Ví dụ:

```http
POST /chatRooms/1/statEntries:write
```

Body:

```json
{
  "entry": {
    "name": "room_opened",
    "value": 1
  }
}
```

Rule nhớ nhanh:

```text
Anonymous write = ghi entry không có ID.

Không dùng:
CreateDataPoint nếu datapoint không bao giờ được address riêng.

Dùng:
WriteStatEntry
WriteLogEntry
WriteMetricEntry

Quan trọng:
- dùng custom method :write
- target collection-like path
- request dùng entry
- response void
- không trả fake ID
- không trả LRO trừ trường hợp rất đặc biệt
- dữ liệu có thể eventually visible
- nếu cần báo chưa xử lý xong, dùng 202 Accepted
- nếu sợ duplicate, dùng request deduplication
```

Phần tiếp theo là **Pagination**.
