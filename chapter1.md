# Ecto không phải là thư viện ORM

Elixir không phải là ngôn ngữ lập trình hương đối tượng, chính vì thế Ecto cũng không thể là một thư viện ORM \(Object relational Mapper\).

## Object

Đối tượng bao hàm trạng thái \(`state`\) và các hành vi \(`behaviour`\). Trong cùng một đối tượng `user`, bạn vừa có thể có dữ liệu, giống như `user.name`, vừa có thể có các hành vi của đối tượng, ví dụ như xác nhận một tài khoản thông qua một phương thức `user.confirm()`.

Tuy nhiên điề này là không thể với Elixir. Trong Elixir, chúng ta làm việc với các kiểu dữ liệu khác nhau như `tuple`, `list`, `map`, ... Các hành vi không thể được gắp với đối tượng của dữ liệu như vậy. Hành vi trong Elixir luôn luôn được thêm vào các modules thông qua các hàm.

Để làm việc với một kiểu dữ liệu có cấu trúc, Elix cung cấp các `struct`. Struct định nghĩa một tập các trường. Một `struct` sẽ được tham chiếu bởi tên của module mà nó được định nghĩa bên trong

```elixir
defmodule User do
  defstruct [:name, :email]
end

user = %User{name: "Kien Nguyen", email: "trungkien2288@gmail.com"}
```

Một khi `user` được tạo ra, chúng ta có thể truy vập vào email của nó bằng cách gọi `user.email`. Một điểm cần nhấn mạnh là `struct` trong Elixir chỉ chứa dữ liệu. Chúng ta không thể nào gọi một hàm `user.confirm()` với một instance của `User` như cách chúng ta gọi trong các ngôn ngữ lập trình hướng đối tượng.

Mặc dù vậy, chúng ta vẫn có thể thêm các hàm vào cùng một module định nghĩa struct như sau

```elixir
defmodule User do
  defstruct [:name, :email]

  def confirm(user) do
    # Code to confirm the user 's email
  end
end
```

Bằng cách định nghĩa như trên, hàm `confirm` được định nghĩa là một hàm của module `User`. Thực tế, khi gọi hàm `confirm`, chúng ta bắt buộc phải gọi nó một cách chính xác là `User.confirm` \(trong trường hợp, bạn sử dụng `alias`, bạn có thể bỏ qua `User`, nhưng lúc đó hàm vẫn dược hiểu là hàm `confirm` của module User\`\). Mọi hàm trong Elixir băt buộc phải thuộc vào một module. Và trong trường hợp chúng ta muốn thêm hành vi cho các struct trong Elixir, tham sô đầu tiên của hàm luôn luôn phải là struct mà chúng ta muốn gọi tới. Nhấn mạnh một lần nữa, trong Elixir chúng ta không có **methods** mà chỉ có các **functions**

Với việc không hề có các đối tượng trong Elixir, hiển nhiên Ecto không thể trả thành thư viên ORM. Tuy nhiên, liệu Ecto có làm được phần việc còn lại của một thư viện "Relational Mapper"

## Relational Mappers

ORM là một kỹ thuật để chuyển dổi dữ liệu giữa các hệ thống không tương thích trực tiếp với nhau, thông thường là giữa Databases tớicác objects và ngược lại

Tương tự, Ecto cung cấp các `Schema` để ánh xạ từ bất cứ nguồn dữ liệu nào tới một Elixir struct. Khi áp dụng vào các Database, Ecto Schema sẽ trở thành các Relational Mappers. Việc ánh xạ từ từ các Database vào Schema chỉ là một trong nhiều chức năng mà Ecto muốn cung cấp. Với sự linh hoạt trong thiết kế của mình, Ecto là một **tập các công cụ để giúp lập trình viên làm việc với cả nguồn dữ liệu khác nhau**, và Database là một nguồn phổ biến.

Ví dụ, schema bên dưới sẽ ánh xạ các trường `name`, `email`, `inserted_at` và `updated_at` tới các cột tương ứng trong bảng `users` của Database

```elixir
defmodule User do
  use Ecto.Schema

  schema "users" do
    field :name
    field :email
    timestamps()
  end
end
```

Bằng Schema, chúng ta định dạng cấu trúc của dữ liệu một lần, và sau đó chúng ta có thể sử dụng cấu trúc này để lấy dữ liệu từ Database cũng như cập nhật những sự thay đổi từ dữ liệu vào Database:

```elixir
MyApp.User
|> MyApp.Repo.get!(13)
|> Ecto.Changeset.cast([name: "new_name"], [:name, :email])
|> MyApp.Repo.update!
```

Bằng cách dựa vào các thông tin của Schema, Ecto sẽ biết cách để đọc và ghi dữ liệu mà không cần thêm các thông tin khác từ lập trình viến \(các thông tin khác ở đây bao gồm: kiểu dữ liệu là gì? table trong Database là gì, ...Các thông tin này đã được định nghĩa ngay khi bạn định nghĩa một Ecto Schema\). Trong những ứng dụng nhỏ, điều này tạo nên sự kết dính giữa các Schema và các table trong Database mà nó đại diện.

Một Schema của Ecto không nhất thiết phải ánh xạ toàn bộ các trường của nó với tables trong Database. Ngay từ đầu chương, chúng tối muốn nhấn mạnh "Ecto không phải là ORM" nhằm mục đích muốn phân biệt Ecto với các thư viện ORM khác \(Active Record của Rails hay Django ORM\), đồng thời muốn các lập trình viên tránh việc sử dụng Ecto một cách sai trái như trong các thư viện khác.

Sau đây là một số vấn đề thường gặp phải khi sử dụng Ecto:

* Các dự án sử dụng Ecto có thể dẫn tới tình trạng `God Schemas`, hay thường được biết với tên `God Models`, `Fat Models` hoặc `Canonical Models` trong các ngôn ngữ và frameworks khác. Những schema này thường chứa hàng trăm trường, và thường biển thị  những quyết định kém về mặt thiết kế ở tằng dữ liệu. Thay vì cung cấp một schema với rất nhiều trường phục vụ nhiều ý nghĩa khác nhau, tốt hơn chúng ta nên chia nhỏ schema dựa vào các ngữ cảnh. Ví dụ, thay vì có một `MyApp.User` với hàng tá trường, có thể cân nhắc việc chia nó thành `MyApp.Account.User`, `MyApp.Purchases.User`, ... Mỗi struct chỉ chứa các trường nằm trong ngữ cảnh của nó mà thôi.

+ Các lập trình viện thường dựa dẫm nhiều vào schema, trong khi cách tốt nhất để lấy dữ liệu từ Database là dựa vào các cấu trúc dũ liệu thống thường (giống như `map`, `tuple` hoặc các struct chưa được định nghĩa trước). Ví dụ, khi gọi một hàm searchs, hoặc sinh ra các báo cáo, không có lý do gì để phụ thuộc hoặc trả về các schema từ những câu query nào vì chúng thường dựa vào dữ liệu từ nhiều tables với những yêu cầu khác nhau.

+ Các lập trình viên thường cố gắng sử dụng cùng một schema cho những hoạt động khá khác biệt về mặt cấu trúc. Rất nhiều ứng dụng thường gắn các tính năng như đăng ký, đăng nhập vào một User scheam, trong khi việc xử lý những hành động này là riêng biết. Cách tốt hơn đó là chúng ta có thể chia ra mỗi hành động sử dụng một schema (và schema này có thể không cần ánh xạ với một bảng trong cơ sở dữ liệu)

Trong hai chương kế tiếp, chúng ta sẽ chia nhỏ những "bad practices" kể trên bằng cách khám phá cách sư dụng Ecto mà không sử dụng hoặc sử dụng nhiều schema trong một ngữ cảnh. Thông qua việc học cách `insert`, `delele`, thay đổi và `validate` dữ liệu với Ecto.
