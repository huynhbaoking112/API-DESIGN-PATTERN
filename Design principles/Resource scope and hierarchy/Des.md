Phần **Resource scope and hierarchy** nói về cách **chọn resource nào nên tồn tại trong API, và các resource đó nên liên hệ với nhau như thế nào**. Đây là chương 4 trong Part 2. Ý chính: khi thiết kế API theo hướng resource-oriented, không chỉ đặt tên resource là đủ; ta còn phải quyết định resource nào là cha, resource nào là con, resource nào chỉ nên tham chiếu, resource nào nên được nhúng trực tiếp. 

## 1. Resource layout là gì?

**Resource layout** là cách ta sắp xếp các resource trong API: resource nào tồn tại, resource có field gì, và chúng liên hệ với nhau ra sao. Nó khá giống việc thiết kế schema database hoặc entity relationship model.

Ví dụ với chat app, ta có thể có:

```ts
interface ChatRoom {
  id: string;
  name: string;
  memberIds: string[];
}

interface User {
  id: string;
  name: string;
}
```

Ở đây ta phải quyết định: `ChatRoom` có nên chứa danh sách user không? User có nên biết mình thuộc những phòng nào không? Message có thuộc ChatRoom không? Đó chính là vấn đề của resource layout.

Trong hình 4.1 và 4.2, sách minh họa resource bằng các hộp, còn quan hệ giữa chúng bằng các đường nối. Ví dụ hình 4.1 mô tả `ChatRoom` có nhiều `User` là members; hình 4.2 mô tả `User`, `PaymentMethod`, `Address` trong một API mua sắm có nhiều loại quan hệ khác nhau. 

## 2. Các loại relationship chính

### Reference relationship

Đây là quan hệ đơn giản nhất: resource này **trỏ tới** resource khác bằng ID.

Ví dụ:

```ts
interface Message {
  id: string;
  content: string;
  authorId: string;
}
```

`Message` không chứa toàn bộ thông tin của `User`, mà chỉ có `authorId`. Nghĩa là message “tham chiếu” đến user đã viết nó.

Trong sách, ví dụ `Message` có field `author` trỏ tới một `User`. Đây giống foreign key trong database: nhiều message có thể cùng trỏ tới một user. 

### Many-to-many relationship

Đây là khi hai resource có quan hệ nhiều-nhiều.

Ví dụ:

```ts
interface ChatRoom {
  id: string;
  memberIds: string[];
}

interface User {
  id: string;
}
```

Một `ChatRoom` có nhiều `User`, và một `User` cũng có thể tham gia nhiều `ChatRoom`. Đây là quan hệ many-to-many. Sách nhấn mạnh loại quan hệ này rất phổ biến, nhưng cách biểu diễn nó có nhiều trade-off, nên các chương sau sẽ bàn sâu hơn. 

### Self-reference relationship

Đây là khi một resource trỏ tới resource **cùng loại**.

Ví dụ:

```ts
interface Employee {
  id: string;
  managerId: string;
  assistantId?: string;
}
```

`Employee` trỏ tới một `Employee` khác là manager. Trong hình 4.4, sách dùng ví dụ employee có manager và assistant, đều là employee khác. Loại quan hệ này thường xuất hiện trong cấu trúc cây, tổ chức công ty, hoặc social graph. 

### Hierarchical relationship

Đây là quan hệ cha-con, tức resource con **thuộc về** resource cha.

Ví dụ:

```http
/chatRooms/123/messages/456
```

Ở đây `Message` là con của `ChatRoom`. Nếu xóa `ChatRoom`, thường các `Message` bên trong cũng bị xóa theo. Nếu user có quyền truy cập `ChatRoom`, thường họ cũng có quyền xem `Message` trong đó.

Điểm quan trọng: hierarchy không chỉ là “có liên quan”, mà là **ownership / containment**. Sách so sánh với folder trên máy tính: file nằm trong folder; xóa folder thường kéo theo xóa file bên trong. 

## 3. Khi nào nên tạo relationship?

Sách cảnh báo: đừng thấy resource nào liên quan nhau là nối hết lại. Relationship có chi phí dài hạn.

Ví dụ mạng xã hội có `User` follow `User`. Nếu một người nổi tiếng có hàng triệu followers, việc xóa hoặc cập nhật tài khoản đó có thể ảnh hưởng đến hàng triệu record. Vì vậy, relationship không miễn phí.

Quy tắc thực tế: **chỉ tạo relationship nếu nó thật sự cần cho chức năng chính của API**. Không nên tạo chỉ vì “có thể sau này cần”. Với Instagram/Twitter, quan hệ follow là cốt lõi. Nhưng với app nhắn tin đơn giản, việc lưu toàn bộ social graph giữa mọi user có thể không cần thiết. 

## 4. Reference hay in-line data?

Khi một resource cần dùng dữ liệu từ resource khác, có hai cách:

Cách 1: dùng reference.

```ts
interface ChatRoom {
  id: string;
  adminUserId: string;
}
```

Muốn biết tên admin, client phải gọi thêm API lấy user:

```ts
const room = GetChatRoom(id);
const admin = GetUser(room.adminUserId);
```

Cách 2: in-line data.

```ts
interface ChatRoom {
  id: string;
  adminUser: User;
}
```

Muốn biết tên admin thì chỉ cần một request:

```ts
const room = GetChatRoom(id);
return room.adminUser.name;
```

In-line tiện hơn khi dữ liệu đó thường xuyên cần dùng. Nhưng nó có thể làm response phình to, gây tốn bandwidth và compute, đặc biệt nếu `User` cũng chứa nhiều dữ liệu khác. Sách đưa ra quy tắc: **optimize cho common case, nhưng đừng làm advanced case trở nên bất khả thi**. Nếu thông tin admin hiếm khi được xem, chỉ lưu `adminUserId` có thể tốt hơn. 

## 5. Khi nào nên dùng hierarchy?

Dùng hierarchy khi quan hệ cha-con thật sự có ý nghĩa về hành vi.

Ví dụ tốt:

```http
/chatRooms/{chatRoomId}/messages/{messageId}
```

Vì message thuộc về đúng một chat room. Khi xóa chat room, thường message cũng nên biến mất. Khi cấp quyền vào chat room, thường cũng cấp quyền xem message.

Không nên dùng hierarchy nếu resource con có thể thuộc nhiều cha hoặc có thể chuyển cha thường xuyên. Sách nói child resource chỉ nên có một parent. Nếu `Message` cần xuất hiện trong nhiều `ChatRoom`, thì không nên model nó là child resource của một chat room duy nhất. 

Một điểm hay trong hình 4.12: `RetentionPolicy` là con của `Company`, nhưng nhiều `ChatRoom` có thể reference tới cùng một `RetentionPolicy`. Nghĩa là một resource vẫn có thể có parent, đồng thời được resource khác tham chiếu. Hierarchy và reference có thể cùng tồn tại.

## 6. Entity relationship diagram

Phần này giải thích cách đọc sơ đồ quan hệ giữa resource. Các hộp là resource; đường nối biểu diễn quan hệ; ký hiệu ở đầu đường cho biết “một” hay “nhiều”.

Ví dụ trong hình 4.6: một `School` có nhiều `Student`, mỗi `Student` thuộc một `School`; đồng thời `Student` và `Class` là many-to-many vì một student học nhiều class, một class có nhiều student. Hình 4.7 thêm vòng tròn để biểu diễn quan hệ optional, tức có thể là zero hoặc many. 

Điểm cần nhớ: khi đọc sơ đồ, bắt đầu từ một resource, đi theo đường nối sang resource kia, rồi nhìn ký hiệu ở đầu bên kia để hiểu số lượng.

## 7. Các anti-pattern cần tránh

### Anti-pattern 1: Resources for everything

Không phải khái niệm nào cũng cần thành resource riêng.

Ví dụ API annotation cho image có các khái niệm:

```text
Image
Annotation
BoundingBox
Note
```

Nếu biến tất cả thành resource, ta phải có CRUD cho cả `BoundingBox` và `Note`, làm API phức tạp không cần thiết.

Sách gợi ý: `Image` và `Annotation` có thể là resource thật; còn `BoundingBox`, `Point`, `Note` có thể chỉ là data type nằm trong `Annotation`. Nếu không cần thao tác độc lập với một khái niệm, đừng vội biến nó thành resource. 

Ví dụ tốt hơn:

```ts
interface Annotation {
  id: string;
  imageId: string;
  boundingBox: BoundingBox;
  notes: Note[];
}

interface BoundingBox {
  bottomLeftPoint: Point;
  topRightPoint: Point;
}

interface Note {
  content: string;
  createTime: Date;
}
```

`BoundingBox` không cần URL riêng như:

```http
/boundingBoxes/123
```

nếu nó chỉ có ý nghĩa bên trong annotation.

### Anti-pattern 2: Deep hierarchies

Hierarchy quá sâu làm API khó hiểu và khó dùng.

Ví dụ xấu:

```http
/universities/{u}/schools/{s}/courses/{c}/semesters/{sem}/documents/{doc}
```

Muốn lấy một document, client phải biết quá nhiều parent. Sách đưa ví dụ course catalog: nếu đưa `University → School → Course → Semester → Document` vào một cây sâu, technically vẫn chạy, nhưng rất khó reasoning và khó nhớ. 

Thiết kế tốt hơn là làm hierarchy nông hơn:

```http
/universities/{u}/courses/{c}/documents/{doc}
```

và biến `schoolId`, `semester` thành field hoặc reference/filter:

```ts
interface Course {
  id: string;
  universityId: string;
  schoolId: string;
  semester: string;
}
```

Ý chính: chỉ dùng hierarchy cho những tầng thực sự cần behavior cha-con như quyền truy cập, cascade delete, ownership.

### Anti-pattern 3: In-line everything

Ngược lại với “resources for everything”, một lỗi khác là nhúng tất cả dữ liệu vào một resource.

Ví dụ:

```ts
interface Book {
  id: string;
  name: string;
  authors: Author[];
}
```

Nếu author được nhúng vào từng book, khi tên author đổi thì sao? Có cập nhật tất cả sách của author đó không? Nếu không, dữ liệu bị lệch. Nếu có, update sẽ phức tạp.

Thiết kế an toàn hơn thường là:

```ts
interface Book {
  id: string;
  name: string;
  authorIds: string[];
}

interface Author {
  id: string;
  name: string;
}
```

Sách nhấn mạnh vấn đề ở đây là **data integrity**: dữ liệu dùng chung mà bị copy nhiều nơi sẽ dễ không nhất quán. 

## Tóm tắt dễ nhớ

Phần này có thể nhớ bằng vài câu:

**Resource layout** là cách sắp xếp resource và quan hệ giữa chúng trong API.

**Reference** dùng khi resource chỉ cần trỏ tới resource khác.

**In-line** dùng khi dữ liệu nhỏ, thường xuyên cần, và không gây vấn đề đồng bộ.

**Hierarchy** dùng khi có ownership thật sự: parent xóa thì child cũng nên xóa, parent có quyền thì child kế thừa quyền.

**Đừng tạo resource cho mọi thứ**, vì API sẽ phình to.

**Đừng tạo hierarchy quá sâu**, vì API khó hiểu và khó dùng.

**Đừng nhúng tất cả**, vì dữ liệu dùng chung sẽ dễ mất nhất quán.

Nói ngắn gọn: **chọn resource và quan hệ không chỉ để mô tả dữ liệu, mà để tạo ra API dễ hiểu, dễ mở rộng và ít gây lỗi về sau.**
