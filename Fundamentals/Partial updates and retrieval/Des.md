Phần **Partial updates and retrievals** nói về cách client có thể làm việc với **một phần của resource**, thay vì lúc nào cũng phải lấy hoặc ghi toàn bộ resource. Chương này mở rộng hai standard methods đã học ở phần trước: **Get** và **Update**, bằng công cụ chính gọi là **field mask**. Sách nói field mask dùng để chỉ rõ “tôi chỉ quan tâm đến những field này” khi retrieve hoặc update resource. 

## 1. Vấn đề: trước giờ ta xem resource như một khối nguyên

Giả sử `Bookmark` có dạng:

```ts
interface Bookmark {
  id: string;
  url: string;
  title: string;
  description: string;
  folderId: string;
  tags: string[];
  metadata: {
    imageUrl: string;
    wordCount: number;
    lastFetchedAt: string;
  };
}
```

Nếu dùng `GetBookmark` bình thường:

```http
GET /bookmarks/bmk_123
```

server trả về toàn bộ:

```json
{
  "id": "bookmarks/bmk_123",
  "url": "https://example.com",
  "title": "Example",
  "description": "Long description...",
  "folderId": "folders/fld_456",
  "tags": ["api", "design"],
  "metadata": {
    "imageUrl": "https://example.com/cover.png",
    "wordCount": 1200,
    "lastFetchedAt": "2026-05-24T10:00:00Z"
  }
}
```

Nhưng có lúc client chỉ cần `title` và `url`. Lấy toàn bộ resource sẽ tốn bandwidth, memory, và xử lý thừa. Đây là lý do cần **partial retrieval**.

Ngược lại, khi update, client có thể chỉ muốn đổi `title`. Nếu bắt client gửi lại toàn bộ bookmark, rất dễ ghi đè nhầm field khác. Đây là lý do cần **partial update**.

## 2. Partial retrieval: lấy một phần resource

**Partial retrieval** nghĩa là khi `GET`, client nói rõ muốn lấy field nào.

Ví dụ:

```http
GET /bookmarks/bmk_123?fieldMask=title&fieldMask=url
```

Response chỉ cần:

```json
{
  "id": "bookmarks/bmk_123",
  "title": "Example",
  "url": "https://example.com"
}
```

Trong sách, use case này đặc biệt hữu ích khi resource rất lớn, client có tài nguyên hạn chế, hoặc đường truyền đắt/chậm; khi list nhiều resource, mỗi field thừa nhỏ cũng bị nhân lên rất nhiều lần. 

Lưu ý: thường vẫn trả `id`, vì client cần biết resource nào đang được trả về.

## 3. Partial update: sửa một phần resource

**Partial update** nghĩa là khi `PATCH`, client chỉ sửa đúng field nó muốn sửa.

Ví dụ bookmark hiện tại:

```json
{
  "id": "bookmarks/bmk_123",
  "url": "https://example.com",
  "title": "Old title",
  "description": "Old description",
  "folderId": "folders/fld_456"
}
```

Client chỉ muốn đổi title:

```http
PATCH /bookmarks/bmk_123?fieldMask=title
Content-Type: application/json

{
  "title": "New title"
}
```

Kết quả:

```json
{
  "id": "bookmarks/bmk_123",
  "url": "https://example.com",
  "title": "New title",
  "description": "Old description",
  "folderId": "folders/fld_456"
}
```

Chỉ `title` đổi. Các field khác giữ nguyên.

Điểm quan trọng: `PATCH` khác `PUT`. `PATCH` là sửa một phần; `PUT`/`Replace` là thay toàn bộ resource. Nếu dùng `PUT` mà chỉ gửi `title`, server có thể hiểu rằng các field còn lại nên bị xóa/reset.

## 4. Field mask là gì?

**Field mask** đơn giản là danh sách tên field mà client muốn tác động đến.

Trong TypeScript có thể hình dung như sách viết:

```ts
type FieldMask = string[];
```

Với `Get`:

```ts
interface GetBookmarkRequest {
  id: string;
  fieldMask: FieldMask;
}
```

Với `Update`:

```ts
interface UpdateBookmarkRequest {
  resource: Bookmark;
  fieldMask: FieldMask;
}
```

Sách cũng đưa ra API definition tương tự: thêm `fieldMask` vào request của standard get và standard update. 

Ví dụ field mask:

```json
["title", "url"]
```

Nghĩa là: “chỉ lấy hoặc chỉ update `title` và `url`.”

## 5. Vì sao field mask nằm ở query parameter?

Với `GET`, HTTP không nên có request body. Với `PATCH`, body nên là chính resource đang update, không nên nhét thêm metadata kiểu `{ resource: ..., fieldMask: ... }` vào body vì sẽ phá cấu trúc chuẩn của `PATCH`.

Vì vậy sách khuyến nghị truyền field mask qua **query string**:

```http
GET /bookmarks/bmk_123?fieldMask=title&fieldMask=url
```

hoặc:

```http
PATCH /bookmarks/bmk_123?fieldMask=title
{
  "title": "New title"
}
```

Sách cũng bàn rằng repeated query parameter như `?fieldMask=title&fieldMask=description` là cách dễ tương thích hơn so với nhét tất cả vào một chuỗi phân tách bằng dấu phẩy. 

## 6. Nested fields: field nằm bên trong object

Nếu resource có nested object:

```json
{
  "metadata": {
    "imageUrl": "https://example.com/cover.png",
    "wordCount": 1200
  }
}
```

Muốn lấy chỉ `metadata.imageUrl`:

```http
GET /bookmarks/bmk_123?fieldMask=metadata.imageUrl
```

Response:

```json
{
  "id": "bookmarks/bmk_123",
  "metadata": {
    "imageUrl": "https://example.com/cover.png"
  }
}
```

Dấu `.` dùng để đi vào field con.

Ví dụ update:

```http
PATCH /bookmarks/bmk_123?fieldMask=metadata.imageUrl
{
  "metadata": {
    "imageUrl": "https://cdn.example.com/new.png"
  }
}
```

Chỉ `metadata.imageUrl` đổi, còn `metadata.wordCount` giữ nguyên.

## 7. Dấu `*`: lấy tất cả field

Nếu muốn nói “tất cả field”, dùng field mask:

```text
*
```

Ví dụ:

```http
GET /bookmarks/bmk_123?fieldMask=*
```

Sách nói `*` là sentinel value để chỉ “everything”; nếu `*` xuất hiện trong field mask thì tất cả field nên được trả về. 

Điểm này hữu ích khi API có một số field mặc định không trả về vì quá nặng, nhưng client thật sự muốn lấy hết.

## 8. Default behavior: không gửi fieldMask thì sao?

Với **GET**, nếu không gửi `fieldMask`, mặc định thường là trả về toàn bộ field của resource. Ngoại lệ là những field quá lớn hoặc tính toán quá đắt; nếu API loại field đó khỏi response mặc định, tài liệu phải nói rõ. 

```http
GET /bookmarks/bmk_123
```

Nghĩa là: “trả về resource như bình thường.”

Với **PATCH**, nếu không gửi `fieldMask`, API có thể **infer field mask** từ những field có mặt trong JSON body.

Ví dụ:

```http
PATCH /bookmarks/bmk_123
{
  "title": "New title"
}
```

API suy ra:

```json
["title"]
```

Nghĩa là chỉ update `title`.

Đây gọi là **implicit field mask**. Sách xem đây là cách rất tiện, vì client chỉ cần gửi field muốn đổi.

## 9. Missing khác `null`

Chỗ này rất quan trọng.

```http
PATCH /bookmarks/bmk_123
{
  "title": "New title"
}
```

Ở đây `description` **missing**, nghĩa là không đụng đến `description`.

Nhưng:

```http
PATCH /bookmarks/bmk_123
{
  "description": null
}
```

Ở đây `description` có mặt và có giá trị `null`, nghĩa là client muốn set `description = null`.

Tóm lại:

```text
field bị thiếu     → không update field đó
field có null      → update field đó thành null
field có giá trị   → update field đó thành giá trị mới
```

Đây là lý do implicit field mask phải phân biệt “missing” và “null”.

## 10. Dynamic map: xóa một key như thế nào?

Giả sử bookmark có field dynamic:

```json
{
  "settings": {
    "pinned": true,
    "color": "blue"
  }
}
```

Muốn set `settings.color = null`:

```http
PATCH /bookmarks/bmk_123?fieldMask=settings.color
{
  "settings": {
    "color": null
  }
}
```

Nhưng muốn **xóa hẳn key `color`**, không phải set null, sách gợi ý dùng explicit field mask nhưng body không chứa field đó:

```http
PATCH /bookmarks/bmk_123?fieldMask=settings.color
Content-Type: application/json

{}
```

Ý nghĩa: “hãy update `settings.color`, nhưng giá trị input không tồn tại, vậy hãy remove key này khỏi map.” Sách dùng ví dụ tương tự với `settings.test` và nhấn mạnh đây là remove field entirely, không phải set field đó thành `null`. 

## 11. Repeated fields / arrays: không update bằng index

Giả sử:

```json
{
  "tags": ["api", "design", "rest"]
}
```

Có nên cho phép như này không?

```http
PATCH /bookmarks/bmk_123?fieldMask=tags[1]
{
  "tags": ["architecture"]
}
```

Không nên.

Lý do: index trong array không phải identifier ổn định. Hôm nay `"design"` ở index 1, ngày mai có người thêm tag ở đầu array thì nó thành index 2. Sách khuyến nghị field mask **không nên cho phép address item trong array bằng vị trí/index**. 

Cách đúng hơn: xem array như một field atomic. Muốn đổi `tags`, thay cả list:

```http
PATCH /bookmarks/bmk_123?fieldMask=tags
{
  "tags": ["api", "architecture", "rest"]
}
```

Nếu bạn cần update từng phần tử riêng lẻ, có thể đó không nên là array nữa; nên cân nhắc dùng map hoặc tách thành sub-resource.

## 12. Invalid field thì xử lý sao?

Ví dụ client gửi:

```http
PATCH /bookmarks/bmk_123?fieldMask=oldField
{
  "oldField": "value"
}
```

Nhưng API version hiện tại không còn `oldField`.

Theo sách, đừng vội trả lỗi. Vì API có thể thay đổi qua version: field từng tồn tại rồi bị bỏ, hoặc client/server lệch version. Sách khuyến nghị xử lý field không tìm thấy như `undefined`, thay vì throw error ngay, để API bền hơn trước thay đổi tương thích. 

## 13. Field mask không phải GraphQL

Một điểm sách cảnh báo: field mask rất hữu ích, nhưng mục tiêu của nó hẹp. Nó dùng để:

```text
- giảm dữ liệu trả về
- update chính xác một vài field
```

Nó không phải công cụ query phức tạp để join nhiều resource liên quan. Nếu API cần lấy dữ liệu quan hệ phức tạp kiểu “bookmark + folder + owner + related bookmarks”, field mask không phải giải pháp mạnh; lúc đó GraphQL hoặc cơ chế query khác có thể phù hợp hơn. 

## 14. Áp dụng vào bookmark API

Thiết kế hợp lý:

```ts
interface Bookmark {
  id: string;
  url: string;
  title: string;
  description?: string;
  folderId?: string;
  tags: string[];
  metadata: {
    imageUrl?: string;
    wordCount?: number;
    lastFetchedAt?: string;
  };
}
```

Partial retrieval:

```http
GET /bookmarks/bmk_123?fieldMask=title&fieldMask=url
```

Partial update explicit:

```http
PATCH /bookmarks/bmk_123?fieldMask=title
{
  "title": "New title"
}
```

Partial update implicit:

```http
PATCH /bookmarks/bmk_123
{
  "title": "New title"
}
```

Nested field:

```http
PATCH /bookmarks/bmk_123?fieldMask=metadata.imageUrl
{
  "metadata": {
    "imageUrl": "https://cdn.example.com/cover.png"
  }
}
```

Replace entire tags list:

```http
PATCH /bookmarks/bmk_123?fieldMask=tags
{
  "tags": ["api", "design-patterns"]
}
```

Không nên:

```http
PATCH /bookmarks/bmk_123?fieldMask=tags[0]
```

## Tóm lại

**Partial updates and retrievals** xoay quanh một câu: client nên có cách nói rõ “tôi chỉ quan tâm đến những field này”.

Với `GET`, field mask giúp lấy ít dữ liệu hơn.

Với `PATCH`, field mask giúp sửa đúng field cần sửa, tránh ghi đè nhầm dữ liệu.

Quy tắc nhớ nhanh:

```text
GET + fieldMask    → chỉ lấy những field này
PATCH + fieldMask  → chỉ update những field này
PATCH không mask   → suy ra mask từ field có mặt trong body
*                  → tất cả field
nested.field       → field con
arrays             → không update bằng index
missing            → không đụng tới / hoặc remove nếu explicit mask trên dynamic map
null               → set field thành null
```
