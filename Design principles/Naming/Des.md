Trong **Part 2 – Design principles**, mục **Naming** là chương 3. Ý chính: *đặt tên trong API không phải chuyện “trang trí”, mà là một phần quan trọng của thiết kế API*, vì tên của resource, method, field, enum… là thứ người dùng API sẽ nhìn thấy, gọi đến, ghi vào code và rất khó đổi về sau. 

## 1. Vì sao tên quan trọng?

Trong code nội bộ, nếu đặt tên hàm/biến chưa tốt, ta còn có thể refactor. Nhưng trong API public, tên đã xuất hiện trong client code của người dùng. Một khi đổi tên field hoặc method, code cũ của họ có thể hỏng. Vì vậy, tác giả so sánh việc đổi tên API giống như đổi số điện thoại hoặc địa chỉ: bạn không thể chắc chắn mọi người sẽ cập nhật theo. 

Ví dụ: nếu API từng có field:

```ts
userName
```

sau đó đổi thành:

```ts
username
```

thì rất nhiều client đang dùng `userName` có thể lỗi. Vì vậy, chọn tên đúng ngay từ đầu rất quan trọng.

## 2. Một tên “tốt” cần có 3 đặc điểm

### Expressive – Diễn đạt rõ nghĩa

Tên phải nói rõ nó đại diện cho cái gì. Ví dụ:

```ts
postal_address
```

rõ hơn:

```ts
address
```

nếu trong hệ thống có nhiều loại địa chỉ như địa chỉ email, địa chỉ IP, địa chỉ giao hàng.

Nhưng cũng phải chú ý ngữ cảnh. Từ `topic` có thể nghĩa là “chủ đề” trong machine learning, nhưng cũng có thể là “topic” trong hệ thống message queue như Kafka. Nếu API có cả hai khái niệm, nên dùng tên rõ hơn như:

```ts
model_topic
messaging_topic
```

### Simple – Đơn giản

Tên không nên dài quá mức. Tên phải đủ rõ, nhưng không thêm chữ không cần thiết.

Ví dụ trong sách:

```ts
UserSpecifiedPreferences
```

quá dài, vì `Specified` không thêm nhiều ý nghĩa.

```ts
Preferences
```

lại quá ngắn, vì không rõ là preferences của ai.

Tên tốt hơn là:

```ts
UserPreferences
```

Nó vừa rõ, vừa gọn.

### Predictable – Dễ đoán, nhất quán

Cùng một khái niệm thì phải dùng cùng một tên. Không nên lúc thì gọi là:

```ts
topic
```

lúc khác lại gọi:

```ts
messagingTopic
```

nếu cả hai đều chỉ cùng một thứ. Người dùng API thường không đọc toàn bộ tài liệu; họ học một phần rồi suy luận phần còn lại. Nếu API đặt tên không nhất quán, họ sẽ đoán sai và mất thời gian debug. 

## 3. Quy tắc về ngôn ngữ, ngữ pháp và cú pháp

Tác giả khuyên API nên dùng **American English** nếu muốn tối đa khả năng tương thích toàn cầu. Ví dụ dùng:

```ts
color
```

thay vì:

```ts
colour
```

Với method/action, nên dùng dạng mệnh lệnh, tức là động từ ra lệnh:

```ts
CreateBook
DeleteMessage
ArchiveLogEntry
```

Không nên đặt tên mơ hồ kiểu:

```ts
isValid()
```

vì người dùng có thể không rõ đây là hàm local hay remote call, và response trả về là boolean hay danh sách lỗi. Tên như sau rõ hơn:

```ts
GetValidationErrors()
```

## 4. Cẩn thận với giới từ trong tên

Các từ như `with`, `for`, `to`, `from` đôi khi là dấu hiệu cho thấy thiết kế API đang có vấn đề.

Ví dụ:

```ts
ListBooksWithAuthors()
ListBooksWithPublishers()
ListBooksWithAuthorsAndPublishers()
```

Nếu cứ thêm các biến thể như vậy, API sẽ phình ra rất nhanh. Thay vào đó, có thể dùng cơ chế như **field mask** hoặc **view** để người dùng chọn dữ liệu liên quan cần lấy.

Tác giả gọi prepositions trong tên là một dạng “API smell”: không phải lúc nào cũng sai, nhưng nên kiểm tra kỹ vì có thể đang che giấu một vấn đề thiết kế lớn hơn. 

## 5. Số nhiều và collection name

Resource thường đặt ở dạng số ít:

```ts
Book
Author
Publisher
```

Nhưng URL collection thường dùng số nhiều:

```http
/books/1234
/authors/5678
```

Không nên máy móc thêm `s` cho mọi trường hợp. Ví dụ:

```ts
Person
```

collection nên là:

```http
/people
```

không phải:

```http
/persons
```

Quan trọng nhất là chọn một quy ước đúng và dùng nhất quán.

## 6. Cú pháp: camelCase, snake_case, kebab-case

Tên field, resource, message, method nên theo convention của hệ sinh thái bạn dùng.

Ví dụ:

```ts
firstName     // camelCase
first_name    // snake_case
first-name    // kebab-case
```

Không có kiểu nào “đúng tuyệt đối”; vấn đề là phải nhất quán. Nếu trong Protocol Buffers, message thường dùng `UpperCamelCase` như:

```ts
UserSettings
```

còn field thường dùng `snake_case` như:

```ts
first_name
```

thì nên theo chuẩn đó.

Ngoài ra, tránh dùng reserved keywords làm tên field, ví dụ `from`, `to`, `string`, `class`. Với email API, thay vì:

```ts
from
to
```

có thể dùng:

```ts
sender
recipient
```

## 7. Context – Ngữ cảnh ảnh hưởng đến ý nghĩa tên

Một từ có thể đúng trong API này nhưng gây hiểu nhầm trong API khác.

Ví dụ từ:

```ts
book
```

Trong Library API, nó là resource “sách”. Nhưng trong Flight Reservation API, `book` có thể là hành động “đặt vé”.

Vì vậy, khi đặt tên, không chỉ nhìn riêng cái tên mà phải xem nó nằm trong API nào, domain nào, cạnh những khái niệm nào.

## 8. Data types và units – Nên đưa đơn vị vào tên khi cần

Một field như:

```ts
size
```

rất mơ hồ. Với audio, `size` có thể là bytes hoặc seconds. Với image, `size` có thể là bytes, pixels, hoặc megapixels.

Tên rõ hơn:

```ts
sizeBytes
durationSeconds
sizeMegapixels
```

Nếu dữ liệu phức tạp hơn, nên dùng type giàu ý nghĩa thay vì nhét mọi thứ vào string.

Không nên:

```ts
dimensionsPixels: "1024x768"
```

Tốt hơn:

```ts
dimensions: Dimensions

interface Dimensions {
  widthPixels: number;
  heightPixels: number;
}
```

Như vậy API vừa dễ hiểu, vừa ít lỗi parsing hơn.

## 9. Ví dụ hậu quả của tên xấu

Một ví dụ trong sách là `pageSize` trong pagination. Người dùng có thể hiểu `pageSize = 10` nghĩa là “trả đúng 10 item”. Nhưng thực tế API có thể chỉ cam kết “tối đa 10 item”. Tên rõ hơn sẽ là:

```ts
maxPageSize
```

Thiếu chữ `max` khiến người dùng hiểu sai, ví dụ nhận 8 item rồi tưởng là đã hết dữ liệu, trong khi thực ra vẫn còn trang tiếp theo. 

Ví dụ nghiêm trọng hơn là Mars Climate Orbiter: một nhóm dùng đơn vị pound-force seconds, nhóm khác dùng Newton seconds. Nếu tên field chỉ là:

```ts
impulse
```

thì rất dễ hiểu nhầm. Nếu đặt rõ:

```ts
impulsePoundForceSeconds
impulseNewtonSeconds
```

thì sự khác biệt đơn vị sẽ lộ ra ngay. 

## Tóm tắt dễ nhớ

Tên trong API nên:

1. **Rõ nghĩa**: người đọc hiểu nó đại diện cho gì.
2. **Đơn giản**: không dài hơn mức cần thiết.
3. **Nhất quán**: cùng khái niệm dùng cùng một tên.
4. **Theo convention**: language, grammar, casing phải ổn định.
5. **Có ngữ cảnh**: tên phải phù hợp domain.
6. **Có đơn vị khi cần**: đặc biệt với number field.
7. **Tránh mơ hồ**: vì đổi tên API sau khi public là rất khó.

Nói ngắn gọn: **đặt tên tốt giúp API dễ học, dễ đoán, ít bug và bền vững hơn khi phát triển lâu dài.**
