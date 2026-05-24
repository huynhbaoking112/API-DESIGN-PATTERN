Trong Part 4, chương **Singleton sub-resources** nói về một kiểu thiết kế nằm giữa **field/property bình thường** và **resource riêng biệt**. Ý tưởng chính: có một nhóm dữ liệu “thuộc về” resource cha, nhưng vì nó lớn, thay đổi độc lập, có quyền truy cập riêng, hoặc có pattern sử dụng khác, ta tách nó ra thành một **sub-resource chỉ có đúng 1 bản thể cho mỗi resource cha**. Sách mô tả pattern này là cách chuyển một nhóm property của resource thành một singleton child resource, ví dụ `Driver.location` được tách thành `drivers/{id}/location`. 

## 12.1 Motivation

Vấn đề xuất phát từ tình huống: về mặt mô hình dữ liệu, một phần dữ liệu **rõ ràng là thuộc về resource cha**, nhưng nếu nhét thẳng vào resource cha thì API lại trở nên tệ hơn. Ví dụ `Document` có thể có ACL, tức danh sách ai được quyền truy cập document. ACL đúng là “thuộc về document”, nhưng nó có thể lớn, nhạy cảm, hoặc có quyền đọc/sửa khác với chính document. Sách nêu ví dụ shared document cần lưu access control list để xác định ai có quyền truy cập và với vai trò nào. 

Lý do nên tách thành singleton sub-resource thường gồm: dữ liệu quá lớn, cấu trúc phức tạp, có yêu cầu bảo mật riêng, hoặc có access pattern khác với resource cha. Ví dụ trong ride-sharing API, `Driver` có `name`, `licensePlate` khá ổn định, nhưng `DriverLocation` thay đổi liên tục. Nếu cứ nhét `lat/long` vào `Driver`, mỗi lần cập nhật vị trí sẽ làm resource `Driver` thay đổi liên tục dù phần lớn dữ liệu của driver không đổi.

Nói ngắn gọn: **dữ liệu vẫn là một phần tự nhiên của resource cha, nhưng cần được cô lập để dễ quản lý hơn.**

## 12.1.1 Why should we use a singleton sub-resource?

Ta dùng singleton sub-resource khi không muốn biến dữ liệu đó thành một collection thông thường, vì mỗi resource cha chỉ nên có **một** phần dữ liệu đó.

Ví dụ:

```http
GET /drivers/1
```

trả về:

```json
{
  "id": "drivers/1",
  "name": "Alice",
  "licensePlate": "ABC-123"
}
```

Còn vị trí hiện tại nằm riêng:

```http
GET /drivers/1/location
```

trả về:

```json
{
  "id": "drivers/1/location",
  "lat": 40.741718,
  "long": -74.004159,
  "updateTime": "..."
}
```

`location` không phải là một list kiểu `/drivers/1/locations/{locationId}` vì mỗi driver chỉ có một current location. Nhưng nó cũng không còn là field trực tiếp của `Driver`, vì nó có vòng đời truy cập và cập nhật riêng. Đây chính là “singleton”: **một và chỉ một sub-resource cố định bên dưới parent**.

## 12.2 Overview

Sách mô tả singleton sub-resource là một dạng **hybrid**: vừa giống resource, vừa giống property. Nó giống resource ở chỗ có URL riêng, có thể `GET`, `PATCH` riêng. Nhưng nó giống property ở chỗ không được tạo/xóa độc lập; nó tồn tại vì parent tồn tại. Sách nói rõ pattern này tạo ra một thành phần có một số đặc tính của full resource và một số đặc tính của simple resource property. 

Có thể hình dung:

```text
Không dùng singleton:

Driver
- id
- name
- licensePlate
- locationLat
- locationLong

Dùng singleton:

Driver
- id
- name
- licensePlate

DriverLocation
- id = drivers/1/location
- lat
- long
- updateTime
```

Điểm hay là client nào chỉ cần thông tin driver thì gọi `GET /drivers/1`, còn client nào cần vị trí thì gọi `GET /drivers/1/location`.

## 12.3 Implementation

Phần implementation là phần quan trọng nhất. Sách so sánh singleton sub-resource với các standard methods: `Get` và `Update` hành xử như resource riêng; `Create` và `Delete` hành xử như property; `List` thì không áp dụng vì chỉ có đúng một sub-resource. 

### 12.3.1 Standard methods

**GET và UPDATE**

`GET` và `UPDATE` hoạt động như resource riêng. Nghĩa là singleton sub-resource có URL riêng và identifier riêng. Ví dụ:

```http
GET /drivers/1/location
```

và:

```http
PATCH /drivers/1/location
{
  "driverLocation": {
    "lat": 40.742
  },
  "fieldMask": ["lat"]
}
```

Sách nhấn mạnh rằng `GetDriverLocation` nhận identifier đầy đủ như `drivers/1/location`, không phải chỉ nhận parent id như `drivers/1`. 

Điều này rất quan trọng: ta đang thao tác trực tiếp với `DriverLocation`, không phải update gián tiếp thông qua `Driver`.

**CREATE**

Không có `CreateDriverLocation`.

Lý do: singleton sub-resource tồn tại tự động khi parent tồn tại. Khi tạo `Driver`, hệ thống ngầm hiểu `drivers/1/location` cũng tồn tại. Sách nói singleton sub-resource “comes into existence exactly when the parent resource does”, tức nó sinh ra cùng resource cha. 

Vì vậy flow hợp lý là:

```http
POST /drivers
```

server trả về:

```json
{
  "id": "drivers/1",
  "name": "Alice",
  "licensePlate": "ABC-123"
}
```

Sau đó có thể gọi:

```http
GET /drivers/1/location
```

Nhưng lưu ý: không nên cho phép tạo `Driver` và set luôn `DriverLocation` trong cùng một `CreateDriver`. Vì mục tiêu của pattern là tách hai phần này ra. Sách nói việc tạo `Driver` và set thông tin `DriverLocation` trong cùng một call không nên được hỗ trợ; thay vào đó cần chọn default hợp lý cho singleton sub-resource. 

**DELETE**

Không có `DeleteDriverLocation`.

Khi xóa parent, singleton sub-resource bị xóa theo. Ví dụ:

```http
DELETE /drivers/12345
```

sau đó:

```http
GET /drivers/12345/location
```

sẽ trả về `404 Not Found`. Sách mô tả hành vi này giống cascading delete: delete parent thì các singleton sub-resource gắn với nó cũng bị xóa. 

Điểm cần nhớ: singleton sub-resource **không có đời sống độc lập**. Nó sống và chết cùng parent.

**LIST**

Không có `ListDriverLocations`.

Vì với mỗi driver chỉ có đúng một `location`, nên list là vô nghĩa. Nếu bạn cần list nhiều item, đó không còn là singleton sub-resource nữa mà là subcollection bình thường.

Ví dụ nếu muốn lưu lịch sử vị trí, nên thiết kế:

```http
GET /drivers/1/locationHistory
GET /drivers/1/locationHistory/{locationRecordId}
```

chứ không dùng singleton `location`.

### 12.3.2 Resetting

Sách khuyến nghị singleton sub-resource thường nên có custom method để reset về trạng thái mặc định. Summary của chương nói singleton sub-resource nên hỗ trợ custom `reset` method để restore attributes về default state. 

Ví dụ:

```http
POST /drivers/1/location:reset
```

Tại sao không dùng `DELETE /drivers/1/location` để reset? Vì `DELETE` có nghĩa là xóa resource, trong khi singleton sub-resource không được xóa độc lập. Nó luôn tồn tại cùng parent. Nếu dùng `DELETE` để “reset”, API sẽ gây hiểu nhầm: client tưởng resource biến mất, nhưng thực tế nó vẫn tồn tại với giá trị mặc định.

Vì vậy `reset` rõ nghĩa hơn: “hãy đưa sub-resource này về default”, không phải “hãy xóa nó”.

### 12.3.3 Hierarchy

Singleton sub-resource phải nằm dưới một parent resource cụ thể. Không nên có singleton ở root như:

```http
/config
```

hoặc:

```http
/globalSettings
```

Sách giải thích rằng global singleton giống như một global shared lock: một resource duy nhất được chia sẻ bởi mọi consumer, gây write contention lớn vì tất cả ghi vào cùng một chỗ. 

Cũng không nên để singleton sub-resource làm parent của singleton sub-resource khác. Ví dụ không nên:

```http
/drivers/1/location/metadata
```

nếu `location` là singleton và `metadata` cũng là singleton. Sách khuyên nếu có singleton dưới singleton, hãy đưa chúng thành sibling bên dưới parent chính. 

Nên làm:

```http
/drivers/1/location
/drivers/1/locationMetadata
```

hoặc đặt tên hợp lý hơn tùy domain.

### 12.3.4 Final API definition

Ví dụ cuối chương là Ride Sharing API. `Driver` không chứa field `location`; thay vào đó có `DriverLocation` riêng với id phụ thuộc vào id của `Driver`, ví dụ `drivers/1/location`. API có các method cho `Driver`, và chỉ có `GET`/`PATCH` cho `DriverLocation`, không có create/list/delete riêng cho location. 

Dạng REST có thể hiểu như sau:

```http
GET    /drivers
POST   /drivers
GET    /drivers/{driverId}
PATCH  /drivers/{driverId}
DELETE /drivers/{driverId}

GET    /drivers/{driverId}/location
PATCH  /drivers/{driverId}/location
```

Không có:

```http
POST   /drivers/{driverId}/location
DELETE /drivers/{driverId}/location
GET    /drivers/{driverId}/locations
```

## 12.4 Trade-offs

Pattern này có lợi vì cô lập dữ liệu, nhưng đổi lại mất một số khả năng.

### 12.4.1 Atomicity

Khi tách `DriverLocation` ra khỏi `Driver`, ta không còn thao tác atomic trên cả hai trong cùng một request. Ví dụ trước đây nếu `location` là field của `Driver`, ta có thể tạo driver và set location trong cùng một `POST /drivers`. Sau khi tách ra, phải tạo `Driver` trước, rồi update `DriverLocation` sau.

Sách nói đây là một limitation có chủ ý: mục tiêu chính của việc tách sub-resource là cô lập nhóm dữ liệu đó, có thể vì nó lớn hoặc có yêu cầu bảo mật riêng. 

Nói cách khác: mất atomicity, nhưng đổi lại API rõ ràng hơn, dễ phân quyền hơn, và tránh parent resource bị kéo theo dữ liệu không cần thiết.

### 12.4.2 Exactly one sub-resource

Một trade-off khác: đã gọi là singleton thì mỗi parent chỉ có **một** sub-resource loại đó. Sách nhấn mạnh khác với subcollection có thể chứa nhiều sub-resources cùng loại, pattern này chỉ cho phép một instance duy nhất. 

Ví dụ:

```http
/drivers/1/location
```

hợp lý nếu nói về vị trí hiện tại.

Nhưng nếu cần nhiều vị trí theo thời gian, không nên dùng singleton:

```http
/drivers/1/location
```

mà nên dùng collection:

```http
/drivers/1/locationRecords/{recordId}
```

## 12.5 Exercises

Các câu hỏi cuối chương xoay quanh việc bạn có thật sự cần tách data ra hay không: data lớn đến mức nào thì nên tách, vì sao singleton chỉ hỗ trợ `get` và `update`, vì sao dùng `reset` thay vì `delete`, và vì sao singleton không nên làm parent của singleton khác. 

Tóm lại, công thức nhớ nhanh là:

```text
Singleton sub-resource =
  dữ liệu thuộc về parent
+ chỉ có đúng 1 bản thể cho mỗi parent
+ cần URL và quyền thao tác riêng
- không create riêng
- không delete riêng
- không list
```

Ví dụ tốt: `drivers/{id}/location`, `documents/{id}/acl`, `users/{id}/preferences`.

Ví dụ không tốt: `drivers/{id}/locations` nếu có nhiều location record; lúc đó nên dùng subcollection chứ không phải singleton.
