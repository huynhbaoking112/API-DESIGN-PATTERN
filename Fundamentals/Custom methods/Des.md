Trong Part 3, mục **Custom methods** nói về những API method **không thuộc nhóm method chuẩn** như `Create`, `Get`, `List`, `Update`, `Delete`, `Replace`. Sách định nghĩa phần này là cách “đi vượt ra ngoài standard methods” để hỗ trợ các hành động đặc biệt trong resource-oriented API. 

## 1. Vấn đề: không phải hành động nào cũng hợp với standard methods

Standard methods phù hợp với CRUD:

```http
POST   /emails           // create
GET    /emails/123       // get
PATCH  /emails/123       // update
DELETE /emails/123       // delete
```

Nhưng có nhiều hành động không phải CRUD đơn thuần, ví dụ:

```text
SendEmail
ArchiveEmail
UnsendEmail
LaunchRocket
TranslateText
ExportEmails
ValidateEmailAddress
```

Ta có thể cố “nhét” chúng vào `Update`, nhưng như vậy API sẽ khó hiểu.

Ví dụ xấu:

```ts
UpdateEmail({
  id: "emails/123",
  state: "sent"
});
```

Nhìn thì giống chỉ cập nhật field `state`, nhưng thực tế nó có thể phải:

```text
1. Ghi email vào database
2. Kết nối SMTP server
3. Gửi email thật
4. Đổi trạng thái email thành sent
```

Vậy `UpdateEmail` không còn là “update dữ liệu” nữa, mà đã gây ra **side effect**. Đây là điều standard methods nên tránh, còn custom methods thì được phép làm. Sách nhấn mạnh custom methods là nơi phù hợp cho các hành động có side effect như gửi email, kích hoạt background job, hoặc cập nhật nhiều resource. 

## 2. Custom method là gì?

Custom method là một method riêng cho hành động cụ thể, ví dụ:

```http
POST /emails/123:send
POST /rockets/123:launch
POST /documents/123:archive
```

Cú pháp quan trọng là dấu hai chấm `:`:

```text
/resource/id:action
```

Ví dụ:

```http
POST /rockets/1234567:launch
```

Không nên viết:

```http
POST /rockets/1234567/launch
```

Vì `/launch` có thể bị hiểu nhầm là một sub-resource tên `launch`. Dấu `:` giúp nói rõ: “resource kết thúc ở đây, phần sau là action tùy chỉnh.” Sách cũng khuyến nghị custom methods hầu như luôn dùng HTTP `POST`; `GET` chỉ nên dùng nếu method thật sự an toàn và idempotent. 

## 3. Ví dụ dễ hiểu: gửi email

Thay vì làm thế này:

```ts
UpdateEmail({
  id: "users/1/emails/2",
  state: "sent"
});
```

Ta nên thiết kế:

```http
POST /users/1/emails/2:send
```

API definition:

```ts
abstract class EmailApi {
  @post("/{id=users/*/emails/*}:send")
  SendEmail(req: SendEmailRequest): Email;
}

interface SendEmailRequest {
  id: string;
}
```

Ý nghĩa rõ hơn nhiều: người dùng không “sửa field state”, mà đang yêu cầu hệ thống **gửi email**.

## 4. Khi nào custom method target resource, khi nào target collection?

Nếu hành động áp dụng cho **một resource cụ thể**, target resource đó:

```http
POST /users/1/emails/2:send
POST /documents/123:archive
POST /rockets/abc:launch
```

Nếu hành động áp dụng cho **nhiều resource trong cùng collection**, target collection:

```http
POST /users/1/emails:export
```

Ví dụ `ExportEmails` không phải export một email, mà export toàn bộ hoặc một tập email của user, nên nó nằm trên collection `emails`.

Nếu hành động áp dụng cho nhiều resource ở nhiều parent khác nhau, có thể dùng wildcard `-`:

```http
POST /users/-/emails:archive
```

Ý nghĩa: archive nhiều email, có thể thuộc nhiều user khác nhau.

## 5. Stateless custom methods

Có những custom method không thao tác với resource đã lưu, chỉ tính toán rồi trả kết quả. Ví dụ dịch văn bản:

```http
POST /text:translate
```

```ts
abstract class TranslationApi {
  @post("/text:translate")
  TranslateText(req: TranslateTextRequest): TranslateTextResponse;
}

interface TranslateTextRequest {
  sourceLanguageCode: string;
  targetLanguageCode: string;
  text: string;
}

interface TranslateTextResponse {
  text: string;
}
```

Đây là **stateless custom method**: input vào là text, output là bản dịch, không nhất thiết lưu resource nào.

Nhưng sách cảnh báo: đừng quá vội dùng stateless method. Hôm nay dịch text có thể đơn giản, nhưng sau này bạn có thể cần billing, project, model dịch riêng, glossary riêng, hoặc ML model tùy chỉnh. Khi đó nên gắn method vào một resource như `Project` hoặc `TranslationModel`: 

```http
POST /projects/123/text:translate
```

hoặc:

```http
POST /translationModels/abc/text:translate
```

## 6. Quy tắc đặt tên

Custom method vẫn nên theo dạng:

```text
Verb + Noun
```

Ví dụ tốt:

```text
SendEmail
ArchiveDocument
LaunchRocket
ExportEmails
ValidateEmailAddress
```

Không nên dùng custom method như một cách “nhét tham số vào tên”:

```text
CreateRocketForMars
ExportEmailsWithAttachments
SendEmailToUser
```

Những tên có `for`, `with`, `to` thường là dấu hiệu thiết kế chưa tốt. Thay vào đó, thông tin như “Mars”, “attachments”, “recipient” nên nằm trong request body hoặc trong resource model.

## 7. Trade-off: đừng lạm dụng custom methods

Custom methods rất tiện, nhưng dễ bị dùng sai. Nếu API có quá nhiều custom methods cho những việc CRUD bình thường, có thể resource layout đang sai.

Ví dụ không nên:

```http
POST /books/123:updateTitle
POST /books/123:deleteBook
POST /books:createBook
```

Vì những việc này standard methods đã làm tốt hơn:

```http
PATCH  /books/123
DELETE /books/123
POST   /books
```

Custom method nên dùng khi hành động có bản chất riêng, ví dụ:

```http
POST /emails/123:send
POST /emails/123:unsend
POST /emails/123:archive
POST /users/1/emails:export
POST /emailAddress:validate
```

## Tóm lại

Custom method là “cửa thoát” khi standard methods không diễn đạt đúng hành động. Dùng nó cho các hành động như **gửi**, **launch**, **archive**, **export**, **validate**, **translate**, hoặc những state transition có side effect. Format nên là:

```http
POST /resources/{id}:action
```

hoặc với collection:

```http
POST /parents/{id}/resources:action
```

Điểm quan trọng nhất: **standard methods dành cho CRUD thuần túy; custom methods dành cho hành động nghiệp vụ rõ ràng, đặc biệt là khi có side effects.**
