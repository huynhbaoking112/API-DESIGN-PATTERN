## Part 5 — Batch operations

Chương này nói về cách thao tác **nhiều resource trong một API call**, thay vì client phải gọi `Get`, `Create`, `Update`, `Delete` nhiều lần. Mục tiêu không chỉ là giảm số request, mà quan trọng hơn là có **tính atomic**: hoặc toàn bộ batch thành công, hoặc toàn bộ batch thất bại. Sách định nghĩa batch operations như phiên bản “nhiều resource” của standard methods, nhưng được thiết kế dưới dạng custom methods. 

---

## 18.1 Motivation

Trước giờ các pattern chủ yếu xử lý **một resource** hoặc thậm chí **một phần của resource**. Nhưng thực tế có nhiều case cần xử lý một nhóm resource cùng lúc.

Ví dụ bạn muốn update 2 message:

```http
PATCH /chatRooms/1/messages/a
PATCH /chatRooms/1/messages/b
```

Nếu request đầu thành công, request sau fail thì hệ thống rơi vào trạng thái nửa vời. Trong database, ta giải quyết bằng transaction: hoặc commit toàn bộ, hoặc rollback toàn bộ. Nhưng web API thường không expose transaction tổng quát kiểu database vì quá phức tạp. Batch operations là giải pháp “vừa đủ”: cho một số thao tác phổ biến chạy theo kiểu transaction-like. 

Nói đơn giản:

```text
Không batch:
- gọi 100 request riêng
- có thể 70 cái thành công, 30 cái fail
- client phải tự xử lý trạng thái lưng chừng

Có batch:
- gọi 1 request
- hoặc 100 cái cùng thành công
- hoặc không cái nào được áp dụng
```

---

## 18.2 Overview

Sách không đề xuất xây hệ thống transaction tổng quát cho API. Thay vào đó, tạo các custom methods tương ứng với standard methods:

```ts
BatchGet<Resources>()
BatchCreate<Resources>()
BatchUpdate<Resources>()
BatchDelete<Resources>()
```

Không có `BatchList`, vì `List` vốn đã làm việc trên collection rồi. Nếu muốn lấy nhiều resource theo điều kiện, dùng `List` + `filter` sẽ hợp lý hơn.

Ví dụ:

```ts
@post("/{parent=chatRooms/*}/messages:batchDelete")
BatchDeleteMessages(req: BatchDeleteMessagesRequest): void;
```

HTTP sẽ là:

```http
POST /chatRooms/1/messages:batchDelete
```

Điểm quan trọng: batch methods là custom methods nhưng vẫn mô phỏng behavior của standard methods. `BatchGet` giống nhiều lần `Get`, `BatchCreate` giống nhiều lần `Create`, nhưng tất cả được đóng gói vào một operation atomic. 

---

# 18.3 Implementation

## 18.3.1 Atomicity

Đây là ý quan trọng nhất của chương. Batch operation phải **atomic hoàn toàn**.

Ví dụ:

```http
GET /chatRooms/1/messages:batchGet?ids=a&ids=b&ids=c
```

Nếu `a` và `b` tồn tại nhưng `c` không tồn tại, API không nên trả về:

```json
{
  "resources": [messageA, messageB, null]
}
```

Theo sách, request phải fail toàn bộ. Lý do là nếu hỗ trợ partial success thì response sẽ phải chứa status riêng cho từng item, error riêng cho từng item, logic retry riêng cho từng item. API surface sẽ phức tạp hơn rất nhiều. Vì vậy batch methods trong pattern này chỉ có 2 trạng thái:

```text
Tất cả thành công
hoặc
Tất cả thất bại
```

Kể cả `BatchGet` cũng vậy, dù nghe có vẻ hơi nghiêm ngặt. 

---

## 18.3.2 Operation on the collection

Batch method nên target vào **collection**, không nên target vào parent resource rồi nhét tên resource vào action.

Nên dùng:

```http
POST /chatRooms/1/messages:batchUpdate
```

Không nên dùng:

```http
POST /chatRooms/1:batchUpdateMessages
```

Lý do: operation đang thao tác trên collection `messages`, nên URL nên thể hiện rõ collection đó. Custom action chỉ cần là `:batchUpdate`, `:batchDelete`, `:batchCreate`, `:batchGet`. Sách cũng đưa bảng so sánh giữa target parent resource và target collection, và khuyến nghị target collection khi batch thao tác trên nhiều resource cùng loại. 

---

## 18.3.3 Ordering of results

Response của batch method phải giữ **đúng thứ tự** với request.

Ví dụ create 2 chat room:

```json
{
  "requests": [
    { "resource": { "title": "Chat 1" } },
    { "resource": { "title": "Chat 2" } }
  ]
}
```

Response phải trả:

```json
{
  "resources": [
    { "id": "chatRooms/abc", "title": "Chat 1" },
    { "id": "chatRooms/xyz", "title": "Chat 2" }
  ]
}
```

Tức là `resources[0]` ứng với `requests[0]`, `resources[1]` ứng với `requests[1]`.

Điều này đặc biệt quan trọng với `BatchCreate`, vì trước khi create, client chưa biết ID server sẽ sinh ra. Nếu response đảo thứ tự, client phải tự deep-compare từng resource để đoán cái nào ứng với cái nào. 

---

## 18.3.4 Common fields

Khi thiết kế batch request, có 2 cách:

Cách 1: nhận list các request chuẩn.

```ts
interface BatchGetMessagesRequest {
  requests: GetMessageRequest[];
}
```

Cách 2: “hoist” field chung lên batch request.

```ts
interface BatchGetMessagesRequest {
  ids: string[];
}
```

Với action đơn giản như `BatchGet` hoặc `BatchDelete`, chỉ cần list ID nên dùng `ids: string[]` cho gọn. Với action phức tạp như `BatchCreate` hoặc `BatchUpdate`, nên dùng list standard request vì mỗi item có thể có parent, resource, field mask khác nhau. Sách gọi kỹ thuật kéo field chung lên trên là **hoisting common fields**. 

Ví dụ `BatchUpdate` có thể vừa dùng list request, vừa hoist field mask:

```ts
interface BatchUpdateMessagesRequest {
  parent: string;
  requests: UpdateMessageRequest[];
  fieldMask: FieldMask;
}
```

Nếu `fieldMask` ở batch level khác với `fieldMask` trong từng request con, API nên reject request. Không nên đoán ý định của user khi đang thao tác hàng loạt. 

---

## 18.3.5 Operating across parents

Có khi cần batch trên nhiều parent khác nhau. Ví dụ tạo message vào nhiều chat room:

```http
POST /chatRooms/-/messages:batchCreate
```

Dấu `-` nghĩa là wildcard: “nhiều parent khác nhau”.

Body:

```json
{
  "requests": [
    {
      "parent": "chatRooms/1",
      "resource": { "text": "Hello room 1" }
    },
    {
      "parent": "chatRooms/2",
      "resource": { "text": "Hello room 2" }
    }
  ]
}
```

Không nên thiết kế kiểu song song 2 list:

```ts
interface BatchCreateMessageRequest {
  parents: string[];
  resources: Message[];
}
```

Vì cách này phụ thuộc vào index giữa `parents[]` và `resources[]`, rất dễ lỗi. Dùng list request chuẩn sẽ rõ hơn. Nếu URL nói parent là `chatRooms/1` nhưng request con lại nói parent là `chatRooms/2`, API nên reject vì intent bị mâu thuẫn. 

Một lưu ý quan trọng: đừng tự động support cross-parent nếu không chắc. Hôm nay data có thể nằm trong một database, nhưng sau này parent có thể trở thành distribution key. Lúc đó query across parents có thể rất đắt hoặc không khả thi. 

---

## 18.3.6 Batch Get

`BatchGet` dùng để lấy nhiều resource theo ID. Vì nó tương tự standard `Get` và idempotent, sách cho phép dùng HTTP `GET`.

Ví dụ top-level resource:

```ts
@get("/chatRooms:batchGet")
BatchGetChatRooms(req: BatchGetChatRoomsRequest): BatchGetChatRoomsResponse;

interface BatchGetChatRoomsRequest {
  ids: string[];
}

interface BatchGetChatRoomsResponse {
  resources: ChatRoom[];
}
```

Với child resource:

```ts
@get("/{parent=chatRooms/*}/messages:batchGet")
BatchGetMessages(req: BatchGetMessagesRequest): BatchGetMessagesResponse;

interface BatchGetMessagesRequest {
  parent: string;
  ids: string[];
}
```

Response phải trả resource theo đúng thứ tự ID input. Nếu một ID không tồn tại, không có quyền truy cập, hoặc conflict với parent đã chỉ định, toàn bộ request fail. Nếu muốn behavior mềm hơn, sách khuyên dùng `List` + `filter` thay vì batch get. 

Batch get cũng có thể hỗ trợ partial retrieval bằng cách hoist một `fieldMask` chung:

```ts
interface BatchGetMessagesRequest {
  parent: string;
  ids: string[];
  fieldMask: FieldMask;
}
```

Một điểm dễ nhầm: **batch get không nên pagination**. Thay vào đó, API nên document giới hạn tối đa số resource được get trong một batch, ví dụ 100 hoặc 1000 item tùy resource size. 

---

## 18.3.7 Batch Delete

`BatchDelete` dùng để xóa nhiều resource theo ID. Dù standard `Delete` dùng HTTP `DELETE`, batch delete nên dùng `POST` vì đây là custom method và request cần body/list ID.

```ts
@post("/chatRooms:batchDelete")
BatchDeleteChatRooms(req: BatchDeleteChatRoomsRequest): void;

@post("/{parent=chatRooms/*}/messages:batchDelete")
BatchDeleteMessages(req: BatchDeleteMessagesRequest): void;

interface BatchDeleteMessagesRequest {
  parent: string;
  ids: string[];
}
```

Nó cũng atomic: nếu 1 resource trong 100 cái không xóa được, toàn bộ batch fail. Kể cả resource đã bị xóa trước đó cũng phải fail, vì theo tư duy imperative của sách, delete thành công nghĩa là “operation này đã xóa resource”, không chỉ là “sau operation resource không còn tồn tại”. 

---

## 18.3.8 Batch Create

`BatchCreate` phức tạp hơn `BatchGet` và `BatchDelete`, vì create không chỉ cần ID mà cần toàn bộ create request.

```ts
@post("/{parent=chatRooms/*}/messages:batchCreate")
BatchCreateMessages(req: BatchCreateMessagesRequest): BatchCreateMessagesResponse;

interface CreateMessageRequest {
  parent: string;
  resource: Message;
}

interface BatchCreateMessagesRequest {
  parent: string;
  requests: CreateMessageRequest[];
}

interface BatchCreateMessagesResponse {
  resources: Message[];
}
```

Vì sao không dùng `resources: Message[]` thôi? Vì parent thường không nằm trong resource body, mà nằm trong create request. Nếu muốn create message ở nhiều parent khác nhau mà không bắt client tự tạo ID, ta cần giữ nguyên `CreateMessageRequest` để mỗi item mang theo parent của nó.

Response của `BatchCreate` càng phải giữ đúng order, vì server mới sinh ID sau khi create. Nếu order bị đảo, client rất khó biết ID nào thuộc resource nào. 

---

## 18.3.9 Batch Update

`BatchUpdate` dùng để update nhiều resource trong một operation atomic.

```ts
@post("/{parent=chatRooms/*}/messages:batchUpdate")
BatchUpdateMessages(req: BatchUpdateMessagesRequest): BatchUpdateMessagesResponse;

interface UpdateMessageRequest {
  resource: Message;
  fieldMask: FieldMask;
}

interface BatchUpdateMessagesRequest {
  parent: string;
  requests: UpdateMessageRequest[];
  fieldMask: FieldMask;
}
```

Dù standard `Update` dùng HTTP `PATCH`, batch update vẫn dùng `POST` vì nó là custom method. Với update, `fieldMask` rất quan trọng. Có 2 cách:

```text
1. fieldMask chung ở batch level
2. fieldMask riêng trong từng UpdateMessageRequest
```

Nếu cả hai cùng tồn tại thì không được conflict. Ví dụ batch-level `fieldMask = ["title"]`, nhưng một request con lại `fieldMask = ["description"]`, API nên reject. 

---

## 18.3.10 Final API definition

Một API đầy đủ sẽ có dạng:

```ts
abstract class ChatRoomApi {
  @post("/chatRooms:batchCreate")
  BatchCreateChatRooms(req: BatchCreateChatRoomsRequest): BatchCreateChatRoomsResponse;

  @post("/{parent=chatRooms/*}/messages:batchCreate")
  BatchCreateMessages(req: BatchCreateMessagesRequest): BatchCreateMessagesResponse;

  @get("/chatRooms:batchGet")
  BatchGetChatRooms(req: BatchGetChatRoomsRequest): BatchGetChatRoomsResponse;

  @get("/{parent=chatRooms/*}/messages:batchGet")
  BatchGetMessages(req: BatchGetMessagesRequest): BatchGetMessagesResponse;

  @post("/chatRooms:batchUpdate")
  BatchUpdateChatRooms(req: BatchUpdateChatRoomsRequest): BatchUpdateChatRoomsResponse;

  @post("/{parent=chatRooms/*}/messages:batchUpdate")
  BatchUpdateMessages(req: BatchUpdateMessagesRequest): BatchUpdateMessagesResponse;

  @post("/chatRooms:batchDelete")
  BatchDeleteChatRooms(req: BatchDeleteChatRoomsRequest): void;

  @post("/{parent=chatRooms/*}/messages:batchDelete")
  BatchDeleteMessages(req: BatchDeleteMessagesRequest): void;
}
```

Nhìn chung:

```text
BatchGet/Delete:
- input chính là ids[]
- đơn giản, gọn

BatchCreate/Update:
- input là requests[]
- vì mỗi item có thể cần parent, resource, fieldMask riêng
```

---

## 18.4 Trade-offs

Trade-off lớn nhất là sách chọn **atomicity hơn convenience**. Điều này đôi khi hơi khó chịu: chỉ một item fail thì cả batch fail. Nhưng đổi lại API đơn giản hơn, predictable hơn, không cần response kiểu “item này success, item kia fail, item nọ warning”. 

Trade-off thứ hai là format input không hoàn toàn đồng nhất. `BatchGet` và `BatchDelete` dùng `ids[]`, còn `BatchCreate` và `BatchUpdate` dùng `requests[]`. Nếu cố ép tất cả dùng `requests[]` thì API nhất quán hơn, nhưng `BatchGet` sẽ dài dòng không cần thiết. Sách chọn sự đơn giản ở từng method hơn là đồng nhất tuyệt đối. 

---

## 18.5 Exercises

Các bài tập trong phần này xoay quanh những câu hỏi thiết kế chính:

1. Vì sao response phải giữ đúng order với request?
2. Nếu `parent` ở batch request khác với parent trong request con thì nên xử lý thế nào?
3. Vì sao batch request nên atomic?
4. Vì sao `BatchDelete` dùng `POST` thay vì `DELETE`?

Câu trả lời ngắn:

```text
1. Để client map result với input dễ dàng.
2. Reject request vì intent mâu thuẫn.
3. Để tránh partial success và trạng thái nửa vời.
4. Vì đây là custom method có body/list ID, không phải standard DELETE.
```

---

## Tóm tắt thực dụng

Nên thiết kế:

```http
GET  /chatRooms:batchGet?ids=1&ids=2
POST /chatRooms:batchCreate
POST /chatRooms:batchUpdate
POST /chatRooms:batchDelete

GET  /chatRooms/1/messages:batchGet?ids=a&ids=b
POST /chatRooms/1/messages:batchCreate
POST /chatRooms/1/messages:batchUpdate
POST /chatRooms/1/messages:batchDelete
```

Rule nhớ nhanh:

```text
Batch operation = standard method cho nhiều resource.

Tên:
BatchGetMessages
BatchCreateMessages
BatchUpdateMessages
BatchDeleteMessages

URL:
collection:batchMethod

Quan trọng:
- atomic toàn bộ
- response giữ đúng order input
- BatchGet/Delete dùng ids[]
- BatchCreate/Update dùng requests[]
- parent conflict thì reject
- cross-parent dùng "-"
- không pagination trong batch get; dùng max limit
```

Phần tiếp theo trong Part 5 là **Criteria-based deletion**.
