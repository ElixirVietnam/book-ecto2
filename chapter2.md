# Schemaless queries

Phần lớn các queries trong Ecto được viết bằng cách sử dụng Schema. Ví dụ, để lấy ra tất cả các posts trong một Database, có thể viết như sau:

```elixir
MyApp.Repo.all(Post)
```

Trong câu query trên, Ecto biết tất cả các trường và kiểu của từng trường trong `Post` schema, chúng ta có thể viết lại câu query trên mà không dùng schema như sau

```elixir
MyApp.Repo.all(from p in Post, select: %Post{title: p.title, body: p.body, ...}}
```

Ở những ngày đầu tiên Ecto được thiết kế, Schema chưa tồn tại. Những queries lúc đó được viết bằng cách truyền trực tiếp Database table vào câu query thông qua một string như sau:

```elixir
MyApp.Repo.all(from p in "posts", select: {p.title, p.body})
```

Những câu query không sử dụng schema như 2 câu query kể trên, được gọi là Schemaless query. Khi viết schemaless query, chúng ta phải chỉ rõ ra những trường mà chúng ta muốn select từ database.

Mặc dù các câu schemaless query rất flexible, nhưng chúng khá là verbose, các lập trình viên phải viết đi viết lại các câu select. Không chỉ vậy, Ecto 1.0 không cho phép bạn insert thêm bản ghi vào Database mà không có schema.

Ecto 2.0 hỗ trợ rất nhiều tính năng mới với các schemaless queries, không chỉ cải thiện cú pháp để đọc và update dữ liệu, mà còn cho phép các hoạt động của Database có thể biểu diễn mà không cần schema

## insert\_all

Một trong những hàm mới được thêm vào Ecto 2.0 đó là `Ecto.Repo.insert_all/3`. Với `insert_all`, các lập trình viên có thể insert nhiều bản ghi một lúc vào trong một Repo

```elixir
MyApp.Repo.insert_all(Post, [
  [title: "hello", body: "world"],
  [title: "another", body: "post"],
])
```

Các hàm `*_all` là cho phép các lập trình viên đọc, tạo mới, cập nhận và xoá nhiều đối tượng một lúc. Hãy xem tiếp một vài ví dụ

Nếu bạn muốn viết một bản báo cáo, nó khá là khó khăn để nghĩ rằng làm sao các Schema trong ứng dụng của bạn lại ánh xạ với một bản báo cáo được sinh ra. Do đó, đơn giản hơn chúng ta sẽ viết một câu query chỉ trả về dữ liệu chúng ta cần mà không cần cố gắng gắn dữ liệu đó với bắt cứ Schema nào cả

```elixir
def running_activities(start_at, end_at)
  MyApp.Repo.all(
    from u in "users",
      join: a in "activities",
      on: a.user_id == u.id,
      where: a.start_at > type(^start_at, Ecto.DateTime) and
             a.end_at < type(^end_at, Ecto.DateTime),
      group_by: a.user_id,
      select: %{
        user_id: a.user_id, 
        interval: a.start_at - a.end_at, 
        count: count(u.id)
      }    
  )
end
```

Hàm trên không phụ thuộc vào bất cứ Schema nào cả, nó trả về đúng những dữ liệu liên quan tới bản báo cáo. Chú ý cách chúng ta sử dụng hàm `type/2` để xác định kiểu mong muốn của tham số truyền vào.

Các hoạt động thêm, cập nhật và xoá cũng được thực hiện thông qua các hàm `insert_all`, `update_all`, và `delete_all` tương ứng.

```elixir
# Insert data into posts and returns its ID
[%{id: id}] = 
  MyApp.Repo.insert_all "posts", [[title: "hello"]], returning: [:id]

# User the ID to trigger updates
post = from p in "posts", where: [id: ^id]
{1, _} = MyApp.Repo.update_all post, set: [title: "new_title"]

# As well as for deleting
{1, _} = MyApp.Repo.delete_all post
```

Không khó để thấy rằng những hoạt động này được ánh xạ trực tiếp tới các câu query SQL tương ứng. Điều này giúp bạn có thể tương tác trực tiếp với Database mà không cần tạo các Schema không cần thiết

## Các câu queries đơn giản hơn

Bên cạnh việc hỗ trợ các câu query schemaless để insert, update và delete, Ecto 2.0 cũng khiến các câu query schemaless trở nên dễ đọc hơn.

Một ví dụ của việc này là khả năng để select tất cả các trường mong muốn mà không phải lặp lại. Trong các version trước, bạn phải viết một câu query select như sau:

```elixir
from p in "posts", select: %{title: p.title, body: p.body}
```

Với Ecto 2.0, bạn đơn giản chỉ cần truyền vào một list các trường mong muốn:

```elixir
from p in "posts", select [:title, :body]
```

Hai câu queries ở trên là tương đương với nhau. Khi nhận được một list các trường, Ecto sẽ tự động chuyển list các trường này thành một `map` hoặc `struct`. 

Việc hỗ trợ truyền vào một list các trường hoặc một keyword list đã được thêm vào hầu hết các câu query của Ecto 2.0. Ví dụ, chúng ta có thể sử dụng một câu query update để thay đổi title của tất cả các post mà không dùng schema:

```elixir
def update_title(post, new_title) do
  query = from "posts" where: [id: ^post.id], update: [set: [title: ^new_title]]
  MyApp.Repo.update_all(query)
end
```

Hàm `update` cũng hỗ trợ bốn câu lệnh:

+ `:set` - thay đổi giá trị của một cột với một gía trị cụ thể
+ `:inc` - tăng giá trị của côt lên một giá trị cụ thể
+ `:push` - đẩy một giá trị mới vào mảng
+ `:pull` - xoá một giá trị cụ thể ra khỏi mảng

Ví dụ, chúng ta có thể tăng giá trị của cột một cách tự động bằng cách sự dụng lệnh `:inc` (sử dụng hoặc không sử dụng schema)

``` elixir
def increment_page_views(post) do
  query = from "posts", where: [id: ^post.id], update: [inc: [page_views: 1]]
  MyApp.Repo.update_all(query)
end
```

Bằng cách cho phép các cấu trúc dữ liệu thông thường có thể truyền vào hấu hết các hoạt động của query, Ecto 2.0 làm cho các câu query trở nên dễ dàng hơn. Không chỉ vậy, nó còn cho phép các lập trình viên có thể viết ra các cậu query động, khi mà các fields, các điều kiện filters, hoặc ordering không thể xác định trước được. Chúng ta sẽ khám phá chi tiết những tính năng này trong các chương tiếp theo. Hiện tại, hãy cùng tiếp tục xem cách sử dụng schemas trong ngữ cảnh của Changeset