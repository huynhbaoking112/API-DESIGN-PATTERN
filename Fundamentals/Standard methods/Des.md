Phần **Standard methods** nói về một ý rất quan trọng: trong resource-oriented API, ta không nên nghĩ “mỗi hành động đặt một tên tùy ý”, mà nên dùng một bộ hành động chuẩn áp dụng lặp lại cho mọi resource. Sách liệt kê các standard methods chính gồm: **Get, List, Create, Update, Delete**, và thêm một method bán-chuẩn là **Replace**. Mục tiêu là làm API **dễ đoán, nhất quán, dễ học**. 

## 1. Standard methods là gì?

Giả sử API của bạn có 2 resource:

```text
Bookmark
Folder
```

Thay vì thiết kế lung tung như:

```text
FetchBookmark()
ShowBookmarkInfo()
MakeNewBookmark()
ModifyBookmark()
RemoveBookmark()
```

Ta dùng một bộ tên chuẩn:

```text
GetBookmark()
ListBookmarks()
CreateBookmark()
UpdateBookmark()
DeleteBookmark()
ReplaceBookmark()
```

Với `Folder` cũng y chang:

```text
GetFolder()
ListFolders()
CreateFolder()
UpdateFolder()
DeleteFolder()
ReplaceFolder()
```

Cái hay là: một khi người dùng hiểu `Get`, `List`, `Create`, `Update`, `Delete`, họ có thể đoán cách dùng với mọi resource khác. Đây chính là tính **predictable** mà sách nhấn mạnh. 

## 2. Bảng tổng quan

| Method    | Ý nghĩa                    | HTTP thường dùng         |
| --------- | -------------------------- | ------------------------ |
| `Get`     | Lấy một resource cụ thể    | `GET /bookmarks/{id}`    |
| `List`    | Lấy danh sách resource     | `GET /bookmarks`         |
| `Create`  | Tạo resource mới           | `POST /bookmarks`        |
| `Update`  | Cập nhật một phần resource | `PATCH /bookmarks/{id}`  |
| `Delete`  | Xóa resource               | `DELETE /bookmarks/{id}` |
| `Replace` | Thay thế toàn bộ resource  | `PUT /bookmarks/{id}`    |

Ví dụ với bookmark API:

```http
GET /bookmarks/bmk_123
```

là `GetBookmark`.

```http
GET /bookmarks
```

là `ListBookmarks`.

```http
POST /bookmarks
{
  "url": "https://example.com",
  "title": "Example"
}
```

là `CreateBookmark`.

```http
PATCH /bookmarks/bmk_123
{
  "title": "New title"
}
```

là `UpdateBookmark`.

```http
DELETE /bookmarks/bmk_123
```

là `DeleteBookmark`.

```http
PUT /bookmarks/bmk_123
{
  "url": "https://example.com",
  "title": "Exact final state",
  "folderId": "folders/fld_456"
}
```

là `ReplaceBookmark`.

## 3. Get: lấy đúng một resource

`Get` dùng khi client đã biết ID và muốn lấy resource đó.

```http
GET /bookmarks/bmk_123
```

Trả về:

```json
{
  "id": "bookmarks/bmk_123",
  "url": "https://example.com",
  "title": "Example",
  "folderId": "folders/fld_456"
}
```

Điểm quan trọng: `Get` chỉ nên **đọc dữ liệu**, không nên gây side effect. Ví dụ `GetBookmark` không nên âm thầm tăng `viewCount`, gửi notification, hoặc cập nhật gì đó quan trọng. Sách nhấn mạnh standard methods nên làm đúng việc của nó và tránh side effects ngoài mong đợi. 

## 4. List: lấy danh sách resource

`List` dùng để xem một collection.

```http
GET /bookmarks
```

Hoặc nếu bookmark nằm dưới folder:

```http
GET /folders/fld_456/bookmarks
```

`List` thường trả về danh sách:

```json
{
  "results": [
    {
      "id": "bookmarks/bmk_123",
      "url": "https://example.com",
      "title": "Example"
    },
    {
      "id": "bookmarks/bmk_789",
      "url": "https://openai.com",
      "title": "OpenAI"
    }
  ]
}
```

Sách cảnh báo một vài thứ ở `List`:

Không nên vội thêm **total count chính xác**, vì về sau dữ liệu lớn, distributed database, filter phức tạp thì đếm chính xác có thể rất tốn kém.

Không nên hỗ trợ **custom sorting** tùy tiện quá sớm, vì sort toàn bộ tập dữ liệu lớn cũng có thể rất nặng.

Nên hỗ trợ **filtering** nếu cần, vì client lấy hết dữ liệu rồi tự lọc sẽ lãng phí.

Pagination sẽ được giải thích kỹ ở chương sau.

## 5. Create: tạo resource mới

`Create` dùng trên collection, không dùng trên resource cụ thể.

```http
POST /bookmarks
{
  "url": "https://example.com",
  "title": "Example",
  "folderId": "folders/fld_456"
}
```

Server tạo ID và trả về resource đã tạo:

```json
{
  "id": "bookmarks/bmk_123",
  "url": "https://example.com",
  "title": "Example",
  "folderId": "folders/fld_456"
}
```

Điểm cần nhớ: `Create` nên tạo resource và trả về resource mới. Nếu tạo xong mà ngay sau đó `Get` hoặc `List` không thấy resource thì API sẽ gây khó hiểu. Vì vậy sách khuyến khích strong consistency cho standard create: tạo xong thì các method chuẩn khác nên nhìn thấy resource ngay. 

Với bookmark API, sau khi gọi:

```http
POST /bookmarks
```

thì ngay sau đó:

```http
GET /bookmarks/bmk_123
```

nên trả về bookmark vừa tạo.

## 6. Update: cập nhật một phần resource

`Update` dùng `PATCH`, nghĩa là chỉ sửa phần được gửi lên, không thay toàn bộ resource.

Ví dụ bookmark hiện tại:

```json
{
  "id": "bookmarks/bmk_123",
  "url": "https://example.com",
  "title": "Example",
  "folderId": "folders/fld_456"
}
```

Client chỉ muốn đổi title:

```http
PATCH /bookmarks/bmk_123
{
  "title": "Example Website"
}
```

Kết quả:

```json
{
  "id": "bookmarks/bmk_123",
  "url": "https://example.com",
  "title": "Example Website",
  "folderId": "folders/fld_456"
}
```

Nghĩa là `url` và `folderId` vẫn giữ nguyên.

Đây là lý do `Update` dùng `PATCH` chứ không dùng `PUT`. `PATCH` là “sửa một phần”, còn `PUT` trong pattern này dành cho `Replace`, tức “thay toàn bộ”.

Phần “sửa một phần chính xác thế nào” sẽ nối sang chương tiếp theo: **Partial updates and retrievals**, dùng field masks.

## 7. Delete: xóa resource

`Delete` dùng để xóa resource cụ thể:

```http
DELETE /bookmarks/bmk_123
```

Nếu thành công, response có thể rỗng:

```http
204 No Content
```

Điểm hơi ngược trực giác trong sách: sách cho rằng standard delete nên mang tính **imperative**, tức là “hãy thực hiện hành động xóa resource này”. Nếu resource đã bị xóa rồi, lần xóa thứ hai nên trả lỗi, ví dụ `404 Not Found`, vì lần thứ hai server không thật sự xóa được gì nữa. Do đó sách nói delete trong pattern này không nên được xem là idempotent theo nghĩa “gọi lặp lại vẫn có kết quả giống nhau”. 

Ví dụ:

```http
DELETE /bookmarks/bmk_123
→ 204 No Content
```

Gọi lại:

```http
DELETE /bookmarks/bmk_123
→ 404 Not Found
```

Lý do sách thích cách này: nó phân biệt rõ “tôi vừa xóa thành công” với “resource đó vốn đã không tồn tại”.

## 8. Replace: thay toàn bộ resource

`Replace` dùng `PUT`. Nó khác `Update`.

`Update`:

```http
PATCH /bookmarks/bmk_123
{
  "title": "New title"
}
```

chỉ đổi `title`.

`Replace`:

```http
PUT /bookmarks/bmk_123
{
  "title": "New title"
}
```

có nghĩa là: “hãy làm resource trên server giống đúng object này”. Những field không gửi lên có thể bị xóa hoặc reset.

Ví dụ trước đó bookmark có:

```json
{
  "id": "bookmarks/bmk_123",
  "url": "https://example.com",
  "title": "Old title",
  "folderId": "folders/fld_456"
}
```

Gọi:

```http
PUT /bookmarks/bmk_123
{
  "title": "Only title remains"
}
```

Thì kết quả có thể thành:

```json
{
  "id": "bookmarks/bmk_123",
  "title": "Only title remains",
  "url": null,
  "folderId": null
}
```

Vì vậy `Replace` nguy hiểm hơn `Update`, nhưng hữu ích khi client muốn đảm bảo trạng thái cuối cùng của resource **chính xác như mình gửi**.

## 9. Có phải resource nào cũng phải có đủ mọi standard method không?

Không bắt buộc. Nhưng sách nói nguyên tắc chung là: mỗi resource nên hỗ trợ standard methods, trừ khi có lý do tốt để không hỗ trợ, và lý do đó nên được document rõ. Nếu method về mặt khái niệm không phù hợp với resource type, có thể trả `405 Method Not Allowed`; nếu method hợp lý về mặt loại resource nhưng instance cụ thể không cho phép, ví dụ resource đang write-protected, thì nên trả `403 Forbidden`. 

Ví dụ `Bookmark` thường có đủ:

```text
CreateBookmark
GetBookmark
ListBookmarks
UpdateBookmark
DeleteBookmark
ReplaceBookmark
```

Nhưng nếu có resource kiểu `SystemStatus`, chỉ để đọc trạng thái hệ thống, có thể chỉ có:

```text
GetSystemStatus
```

Không nhất thiết phải có `CreateSystemStatus` hay `DeleteSystemStatus`.

## 10. Side effects: điểm rất quan trọng

Standard methods không nên làm chuyện phụ quá bất ngờ.

Ví dụ với email API:

```http
POST /emails
```

Nếu `CreateEmail` chỉ tạo bản nháp email thì ổn.

Nhưng nếu `CreateEmail` vừa tạo email vừa gửi email thật qua SMTP, đó là side effect lớn. Sách xem kiểu này không phù hợp với standard create; nên tách thành custom method như `SendEmail`, phần này sẽ học ở chương **Custom methods**. 

Áp vào bookmark API:

```http
POST /bookmarks
```

nên chỉ tạo bookmark.

Không nên âm thầm:

```text
- gửi email cho user
- crawl website ngay lập tức
- post lên mạng xã hội
- trigger workflow nặng
```

Nếu cần crawl metadata của URL, có thể cân nhắc custom method hoặc long-running operation về sau.

## 11. Tóm lại bằng bookmark API

Một thiết kế chuẩn sẽ trông như sau:

```ts
interface Bookmark {
  id: string;
  url: string;
  title: string;
  folderId?: string;
}

interface Folder {
  id: string;
  name: string;
}
```

API methods:

```http
POST   /bookmarks              CreateBookmark
GET    /bookmarks/{id}         GetBookmark
GET    /bookmarks              ListBookmarks
PATCH  /bookmarks/{id}         UpdateBookmark
PUT    /bookmarks/{id}         ReplaceBookmark
DELETE /bookmarks/{id}         DeleteBookmark

POST   /folders                CreateFolder
GET    /folders/{id}           GetFolder
GET    /folders                ListFolders
PATCH  /folders/{id}           UpdateFolder
PUT    /folders/{id}           ReplaceFolder
DELETE /folders/{id}           DeleteFolder
```

Ý chính của chương này: **đừng sáng tạo tên method khi không cần thiết**. Hãy dùng bộ chuẩn trước, vì nó giúp API dễ học, dễ đoán, nhất quán. Chỉ khi hành động không vừa với standard methods, ví dụ `archive`, `send`, `export`, `move`, `copy`, lúc đó mới nghĩ đến custom methods.
