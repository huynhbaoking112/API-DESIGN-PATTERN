Phần **Part 3 – Resource identification** nói về cách **định danh tài nguyên trong API**: mỗi resource phải có một “địa chỉ” ổn định để client có thể `get`, `update`, `delete`, hoặc tham chiếu đến nó. Trong sách, chapter này bắt đầu Part 3 – nhóm các pattern nền tảng áp dụng cho hầu hết API. 

## 1. Identifier là gì?

**Identifier** là giá trị dùng để chỉ đúng **một resource cụ thể** trong một collection.

Ví dụ:

```http
GET /bookmarks/abc123
```

Ở đây:

```text
bookmarks/abc123
```

là identifier đầy đủ của resource `Bookmark`.

Nó gồm 2 phần:

```text
bookmarks / abc123
^ collection ^ unique id trong collection
```

Điểm quan trọng là identifier không chỉ là “một cái số ID trong database”. Trong API, nó là thứ client nhìn thấy, lưu lại, copy, share, dùng trong URL, và dùng làm reference đến resource khác.

## 2. Một identifier tốt cần có gì?

Sách đưa ra nhiều tiêu chí cho identifier tốt: dễ dùng, unique, permanent, dễ generate, khó đoán, dễ đọc/chia sẻ/kiểm tra, và có mật độ thông tin tốt. 

Giải thích ngắn gọn:

**Unique**: ID phải trỏ đến đúng một resource. Không thể có hai bookmark cùng ID trong cùng scope.

**Permanent**: đã cấp ID rồi thì không nên đổi. Ví dụ hôm nay bookmark A là `bookmarks/123`, ngày mai không nên đổi thành `bookmarks/999`, vì client có thể đã lưu ID cũ.

**Không tái sử dụng**: nếu `bookmarks/123` bị xóa, không nên dùng lại `123` cho bookmark khác. Nếu dùng lại, reference cũ có thể vô tình trỏ sang resource mới.

**Unpredictable**: không nên dùng ID tuần tự như `1`, `2`, `3` cho public API, vì người khác có thể đoán ID tiếp theo và dò tài nguyên.

**Dễ đọc/dễ share**: ID có thể phải copy vào ticket support, log, email, hoặc đọc qua điện thoại. Vì vậy nên tránh các ký tự dễ nhầm như `l`, `I`, `1`, `O`, `0`.

## 3. Identifier nên có dạng gì?

Sách khuyến nghị từ góc nhìn client, identifier nên là **string**, không phải number. Lý do là ID không dùng để tính toán; nó giống “token” hơn là số học. Vì vậy cộng hai ID lại không có ý nghĩa. Sách cũng khuyến nghị dùng ASCII và một encoding dễ đọc như **Crockford’s Base32**, vì nó tránh một số ký tự dễ nhầm và vẫn khá gọn. 

Ví dụ nên dùng:

```text
bookmarks/64S36D1N6RVKGE9GC5H66~
```

Thay vì:

```text
bookmarks/123456789
```

Hoặc thay vì một UUID raw dài và khó đọc:

```text
bookmarks/123e4567-e89b-12d3-a456-426655440000
```

## 4. Checksum để phân biệt “không tồn tại” và “ID sai”

Một ý rất hay trong chương này là thêm **checksum character** vào cuối ID.

Ví dụ:

```text
bookmarks/AHM6A83HENMP~
```

Ký tự cuối `~` có thể là checksum. Khi client gửi ID lên, server tính lại checksum. Nếu checksum sai, server biết đây là ID **không hợp lệ**, có thể do copy nhầm hoặc gõ nhầm. Nếu checksum đúng nhưng không tìm thấy resource, lúc đó mới là resource **không tồn tại**. Sách giải thích checksum giúp phân biệt “resource missing” với “identifier không thể hợp lệ”. 

Khác biệt này quan trọng:

```text
GET /bookmarks/AHM6A83HENMP~
→ 404 Not Found
```

Nghĩa là ID hợp lệ, nhưng bookmark không tồn tại.

```text
GET /bookmarks/AHM6A83HENMZ~
→ 400 Bad Request
```

Nghĩa là ID sai format hoặc checksum sai, có thể người dùng copy nhầm.

## 5. Có nên đưa resource type vào ID không?

Có. Sách khuyến nghị prefix ID bằng collection name, ví dụ:

```text
books/abcde-12345-ghjkm-67890
```

Thay vì chỉ:

```text
abcde-12345-ghjkm-67890
```

Lý do là chỉ nhìn `abcde-...` thì không biết đây là `Book`, `Author`, `Folder`, hay `Bookmark`. Khi có prefix, API và con người đều hiểu resource type ngay. Hơn nữa dạng này rất hợp với URL:

```http
GET /books/abcde-12345-ghjkm-67890
```

Sách cũng nói dùng dấu `/` với collection name là lựa chọn tự nhiên vì khớp với HTTP URL. 

## 6. Hierarchy trong identifier: chỉ dùng khi thật sự là ownership

Ví dụ:

```text
folders/abc/bookmarks/xyz
```

Dạng này nói rằng bookmark `xyz` là con của folder `abc`.

Nhưng sách cảnh báo: hierarchy chỉ nên dùng khi quan hệ là **ownership thật sự**, không chỉ là “đang nằm ở đâu đó”. Nếu child có thể move sang parent khác, identifier dạng hierarchy sẽ gây vấn đề vì ID phải đổi. Mà chương này đã nhấn mạnh ID nên permanent. 

Áp vào bookmark API của bạn:

Nếu bookmark **có thể chuyển folder**, đừng dùng ID kiểu:

```text
folders/{folderId}/bookmarks/{bookmarkId}
```

vì khi move bookmark sang folder khác, URL/ID sẽ đổi.

Nên dùng:

```text
bookmarks/{bookmarkId}
```

và trong resource có field:

```ts
interface Bookmark {
  id: string;        // bookmarks/abc
  folderId: string; // folders/xyz
  url: string;
}
```

Còn nếu bookmark **không bao giờ tồn tại ngoài folder**, xóa folder thì xóa luôn bookmark, quyền truy cập folder áp xuống bookmark, và bookmark không move folder, khi đó hierarchy mới hợp lý:

```text
folders/{folderId}/bookmarks/{bookmarkId}
```

## 7. Generate ID: server nên tạo, không nên để user tự chọn

Sách khuyến nghị API service nên generate ID. Nếu để user tự chọn ID, có vài vấn đề:

Người dùng có thể chọn ID dễ đoán như `my-bookmark`, `test`, `bookmark1`.

Người dùng có thể vô tình đưa thông tin nhạy cảm vào ID.

Có thể gây collision với ID đã từng tồn tại, kể cả resource đã bị xóa.

Do đó, với public API, tốt hơn là server tạo ID random đủ lớn, ví dụ:

```text
bookmarks/01HZX7K9Q4M8P2V6N3A1B~
```

Client chỉ gửi nội dung:

```json
{
  "url": "https://example.com",
  "title": "Example"
}
```

Server trả về resource có ID:

```json
{
  "id": "bookmarks/01HZX7K9Q4M8P2V6N3A1B~",
  "url": "https://example.com",
  "title": "Example"
}
```

## 8. Tomb-stoning: ID đã dùng thì nhớ là đã dùng

Vì ID không nên tái sử dụng, hệ thống cần nhớ những ID đã từng được dùng. Cách đơn giản là soft delete: resource bị xóa nhưng record vẫn còn trạng thái `deleted`. Cách khác là lưu riêng danh sách ID đã dùng. Sách gọi đây là **tomb-stoning**, tức là giữ “bia mộ” cho ID đã chết để không cấp lại ID đó. 

Ví dụ:

```text
bookmarks/abc từng tồn tại
bookmarks/abc bị xóa
```

Sau này không được tạo bookmark mới cũng có ID:

```text
bookmarks/abc
```

## 9. UUID có dùng được không?

Có, nhưng sách không xem UUID là câu trả lời hoàn hảo cho mọi trường hợp. UUID tốt vì phổ biến, có không gian ID lớn, xác suất collision thấp. Nhưng sách nêu vài nhược điểm: UUID khá lớn, string UUID dùng Base16 nên mật độ thông tin thấp hơn Base32, và UUID mặc định không có checksum để phân biệt ID sai với ID không tồn tại. 

Cách dung hòa tốt:

Dùng UUID nội bộ nếu database/support tooling thích UUID.

Nhưng expose ra ngoài bằng dạng Base32 có checksum.

Ví dụ internal:

```text
123e4567-e89b-12d3-a456-426655440000
```

External API:

```text
bookmarks/28F6J8R6X4Z0C9NQ7P1~
```

## Tóm lại

Chương này muốn nói rằng **ID không phải chi tiết nhỏ**. ID là một phần public contract của API. Một thiết kế tốt nên dùng string ID, khó đoán, không đổi, không tái sử dụng, dễ copy/share, có checksum, và có resource type rõ ràng như:

```text
bookmarks/{bookmarkId}
folders/{folderId}
```

Với bookmark API của bạn, hướng hợp lý là:

```text
folders/{folderId}
bookmarks/{bookmarkId}
```

và `Bookmark` có field `folderId`, trừ khi bạn chắc chắn bookmark là child thật sự của folder và không bao giờ move folder.
