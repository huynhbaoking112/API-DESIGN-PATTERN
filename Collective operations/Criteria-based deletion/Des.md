## Part 5 — Criteria-based deletion

Phần tiếp theo là **Criteria-based deletion**: xóa nhiều resource **theo điều kiện**, thay vì phải biết sẵn danh sách ID. Ví dụ: “xóa tất cả message đã archived”, “xóa tất cả log cũ hơn 90 ngày”, “xóa tất cả draft chưa dùng”. Sách gọi method này là **purge**. fileciteturn16file0

---

## 19.1 Motivation

Ở chương trước, `BatchDelete` cho phép xóa nhiều resource một lần, nhưng nó có điều kiện: client phải biết sẵn ID của các resource cần xóa.

Ví dụ muốn xóa tất cả chat room đã archived, nếu chỉ dùng API cũ thì phải làm kiểu:

```ts
const archivedRooms = ListChatRooms({
  filter: "archived: true"
});

BatchDeleteChatRooms({
  ids: archivedRooms.map(room => room.id)
});
```

Cách này có 2 vấn đề lớn.

Thứ nhất, phải gọi ít nhất 2 API call: `List` trước, rồi `BatchDelete` sau. Nếu kết quả nhiều, `List` còn phải phân trang nhiều lần nữa.

Thứ hai, quan trọng hơn, nó **không atomic**. Giữa lúc client list ra các resource archived và lúc gọi batch delete, dữ liệu có thể đã thay đổi. Ví dụ một room vừa được unarchive, nhưng ID của nó vẫn nằm trong list cũ, và sau đó vẫn bị xóa nhầm. Vì vậy sách đề xuất một method riêng để server tự tìm và xóa trong cùng một operation: `purge`. fileciteturn16file0

---

## 19.2 Overview

`purge` là custom method dùng để xóa tất cả resource match một `filter`.

Ví dụ:

```http
POST /chatRooms/1/messages:purge
```

Body:

```json
{
  "filter": "archived = true",
  "force": true
}
```

Có thể hiểu `purge` là kết hợp của:

```text
List + BatchDelete
```

Nhưng thay vì client tự list rồi tự delete, server nhận filter và tự xử lý toàn bộ. Vấn đề là method này rất nguy hiểm: nếu filter match toàn bộ resource, user có thể xóa sạch dữ liệu. Vì vậy sách thêm 2 lớp an toàn chính: mặc định chỉ **validate/preview**, và chỉ xóa thật khi client gửi `force: true`. Response preview nên có `purgeCount` và `purgeSample` để user biết mình sắp xóa bao nhiêu item và một vài item mẫu là gì. fileciteturn16file1

---

# 19.3 Implementation

## 19.3.1 Filtering results

`purge` nhận một `filter` giống như `List` method. Nghĩa là thay vì truyền ID:

```json
{
  "ids": ["messages/1", "messages/2"]
}
```

ta truyền điều kiện:

```json
{
  "filter": "archived = true"
}
```

Server sẽ áp filter đó lên collection rồi xóa các resource match. Filter nên dùng cùng semantics với standard `List`, vì nếu `ListMessages(filter)` trả ra tập nào thì `PurgeMessages(filter)` cũng nên nhắm tới đúng tập đó. Điều này giúp API predictable hơn.

Với filter rỗng, cần cực kỳ cẩn thận. Trong nhiều API, filter rỗng thường có nghĩa là “match tất cả”. Nếu áp dụng vào `purge`, nó có thể thành “xóa tất cả”. Vì vậy trong thực tế, nên reject filter rỗng hoặc bắt user xác nhận rất rõ qua `force`, preview count/sample, và quyền phù hợp.

---

## 19.3.2 Validation only by default

Khác với nhiều method khác, `purge` **không nên xóa thật theo mặc định**. Nếu client gọi:

```http
POST /chatRooms/1/messages:purge
```

với body:

```json
{
  "filter": "archived = true"
}
```

thì API nên chỉ trả preview, chưa xóa gì cả.

Response có thể là:

```json
{
  "purgeCount": 248,
  "purgeSample": [
    "chatRooms/1/messages/a",
    "chatRooms/1/messages/b",
    "chatRooms/1/messages/c"
  ]
}
```

Muốn xóa thật thì phải gọi lại với:

```json
{
  "filter": "archived = true",
  "force": true
}
```

Ý tưởng này giống `validateOnly`, nhưng đảo default: với `purge`, mặc định là validate-only; muốn execute thật thì phải nói rõ `force: true`. Đây là guard rail rất quan trọng vì hành động này có thể phá dữ liệu hàng loạt. fileciteturn16file1

---

## 19.3.3 Result count

`purgeCount` cho biết có bao nhiêu resource sẽ bị ảnh hưởng.

Trong validation request, `purgeCount` có thể là estimate nếu tính chính xác quá đắt. Nhưng sách cảnh báo: **đừng underestimate quá thấp**, vì nó gây cảm giác an toàn giả. Ví dụ API preview nói sẽ xóa khoảng 100 resource, nhưng thực tế xóa 1,000 resource, user sẽ rất khó chịu vì nếu biết con số gần đúng hơn thì họ đã sửa filter trước khi chạy thật. fileciteturn16file3

Với request chạy thật (`force: true`), nếu có thể thì `purgeCount` nên phản ánh số resource thật sự đã bị xóa.

---

## 19.3.4 Result sample set

Chỉ có count chưa đủ. Ví dụ `purgeCount = 50`, nhưng 50 đó là đúng hay sai? Có thể bạn muốn xóa 50 archived resource, nhưng filter viết nhầm thành match 50 unarchived resource.

Vì vậy response nên có `purgeSample`: danh sách một số ID match filter. User hoặc UI có thể dùng `BatchGet` để lấy chi tiết những item mẫu này rồi hiển thị cho user kiểm tra trước khi xóa thật. Sách khuyên với dataset lớn nên trả ít nhất khoảng 100 item mẫu; với dataset nhỏ, nếu rẻ thì có thể trả toàn bộ match. fileciteturn17file0

Ví dụ UI có thể hiển thị:

```text
Bạn sắp xóa khoảng 248 messages.
Một số message sẽ bị xóa:
- chatRooms/1/messages/a
- chatRooms/1/messages/b
- chatRooms/1/messages/c
```

Nếu nhìn sample thấy đúng, user mới bấm confirm và client gửi lại request với `force: true`.

---

## 19.3.5 Consistency

Đây là phần dễ nhầm nhất. Validation preview **không đảm bảo** rằng request chạy thật sau đó sẽ xóa đúng y chang tập resource đã preview.

Ví dụ lúc preview:

```text
filter = "archived = true"
purgeCount = 10
```

Nhưng 5 phút sau, trước khi user confirm, có thêm 1,000 resource được archived. Khi gửi `force: true`, API có thể xóa 1,010 resource.

Sách nói gần như không có cách đảm bảo preview và execution luôn giống nhau trong hệ thống concurrent. Nếu cố bắt purge fail khi dữ liệu thay đổi giữa validation và execution, method sẽ gần như vô dụng trên dataset lớn, volatile, nhiều người dùng. Vì vậy `purge` nên tuân theo consistency guideline giống `List`: filter được áp ở thời điểm request thật chạy, không phải cố “đóng băng” tập preview trước đó. fileciteturn19file2

Nếu bạn thật sự cần “xóa đúng tập đã preview”, thì dùng flow cũ hợp lý hơn:

```text
List theo snapshot / thời điểm cụ thể
=> lấy danh sách ID
=> BatchDelete danh sách ID đó
```

---

## 19.3.6 Final API definition

API cuối cùng trong sách có dạng:

```ts
abstract class ChatRoomApi {
  @post("/{parent=chatRooms/*}/messages:purge")
  PurgeMessages(req: PurgeMessagesRequest): PurgeMessagesResponse;
}

interface PurgeMessagesRequest {
  parent: string;
  filter: string;
  force?: boolean;
}

interface PurgeMessagesResponse {
  purgeCount: number;
  purgeSample: string[];
}
```

HTTP tương ứng:

```http
POST /chatRooms/1/messages:purge
```

Preview:

```json
{
  "filter": "archived = true"
}
```

Xóa thật:

```json
{
  "filter": "archived = true",
  "force": true
}
```

Response:

```json
{
  "purgeCount": 248,
  "purgeSample": [
    "chatRooms/1/messages/a",
    "chatRooms/1/messages/b"
  ]
}
```

Sách tóm tắt rằng `purge` nên dùng để xóa nhiều resource match filter, mặc định chỉ validation, response nên có count và sample, và consistency nên giống standard `List`. fileciteturn19file2

---

## 19.4 Trade-offs

Trade-off lớn nhất: method này **rất nguy hiểm**. Sách ví nó như đưa “bazooka” cho user API: chỉ một filter sai có thể xóa lượng dữ liệu rất lớn. Dù có `force`, `purgeCount`, `purgeSample`, method này vẫn mở cửa cho việc user tự phá dữ liệu của mình. Vì vậy chỉ nên support `purge` khi thật sự cần. fileciteturn19file2

Trong nhiều hệ thống, phương án an toàn hơn là:

```text
List + user review + BatchDelete
```

Dài hơn, nhưng user biết chính xác đang xóa ID nào. `purge` hợp với các tác vụ cleanup lớn, admin tool, retention policy, hoặc dữ liệu có thể phục hồi bằng soft deletion.

---

## 19.5 Exercises

Các bài tập trong phần này xoay quanh 5 câu hỏi chính:

```text
1. Vì sao purge chỉ nên có khi thật sự cần?
2. Vì sao default phải là validation-only?
3. Filter rỗng thì chuyện gì xảy ra?
4. Nếu resource hỗ trợ soft deletion thì purge nên soft-delete hay expunge?
5. Vì sao response cần cả count và sample?
```

Câu trả lời thực dụng:

```text
1. Vì purge có thể xóa rất nhiều dữ liệu chỉ bằng một filter sai.
2. Vì preview giúp user phát hiện lỗi trước khi xóa thật.
3. Có thể match tất cả, nên cực kỳ nguy hiểm.
4. Tùy semantics API, nhưng nếu delete bình thường là soft-delete thì purge thường nên soft-delete theo behavior delete; expunge nên là action riêng.
5. Count cho biết quy mô ảnh hưởng, sample giúp kiểm tra filter có match đúng loại resource không.
```

---

## Tóm tắt thực dụng

Nên thiết kế:

```http
POST /resources:purge
POST /parents/{parent}/resources:purge
```

Request:

```json
{
  "filter": "archived = true",
  "force": false
}
```

Response:

```json
{
  "purgeCount": 100,
  "purgeSample": ["resources/1", "resources/2"]
}
```

Rule nhớ nhanh:

```text
Criteria-based deletion = xóa theo filter.

Tên method:
PurgeMessages
PurgeChatRooms
PurgeLogs

URL:
collection:purge

Bắt buộc nghĩ tới:
- filter phải rõ ràng
- mặc định không xóa thật
- xóa thật cần force: true
- trả purgeCount
- trả purgeSample
- preview không guarantee execution sẽ y chang nếu dữ liệu thay đổi
- chỉ nên support khi thật sự cần
```

Phần tiếp theo là **Anonymous writes**.
