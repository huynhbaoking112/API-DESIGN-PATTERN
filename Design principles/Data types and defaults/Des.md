Phần **Data types and defaults** nói về cách chọn **kiểu dữ liệu** cho field trong API và cách xử lý khi field bị **thiếu**, `null`, rỗng, hoặc có giá trị mặc định. Ý chính: trong API, data type không chỉ là chuyện code nội bộ; vì API được gọi từ nhiều ngôn ngữ khác nhau, dữ liệu phải đi qua serialization như JSON, nên nếu không định nghĩa rõ thì client rất dễ hiểu sai. Sách minh họa bằng hình 5.1: dữ liệu từ server được serialize thành bytes, gửi qua network, rồi client deserialize lại thành dữ liệu trong ngôn ngữ của họ. 

## 1. Vì sao data type quan trọng?

Trong code nội bộ, bạn có thể dùng type của ngôn ngữ như `string`, `number`, `boolean`. Nhưng API không chỉ phục vụ một ngôn ngữ. Client có thể viết bằng JavaScript, Python, Java, Go… Mỗi ngôn ngữ xử lý số, chuỗi, `null`, object hơi khác nhau.

Ví dụ:

```ts
2 + 4 // 6
"2" + "4" // "24"
```

Nếu API không nói rõ field là number hay string, client hoặc server có thể hiểu sai ý định. Sách nhấn mạnh rằng khi thiết kế API, ta không nên chỉ nghĩ theo type của ngôn ngữ đang dùng, mà phải nghĩ theo type của format serialize, thường là JSON. 

## 2. Missing vs null

Đây là phần rất quan trọng.

Giả sử API có resource:

```ts
interface Fruit {
  name: string;
  color: string;
}
```

Các JSON sau không giống nhau hoàn toàn:

```json
{ "name": "Apple", "color": "red" }
{ "name": "Apple", "color": "" }
{ "name": "Apple", "color": null }
{ "name": "Apple" }
```

Trường hợp đầu rõ ràng: color là `"red"`.

Nhưng ba trường hợp sau cần API định nghĩa rõ:

```json
"color": ""
```

nghĩa là màu rỗng?

```json
"color": null
```

nghĩa là không có màu?

```json
// không có field color
```

nghĩa là user không gửi field này, hay muốn server chọn default?

Sách nói đây là một trong những nguồn gây nhầm lẫn lớn nhất. API không nên mặc định “serialization library sẽ xử lý đúng”, vì mỗi client/library có thể xử lý khác nhau. 

Một cách hiểu thực tế:

```text
missing  = người dùng không nói gì về field này
null     = người dùng nói rõ: field này không có giá trị
""       = người dùng nói rõ: giá trị là chuỗi rỗng
0        = người dùng nói rõ: giá trị là số 0
[]       = người dùng nói rõ: danh sách rỗng
{}       = người dùng nói rõ: object/map rỗng
```

## 3. Booleans

Boolean phù hợp cho câu hỏi yes/no hoặc flag:

```ts
interface ChatRoom {
  id: string;
  archived: boolean;
  allowChatbots: boolean;
}
```

Nhưng Boolean có hai vấn đề.

Thứ nhất, nó có thể quá hạn chế. Nếu hôm nay bạn có:

```ts
allowChatbots: boolean
```

ngày mai có thể cần thêm:

```ts
allowModerators: boolean
allowChildren: boolean
allowAnonymousUsers: boolean
```

Lúc này, nhiều Boolean flags có thể là dấu hiệu rằng bạn cần một cấu trúc giàu hơn, ví dụ list roles, policy, hoặc enum-like string.

Thứ hai, tên Boolean nên viết theo hướng **positive** để dễ đọc:

```ts
allowChatbots
```

dễ hiểu hơn:

```ts
disallowChatbots
```

Vì `disallowChatbots === false` là double negative, đọc mệt hơn. Tuy nhiên, có trường hợp phải dùng tên negative để đạt default mong muốn, vì một số serialization format coi Boolean không set là `false`. Ví dụ nếu mặc định muốn anonymous users được phép vào, có thể dùng:

```ts
disallowAnonymousUsers: boolean
```

Khi field không set, giá trị `false` nghĩa là “không cấm anonymous users”. Nhưng sách cũng cảnh báo cách này khóa default vào tên field, làm sau này khó đổi default. 

## 4. Numbers

Number nên dùng cho thứ có ý nghĩa toán học: count, weight, price, size, duration…

Ví dụ tốt:

```ts
viewCount: number
weightGrams: number
priceUsd: number
```

Nhưng không nên dùng number cho identifier chỉ vì nó “trông giống số”.

Ví dụ không nên:

```ts
userId: number
```

nếu bạn không bao giờ cộng, trừ, nhân, chia ID. ID thực chất là token/symbol, nên thường nên là string:

```ts
userId: string
```

Sách nhấn mạnh: number nên được dùng khi giá trị có ý nghĩa số học, không phải chỉ vì nó gồm các chữ số. 

Với number, cần chú ý **bounds**: min, max, có cho phép số âm không, dùng 32-bit hay 64-bit. Đây không chỉ là chuyện database, vì client language có thể không parse được số quá lớn. JavaScript là ví dụ điển hình: nó không có integer type riêng kiểu truyền thống, và số lớn có thể mất precision.

Ví dụ nguy hiểm:

```js
JSON.parse('{"value": 9999999999999999999999999}')
```

Có thể không giữ chính xác giá trị ban đầu.

Với decimal cũng có lỗi floating point:

```js
0.1 + 0.2 // 0.30000000000000004
```

Vì vậy, sách gợi ý với số rất lớn hoặc decimal quan trọng, khi serialize qua JSON có thể biểu diễn bằng string:

```json
{ "price": "0.10" }
{ "largeCount": "9999999999999999999999999" }
```

Sau đó client dùng decimal/big number library để parse. 

## 5. Strings

String là type rất phổ biến và linh hoạt. Nó dùng cho tên, mô tả, địa chỉ, nội dung text, Base64-encoded bytes, và cả identifiers. Sách nói string thường là lựa chọn tốt nhất cho unique identifiers, kể cả khi identifier nhìn giống số. 

Nhưng string cũng cần thiết kế cẩn thận.

Bạn nên đặt giới hạn độ dài:

```ts
displayName: string // max 100 chars
description: string // max 5000 chars
```

Khi input vượt quá giới hạn, sách khuyên nên **reject** thay vì tự cắt bớt. Vì truncation có thể gây bất ngờ.

Ví dụ user gửi:

```json
{ "name": "A very very very long name..." }
```

Không nên âm thầm biến nó thành:

```json
{ "name": "A very very..." }
```

Tốt hơn là trả lỗi rõ ràng:

```text
name must be at most 100 characters
```

Về encoding, sách khuyên API nên dùng **UTF-8** cho string. Với string dùng làm identifier, còn nên chuẩn hóa Unicode theo **Normalization Form C**, vì có những ký tự nhìn giống nhau nhưng bytes khác nhau, ví dụ một ký tự có dấu có thể được biểu diễn bằng một code point hoặc bằng ký tự gốc + dấu. Nếu không chuẩn hóa, hai identifier nhìn giống nhau có thể bị máy tính coi là khác nhau. 

## 6. Enumerations

Enum trong code nội bộ rất tiện:

```ts
enum Color {
  Brown = 1,
  Blue,
  Green
}
```

Nhưng trong API public, sách khuyên **nên tránh enum nếu có thể**.

Lý do: nếu serialize enum thành số, log/request rất khó đọc:

```json
{ "eyeColor": 2 }
```

không rõ `2` là màu gì. Trong khi:

```json
{ "eyeColor": "blue" }
```

rõ hơn nhiều.

Vấn đề khác: sau này server thêm enum value mới, client cũ có thể không hiểu. Ví dụ server thêm `"hazel"`, client library cũ không có enum value này và có thể lỗi.

Vì vậy, nên dùng string + validate ở server:

```ts
eyeColor: string // "brown", "blue", "green", "hazel"
```

Nếu có chuẩn sẵn, dùng chuẩn đó. Ví dụ file type nên dùng media type như:

```text
application/pdf
application/msword
```

thay vì enum `PDF`, `WORD`. 

## 7. Lists

List dùng cho collection đơn giản:

```ts
interface Book {
  id: string;
  title: string;
  categories: string[];
}
```

Nhưng sách nhấn mạnh list nên được xem như **atomic field**. Nghĩa là khi update, nên thay cả list, không nên update item thứ 2, thứ 3 bằng index.

Không nên thiết kế API kiểu:

```http
PATCH /books/123/categories/2
```

Vì index không ổn định. Nếu ai đó insert item ở đầu list, item index 2 có thể thành item khác.

Tốt hơn:

```json
{
  "categories": ["history", "science"]
}
```

Nếu cần thao tác độc lập từng item, có thể list không còn phù hợp; nên biến item đó thành sub-resource.

Ví dụ nếu `comments` có thể rất nhiều, cần create/delete từng comment:

```http
POST /books/{bookId}/comments
DELETE /books/{bookId}/comments/{commentId}
```

thay vì nhúng toàn bộ comment list vào `Book`.

List cũng cần bounds: tối đa bao nhiêu item, mỗi item dài bao nhiêu. Nếu list có thể tăng không giới hạn, nên dùng collection resource + pagination. 

Về default, list khó hơn string. `[]` thường là giá trị hợp lệ, nghĩa là “danh sách rỗng”. Vì vậy không nên dùng `[]` để biểu diễn “hãy dùng default”. Nếu cần default phức tạp, có thể phải xử lý lúc create hoặc đổi thiết kế sang resource riêng. 

## 8. Maps

Map là key-value structure:

```ts
interface GroceryItem {
  id: string;
  name: string;
  calories: number;
  ingredientAmounts: Map<string, string>;
}
```

Ví dụ JSON:

```json
{
  "name": "Pringles",
  "ingredientAmounts": {
    "fat": "9 g",
    "sodium": "150 mg",
    "carbohydrate": "15 g",
    "sugar": "1 g"
  }
}
```

Map phù hợp khi các key là dynamic, không biết trước lúc thiết kế API. Ví dụ mỗi sản phẩm có danh sách thành phần khác nhau, nên map hợp lý hơn tạo field cứng cho mọi thành phần có thể có. 

Nhưng map cũng dễ bị lạm dụng. Nếu bạn biết trước schema, nên dùng custom data type/interface thay vì map.

Ví dụ tốt:

```ts
interface ChatRoom {
  id: string;
  name: string;
  securityConfig: SecurityConfig;
}

interface SecurityConfig {
  password: string;
  requirePassword: boolean;
  allowAnonymousUsers: boolean;
  allowChatBots: boolean;
}
```

Không nên biến mọi thứ thành:

```ts
settings: Map<string, string>
```

nếu các field thật ra đã biết trước.

Với map, key nên là string, không nên là number hay object. Key cũng nên dùng UTF-8 và Normalization Form C nếu key đóng vai trò như identifier. Ngoài ra, map cần giới hạn số lượng key, độ dài key, độ dài value; nếu không, nó có thể phình ra và trở thành nơi chứa dữ liệu tùy tiện. 

## 9. Tóm tắt thực dụng

Khi thiết kế field trong API, hãy tự hỏi:

```text
Field này có thể missing không?
Missing khác gì null?
Empty string/list/map có ý nghĩa gì?
Default là gì?
Type này có ổn khi đi qua JSON không?
Client JavaScript/Python/Java có hiểu đúng không?
Giá trị này có cần bounds không?
Có cần đơn vị trong tên field không?
Có nguy cơ field này lớn vô hạn không?
```

Một bảng nhớ nhanh:

| Type      | Dùng khi                      | Cẩn thận                                      |
| --------- | ----------------------------- | --------------------------------------------- |
| `boolean` | flag yes/no                   | đặt tên positive; tránh quá nhiều flags       |
| `number`  | giá trị có ý nghĩa toán học   | bounds, precision, decimal, không dùng cho ID |
| `string`  | text, ID, encoded data        | length limit, UTF-8, normalization            |
| `enum`    | ít thay đổi, internal ổn      | API public nên ưu tiên string                 |
| `list`    | collection nhỏ, atomic        | không update bằng index, cần size limit       |
| `map`     | key dynamic, không biết trước | key nên là string, cần bounds, tránh lạm dụng |

Nói ngắn gọn: **Data types and defaults dạy ta rằng chọn kiểu dữ liệu trong API là chọn contract dài hạn với client. Type không rõ, default không rõ, hoặc phân biệt sai giữa missing/null/empty sẽ làm API khó đoán và dễ bug.**
