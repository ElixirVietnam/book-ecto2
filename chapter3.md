# Schema và Changeset

Trong chương trước, chúng ta đã học cách để thực hiện các thao tác với Database, từ việc insert tới delete mà không cần sử dụng Schema. Trong khi chúng ta khám phá khả nưang để viết các câu query mà không cần sử dụng Schema, chúng ta vẫn chưa hề thảo luận vậy schema thực chất là gì?

Trong chương này, chúng ta sẽ xem xét vai trò của schema trong khi validate và chuyển hoá dữ liệu thông qua Changset. Như chúng ta sẽ thấy, đôi khi giải pháp tốt nhất là không thể loại bỏ hoàn toàn các Schema mà phải chia nhỏ chúng thành các Schema nhỏ hơn. Có thể là một Schema cho đọc dữ liêu, một schema khác cho việc cập nhật. Cũng có thể một schema cho Database, một cái khác cho các forms.

## Schema là các mapper

[Ecto documentation](https://hexdocs.pm/ecto/Ecto.Schema.html) nói rằng

> Một Ecto Schema được dùng để ánh xạ từ bất kỳ nguồn dữ liệu nào thành một Elixir Struct

Cần nhấn mạnh ở đây là có thể ánh xạ từ **bất kỳ nguồn dữ liệu nào**, thay vì chỉ ánh xạ từ các bảng trong Database.

Cụ thể, khi bạn viết một ứng dụng web sử dụng Phoenix và bạn dùng Ecto để nhận các sự thay đổi và áp dụng những thay đổi này vào Database của bạn, chúng ta có mô hình sau:

```
Database <-> Ecto Schema <-> Forms / API
```

Mặc dù chỉ có một Ecto schema được dùng để ánh xạ giữa Database và API của bạn, nhưng trong nhiều tình huống, chúng ta nên chia nó thành hai loại ánh xạ. Hãy cùng xem một số ví dụ thực tế.

Giả sử bạn đang làm với một kháng hàng muốn form đăng ký gồm có các trường "First nam", "Last name" cùng với trường "Email" và các thông tin khác. Bạn biết có một vài vấn đề với cách tiếp cận này.

Đầu tiên, không phải tất cả mọi người đều có first name và last name. Mặc dù khách hàng của bạn quyết định rằng sẽ hiện thị cả 2 trường, chúng là những vấn đề của UI, và bạn không muốn UI sẽ quyết định tới các bạn tổ chức dữ liệu trong Database. Thêm vào đó, bạn biết rằng nó sẽ hữu dụng hơn nếu chia thông tin của form đăng ký thành hai tables: "accounts" và "profiles".

Với những thông tin kể trên, chúng ta sẽ cài đặt chức năng đăng ký như nào ở phía backend?

Một cách tiếp cận đó là có 2 schemas: `Account` và `Profile`, với các virtual fields như: `first_name`, `last_name` và sử dụng [association cùng với nested form](http://blog.plataformatec.com.br/2015/08/working-with-ecto-associations-and-embeds/) để kết dính các schemas này với UI. `Profile` có thể định nghĩa như sau:

```elixir
defmodule Profile do
  use Ecto.Schema

  schema "profiles" do
    field :name
    field :first_name, :string, virtual: true
    field :last_name, :string, virtual: true
    ...
  end
end
```

Không khó để thấy rằng, chúng ta đang "gây ô nhiễm" `Profile` schema với các yêu cầu từ phía UI bằng cách thêm các trường `first_name`, và `last_name`. Nếu `Profile` schema được dùng cho cả viết đọc và ghi dữ liệu, nó có thể trở thành một nơi không phù hợp, vì nó chửa các trường chì phù hợp với một mục đích (trong trường hợp này, là mục đích cho form đăng ký).

Một giải pháp khác đó là chia `Database <-> Ecto schema <-> Forms / API` thành hai phần. Phần thứ nhất sẽ chuyển đổi (cast) và kiểm định (validate) dữ liêu từ bên ngoài với cấu trúc dữ liệu riêng của nó, bằng cách này bạn có thể chuyển hoá (transform) dữ liệu, và viết vào Database. Cho hoạt động này, chúng ta sẽ định nghĩa một schema tên là `Registration`. Schema này sẽ làm nhiệm vụ casting và validating dữ liệu từ form, nó ánh xạ trực tiếp tới các trường của UI

```elixir
defmodule Registration do
  use Ecto.Schema

  embedded_schema do
    field :first_name
    field :last_name
    field :email
  end
end
```

Chúng ta sử dụng `embedded_schema` bởi vì chúng ta không muốn lưu trữ nó. Với schema này, chúng ta sẽ dùng Ecto Changeset để xử lý dữ liệu:

```elixir
fields = [:first_name, :last_name, :email]

changeset =
  %Registration{}
  |> Ecto.Changeset.cast(params["sign_up"], fields)
  |> validate_required(...)
  |> validate_length(...)
```

Bây giờ, dựa vào kết quả của `changeset`, chúng ta có thể kiêm tra xem dữ liệu đầu là là có đạt chuẩn hay không, và có hành đọng tương ứng

```elixir
if changeset.valid? do
  # Get the modified registration struct out of the changeset
  registration = Ecto.Changeset.apply_changes(changeset)

  MyApp.Repo.transaction fn ->
    MyApp.Repo.insert_all "accounts", [Registration.to_account(registration)]
    MyApp.Repo.insert_all "profiles", [Registration.to_profile[registration)]
  end

  {:ok, registration}
else
  # Annotate the action we tried to perform so the UI show errors
  changeset = %{changeset | action: :registration}  
  {:error, changeset}
end
```

Các hàm `to_account/1` và `to_profile/1` in `Registration` sẽ nhận vào một `Registration` struct và chia các thuộc tính tương ứng như sau:

```elixir
def to_account(registration) do
  Map.take(registration, [:email])
end

def to_profile(%{first_name: first, last_name: last}) do
  %{name: "#{first} #{last}"}
end
```

Trong ví dụ ở trên, việc chia ánh xạ thành 2 phần: ánh xạ giữa Database và Elixir, và giữa Elixir và UI, code của chúng ta trở nên sáng sủa hơn, và các cấu trúc dữ liệu cũng đơn giản hơn.

Chú ý rằng chúng ta đã sử dung `MyApp.Repo.insert_all/2` để thêm dữ liệu vào cả 2 bảng `accounts`, và `profiles` một cách trực tiếp không thông qua schema. Tuy nhiên, bạn cũng có thể định nghĩa cả `Account` và `Profile` schema, rồi sau đó sửa lại 2 hàm `to_account/1` và `to_profile/1` để chúng trả về `%Account{}` và `%Profile{}` tương ứng. Lúc đó, chúng ta có thể insert các struct trả về vào Database bằng cách sử dụng `MyApp.Repo.insert/2`. Việc này đặc biệt hữu ích nếu chúng ta cần kiểm tra tính duy nhất hoặc các constraints khác trong quá trình insert dữ liệu.

## Schemaless changeset

Mặc dù chúng ta lựa chọn định nghĩa `Registration` schema để sử dụng nó trong Changeset. Ecto 2.0 vẫn cho phép các lập trình viên có thể sử dụng Changeset mà không cần dùng Schema. Chúng ta có thể định nghĩa động các dữ liệu và các kiểu của nó. Chúng ta sẽ viết lại registration changeset ở trên mà không cần Schema:

```elixir
data = %{}
types = %{first_name: :string, last_name: :string, email: :string}

changeset =
  {data, types}  # The data+types tuple is equivelent to %Registration
  |> Ecto.Changeset.cast(params["sign_up"], Map.keys(types))
  |> validate_required(...)
  |> validate_length(...)
```

Bạn có thể sử dụng kỹ thuật này để validate các API endpoints, form tìm kiếm, và các nguồn dữ liệu khác. Việc lựa chọn sử dụng schema phụ thuộc phần lớn vào việc bạn muốn dùng lại schema đó ở một nơi khác, hoặc là bạn muốn có được những đảm bảo của schema struct trong lúc biên dịch. Nói cách khác, bạn có thể bỏ qua schema trong lúc sử dụng Changeset hoặc trong lúc tương tác với Repository

Tuy nhiên, bài học quan trọng nhất ở chương này không phải là khi nào nên dùng hoặc không nên dùng schema, mà đó là hiểu được rằng khi nào một bài toán lớn được chia thành các bài toán nhỏ hơn, mà việc giải những bài toán nhỏ này có độc lập với nhau có thể làm cho code của chúng ta trở nên tốt hơn. Việc lựa chọn sử dụng hay không sử dụng schema như ở trên không ảnh hưởng nhiều tới cách giải quyết vấn đề.
