# Quan hệ nhiều nhiều và `casting`

Bên cạnh các mối quan hệ `belongs_to`, `has_one`, `has_many` và `:through`, Ecto 2.0 hỗ trợ cả mối quan hệ `many_to_many`. Quan hệ `many_to_many`, như tên gọi của nó, cho phép X có thể có nhiều association với Y và ngược lại. Mặc dù `many_to_many` có thể được viết thông qua `has_many :through`, tuy nhiên `many_to_many` có thể làm đơn giản hoá một số luồng làm việc.

Trong chương này, chúng ta sẽ nói về mối quan hệ đa hình, và cách mà `many_to_many` có thể giúp loại bớt các đoạn code rườm rà khi so sánh với cách tiếp cận sử dụng `has_many :through`.

## Todo list

Trên web đã có rất nhiều chia sẻ về cách xây dựng ứng dụng "Todo list", nhưng điều này không ngăn được chúng ta tạo ra phiên bản riêng cho mình!

Trong trường hợp của mình, có một khía cạnh của ứng dụng "Todo list" mà chúng ta hứng thú, đó là một quan hệ một "todo list" có thể có nhiều "todo items". Chúng ta đã [khám phá trường hợp này rất chi tiết trong bài viết trên blog của Plataformatice về nested assiciatoins và embed](http://blog.plataformatec.com.br/2015/08/working-with-ecto-associations-and-embeds/). Hãy
cùng điểm lại những điểm chính.

Để mô hình hoá ứng dụng "Todo list", chúng ta sẽ cần 2 schema: `TodoList` và `TodoItem`:

```elixir
defmodule MyApp.TodoList do
  use Ecto.Schema

  schema "todo_lists" do
    field :title
    has_many :todo_items, MyApp.todo_items
    timestamps()
  end
end

defmodule MyApp.TodoItem do
  use Ecto.schema

  schema "todo_items" do
    field :description
    timestamps()
  end
end
```

Một trong những cách để có insert thêm một "todo list" với nhiều items vào trong Database là kết dính UI của chúng ta với schema. Cách tiếp cận này đã được giới thiệu trong bài blog ở trên với Phoenix. Cụ thể như sau:

```elixir
<%= form_for @todo_list_changeset, todo_list_path(@conn, :create), fn f -> %>
  <%= text_input f, :title %>
  <%= inputs_for f, :todo_items, fn i -> %>
    ...
  <% end %>
<% end %>
```

Khi một form được submit trong Phoenix, nó sẽ truyền lên những tham số với định dạng như sau:

```elixir
%{
  "todo_list" => %{
    "title" => "shipping list",
    "todo_items" => %{
      0 => %{"description" => "bread"},
      1 => %{"description" => "egges"},
    }
  }
}
```

Chúng ta có thể nhận vào những tham số kể trên, và truyền nó cho một Ecto changeset, sau đó Ecto tự động xác định được những việc cần phải làm:

```elixir
# In MyApp.TodoList
def changeset(struct, params \\ %{}) do
  struct
  |> Ecto.Changeset.cast(params, [:title])
  |> Ecto.Changeset.cast_assoc(:todo_items, required: true)
end

# And then in MyApp.TodoItem
def changeset(struct, params \\ %{}) do
  struct
  |> Ecto.Changeset.cast(params, [:description])
end
```

Bằng cách gọi `Ecto.Changeset.cast_assoc/3`, Ecto sẽ tìm kiếm khoá "todo_items" trong các tham số truyền vào để ép kiểu, và so sánh những tham số này với các items đã được lưu lại trong cấu trúc của todo list. Ecto cũng tự động sinh ra các chỉ thị để insert, update hoặc delete các todo items như sau:

+ Nếu một todo item được truyền như một tham số có ID, và nó bằng với một todo item đã được gắn với todo list, chúng ta sẽ coi đó là hành động update todo item.
+ Nếu một todo item được truyền vào không có ID (hoặc không bằng với bất cứ ID nào), chúng ta sẽ coi đó là hành động thêm mới một todo item.
+ Nếu một todo item đang được gắn với todo list, nhưng ID của nó không được truyền vào như một tham số, chúng ta coi todo item đó đã được thay thế, và chúng ta sẽ hành động dựa vào callback `:on_replace`. Mặc định `:on_replace` sẽ được gọi, vì thế bạn có thể chọn một cách hành xử giữa việc thay thế, xoá, bỏ quả hoăc là nillifying association.

Lợi điểm của việc sử dụng `cast_assoc/3` đó là **nếu như chúng ta truyền vào dữ liệu chính xác như những định dạng mà Ecto mong muốn**, nó có thể làm tất cả những việc khó để giữ cho các bản ghi được liên kết với nhau. Tuy nhiên, như chúng ta học được ở 3 chương đầu của cuốn sách này, cách tiếp cận này không phải là cách làm mong muốn trong mọi trường hợp, và trong rất nhiều tình huống, chúng ta muốn thiết kế các associations khác biệt hơn, hoặc là phân tách giữa UI với việc biểu diễn Database của chúng ta.

## Đa hình hoá todo items

Giả sử bạn muốn đa hình hoá các "todo item". Ví dụ, bạn muốn có thể thêm các "todo item" vào không chỉ các "todo list" mà còn vào rất nhiều phần khác của ứng dụng, ví dụ như các projects, hoặc các ngày, ...

Đầu tiên, cần nhớ rằng Ecto không cung cấp cùng một loại association đa hình giống như các framework khác như Rails hoặc Laravel. Trong những framework này, một association đa hình sử dụng hai cột, `parent_id` và `parent_type`. Ví dụ, một "todo item" có thể có `parent_id` bằng 1 với `parent_type` là "TodoList", trong khi đó một "todo item" khác có thể có `parent_id` bằng 1 nhưng `parent_type` bằng "Project".

Vấn đề với thiết kế ở trên đó là nó phá vỡ sự tham chiếu trong Database. Database sẽ không còn có khả năng đảm bảo item trong các association tồn tại hoặc sẽ tồn tại trong tương lai. Điều này dẫn tới sự thiếu đồng bộ trong Database, và kết quả là rất nhiều giải pháp tạm thời chỉ đề giải quyết nó.

Thiết kế ở trên cũng đặc biệt không hiệu quả. Trong quá khứ, chúng tôi đã làm việc với rất nhiều khách hàng lớn để loại trừ những kiểu đa hình tham chiếu như vậy bởi vì những câu query đa hình thường xuyên khiến cho Database ngừng hoạt động, kể cả khi đã thêm vào các indexes và tối ưu Database.

May mắn thay, [tài liệu cho macro `belongs_to` bao gồm cả những ví dụ về cách thiết kế một hệ thống đúng đắn cho những associations kiều này](https://hexdocs.pm/ecto/Ecto.Schema.html#belongs_to/3-polymorphic-associations). Một trong những cách tiếp cận đó bao gồm việc sử dụng nhiều bảng nối khác. Bên cạnh các table "todo_list", "project"" và "todo_items", chúng ta có thể tạo thêm các bảng "todo_list_items" và "project_items" để kết nối "todo item" với "todo list" và "todo item" với "project" tương ứng. Chúng ta có thể tạo ra các migration script như sau:

```elixir
create table("todo_lists") do
  add :title
  timestamps()
end

create table("projects") do
  add :name
  timestamps()
end

create table("todo_items") do
  add :description
  timestamps()
end

create table("todo_list_items") do
  add :todo_item_id, references(:todo_items)
  add :todo_list_id, references(:todo_lists)
  timestamps()
end

create table("project_items") do
  add :todo_item_id, references(:todo_items)
  add :project_id, references(:project)
  timestamps()
end
```

Đầu tiên, hãy cùng xem cách cài đặt chức năng này sử dụng một quan hệ `has_many :through` và sau đó sử dụng `many_to_many` để loại bỏ rất nhiều đoạn code thừa mà chúng ta bắt buộc phải sử dụng với cách làm thứ nhất.

## Đa hình với `has_many :through`

Do chúng ta muốn "todo items" có thể đa hình hoá, chúng ta không thể kết nối một "todo list" với một "todo item" trực tiếp. Thay vì vậy, chúng ta sẽ cần tạo ra một schema trung gian để gắn kết `MyApp.TodoList` và `MyApp.TodoItem` với nhau:

```elixir
defmodule MyApp.TodoList do
  use Ecto.Schema

  schema "todo_lists" do
    field :title
    has_many :todo_list_items, MyApp.TodoListItem
    has_many :todo_items, through: [:todo_list_items, :todo_item]
    timestamps()
  end
end

defmodule MyApp.TodoListItem do
  use Ecto.Schema

  schema "todo_list_items" do
    belongs_to :todo_list, MyApp.TodoList
    belongs_to :todo_item, MyApp.TodoItem
    timestamps()
  end  
end

defmodule MyApp.TodoItem do
  use Ecto.Schema

  schema "todo_items" do
    field :description
    timestamps()
  end
end
```

Mặc dù chúng ta sử dụng `MyApp.TodoListItem` như một schema trung gian, `has_many :through` vẫn cho phép chúng ta truy cập tất cả các todo items với một todo list bất kỳ:

```elixir
todo_lists |> Repo.preload(:todo_items)
```

Vấn đề là `:through` association là **read-only** bởi vì Ecto không có đủ thông tin đề tự động điền vào schema trung gian. Điều đó có nghĩa là, nếu chúng ta vẫn muốn sử dụng `cast_assoc` để insert một todo list với nhiều todo items trực tiếp từ UI, chúng ta sẽ phải đầu tiên `cast_assoc(:todo_list_items)` từ `TodoList`, sau đó gọi `cast_assoc(:todo_item)` từ một `TodoListItem` schema:

```elixir
# In MyApp.TodoList
def changeset(struct, params \\ %{}) do
  struct
  |> Ecto.Changeset.cast(params, [:title])
  |> Ecto.Changeset.cast_assoc(:todo_list_items, required: true)
end

# And hen in MyApp.TodoListItem
def changeset(struct, params \\ %{}) do
  struct
  |> Ecto.Changeset.cast_assoc(:todo_item, required: true)
end

# And then in MyApp.TodoItem
def changeset(struct, params \\ %{}) do
  struct
  |> Ecto.Changeset.cast(params, [:description])
end
```

Mọi thứ còn có thể phức tạp hơn, nhớ rằng `cast_assoc` mong muốn một định dạng dữ liệu cụ thể tương ứng với các associations của bạn. Trong trường hợp này, bởi vì các schema trung gian, data gửi lên từ form trong Phoenix cũng phải thay đổi từ "todo_items" thành "todo_list_items" như sau:

```elixir
%{
  "todo_list" => %{
    "title" => "shipping list",
    "todo_list_items" => %{
      0 => %{"todo_item" => %{"description" => "bread"}},
      1 => %{"todo_item" => %{"description" => "egges"}},
    }
  }
}
```

Mọi thứ còn có thể tệ hơn, khi bạn cũng sẽ phải lặp lại logic này với mọi schema trung gian, với việc phải định nghĩa `MyApp.TodoListItem` cho todo list, `MyApp.ProjectItem` cho project, ...

May mắn thay, `many_to_many` sẽ giúp chúng ta loại bỏ những sự dư thừa này.

## Đa hình với `many_to_many`

Ý tưởng đằng sau `many_to_many` là cho phép chúng ta có thể liên kết 2 schema thông qua một schema trung gian, trong khi sẽ tự động lo hết tất cả những chi tiết của schema trung gian. Hãy cùng nhau viết lại schema ở trên với `many_to_many`

```elixir
defmodule MyApp.TodoList do
  use Ecto.Schema

  schema "todo_lists" do
    field :title
    many_to_many :todo_items, join_through: MyApp.TodoListItem
    timestamps()
  end
end

defmodule MyApp.TodoListItem do
  use Ecto.Schema

  schema "todo_list_items" do
    belongs_to :todo_list, MyApp.TodoList
    belongs_to :todo_item, MyApp.TodoItem
    timestamps()
  end  
end

defmodule MyApp.TodoItem do
  use Ecto.Schema

  schema "todo_items" do
    field :description
    timestamps()
  end
end
```

Chú ý rằng, `MyApp.TodoList` không cần phải định nghĩa `has_many` trỏ tới `MyApp.TodoListItem` schema nữa, thay vào đó, chúng ta chỉ cần liên kết tới `:todo_items` bằng `many_to_many`.

Khác với `has_many :through`, `many_to_many` là có **writable**. Điều đó có nghĩa là chúng ta có thể gửi dữ liệu từ form chính xác như những gì chúng ta đã làm ở đầu chương:

```elixir
%{
  "todo_list" => %{
    "title" => "shipping list",
    "todo_items" => %{
      0 => %{"description" => "bread"},
      1 => %{"description" => "egges"},
    }
  }
}
```

Và chúng ta cũng không cần phải định nghĩa các hàm changeset trong schema trung gian nữa:

```elixir
# In MyApp.TodoList
def changeset(struct, params \\ %{}) do
  struct
  |> Ecto.Changeset.cast(params, [:title])
  |> Ecto.Changeset.cast_assoc(:todo_items, required: true)
end

# And then in MyApp.TodoItem
def changeset(struct, params \\ %{}) do
  struct
  |> Ecto.Changeset.cast(params, [:description])
end
```

Nói cách khác, chúng ta có thể sử dụng chính xác cùng đoạn code mà chúng ta có trong trường hợp mà "todo list" `has_many` "todo items". Vậy là thậm chí khi các điều kiện bên ngoài yêu cầu chúng ta phải sử dụng thêm một bảng nối, `many_to_many` vẫn có thể tự động quản lý chúng. Tất cả những gì bạn biết về association đều hoạt động với `many_to_many` association, bao gồm cả những cải tiến mà cũng ta đã thảo luận ở các chương trước.

Cuối cùng, mặc dù chúng ta đã xác định một schema trung gian bằng lựa chọn `:join_through` trong `many_to_many`, `many_to_many` vẫn có thể làm việc mà không cần schema trung gian thay vào đó là một tên của table:

```elixir
defmodule MyApp.TodoList do
  use Ecto.Schema

  schema "todo_lists" do
    field :title
    many_to_many :todo_items, join_through: "todo_list_items"
    timestamps()
  end
end
```

Trong trường hợp này, bạn có thể loại bỏ hoàn toàn schema `MyApp.TodoListItem` khỏi ứng dụng của bạn, và đoạn code trên vẫn hoạt động. Điêm khác biệt duy nhất đó là khi sử dụng table, tất cả những giá trị được sinh ra tự động bởi Ecto, ví dụ như `timestamps` sẽ không được tạo ra nữa (vì chúng ta đâu có dùng schema). Để giải quyết vấn đề này, bạn có thể đơn giản là loại bỏ những trường này ra khỏi migration của bạn, hoặc là khởi tạo cho chúng những giá trị mặc định ở mức Database

## Tổng kết

Trong chương này, chúng ta đã sử dụng `many_to_many` để cải thiện đáng kể thiết kế với các liên kết đa hình mà trước đây dựa vào `has_many :through`. Mục tiêu của chúng ta là cho phếp các "todo items" có thể liên kết với nhiều loại thực thể khác nhau trong code base, giống như "todo list" và "project". Chúng ta thực hiện nó bằng cách tạo ra các bảng trung gian, và sử dụng `many_to_many` để tự động quản lý nhưng bảng nối này.

Cuối cùng, schema của chúng ta sẽ như sau:

```elixir
defmodule MyApp.TodoList do
  use Ecto.Schema

  schema "todo_lists" do
    field :title
    many_to_many :todo_items, join_through: "todo_list_items"
    timestamps()
  end

  def changeset(struct, params \\ %{}) do
    struct
    |> Ecto.Changeset.cast(params, [:title])
    |> Ecto.Changeset.cast_assoc(:todo_items, required: True)
  end
end

defmodule MyApp.Project do
  use Ecto.Schema

  schema "projects" do
    field :name
    many_to_many :todo_items, join_through: "project_items"
  end

  def changeset(struct, params \\ %{}) do
    struct
    |> Ecto.Changeset.cast(params, [:name])
    |> Ecto.Changeset.cast_assoc(:todo_items, required: True)
  end  
end

defmodule MyApp.TodoItem do
  use Ecto.Schema

  schema "todo_items" do
    field :description
    timestamps()
  end

  def changeset(struct, params \\ %{}) do
    struct
    |> Ecto.Changeset(params, [:description])
  end
end
```

Và Database migration sẽ như sau:

```elixir
create table("todo_lists") do
  add :title
  timestamps()
end

create table("projects") do
  add :name
  timestamps()
end

create table("todo_items") do
  add :description
  timestamps()
end

# Primary key and timestamps are not required if using many_to_many without schema
create table("todo_list_items", primary_key: false) do
  add :todo_item_id, references(:todo_items)
  add :todo_list_id, references(:todo_lists)
  # timestamps()
end

# Primary key and timestamps are not required if using many_to_many without schema
create table("project_items", primary_key: false) do
  add :todo_item_id, references(:todo_items)
  add :project_id, references(:project)
  # timestamps()
end
```

Nhìn chung, code của chúng ta được tổ chức giống như cách `has_many` đã làm, mặc dù ở mức Database, các mối quan hệ được biểu diễn bằng cách bảng nối.

Trong chương này, chúng ta đã thay đổi code để phù hợp với các định dạng của tham số yêu cầu bởi `cast_assoc`, trong chương kế tiếp, chúng ta sẽ bỏ `cast_assoc` và sử dụng `put_assoc` - hàm này sẽ đem tới nhiều sự linh động hơn khi làm việc với các associations.
