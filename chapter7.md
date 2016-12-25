# Cải thiện associations và factories

Ecto 2.0 đã cải thiện rất nhiều cách association hoạt động. Để hiểu vì sao Ecto cần làm chuyện này cũng như thư viện đã giải quyết vấn đề này như nào, chúng ta hãy cùng nói về mục tiêu thiết kế của Ecto.

Ecto được bắt đầu như một dự án "Summer of Code" của Eric Meadows-Jonsson (ngày nay là một thành viên của Elixir team, và là creator của Hex), dưới sự hướng dẫn của José Valim, creator của Elixir vào khoảng năm 2013 khi Elixr vẫn còn đang ở phiên bản 0.9!

Giống như rất nhiều dự án lúc đó, một trong những mục tiêu của Ecto là để đánh giá bản thân Elixir như một ngôn ngữ lập trình. Một trong những câu hỏi mà Ecto muốn trả lời đó là: "liệu rằng Elixir có thể sử dụng để tạo ra được một thư viện để tương tác với Database nhanh(performant) và bảo mật (secure) hay không?". Bằng cách trở thành một nền tảng ổn định, nhanh và bảo mật, Ecto có thể thêm vào các cú pháp hỗ trợ (syntax sugar), thuận tiện và năng động sau đó - trong khi hướng ngược lại có thể cực kỳ khó với kinh nghiệm của team Ecto.

*(lời người dịch) chỗ này dịch hơi gượng, bạn nào có thể giúp dịch lại cho tốt được không*

Ecto 1.0 trở thành một nền tảng nhanh và bảo mật theo hướng này, và kết quả là nó trở nên kết dính khá chắt theo nhiều mặt.

Ecto 2.0 cải thiện những sai sót tạo ra bởi Ecto 1.0 và sau đó xay dựng trên những nền tảng của Ecto 1.0 bằng cách thêm vào sự linh động mà cộng đồng đợi chờ từ lâu. Chúng ta đã khám phá rất nhiều tính linh động này trong các chương trước của cuốn sách như: query schemaless, changeset schemaless, query động, ... Trong 3 chương kế tiép, chúng ta sẽ khám phá thêm nhưng cải tiến đã được làm cho schema và associations.

Trong chương này, chúng ta sẽ học cách mà Ecto có thể insert những cấu trúc dữ liệu phức tạp mà không cần phải sử dụng changesets, cũng như cách sử dụng tính năng này để quản lý các dữ liệu phức tạp, ví dụ như khi test ứng dụng của bạn mà không cần dựa vào các dự án bên ngoài.

## Less changesets

Ở thời điểm Ecto 2.0 mang tới rất nhiều tính năng cho changesets, Ecto 2.0 cũng làm cho changeset không cần sử dụng nhiều trong các API khác của Ecto. Ví dụ, với Ecto 1.0, `Ecto.Repo.insert/2` đòi hỏi phải truyền vào changeset. Điều này nghĩa là, để có thể insert một bản ghi vào Database, giống như một post, chúng ta phải bao lấy nó bằng một changeset như sau:

```elixir
%Post{title: "hello world"}
|> Ecto.Changeset.change
|> Ecto.insert!
```

Điều này cũng đúng với phần lớn các API của Ecto 1.0. Nếu bạn muốn tạo ra một post với một vài comments, bạn phải bao lấy mỗi comment trong một changeset, và sau đó đẩy chúng vào post changeset:

```elixir
comment1 = %Comment{body: "excellent article"} |> Ecto.Changeset.change
comment2 = %Comment{body: "I learned something new"} |> Ecto.Changeset.change

%Post{title: "hello world"}
|> Ecto.Changeset.put_assoc(:comments: [comment1, comment2])
|> Repo.insert!
```

Hơn thế nữa, khi xử lý associations, Ecto 1.0 bắt buộc bạn phải luôn luôn viết các parent changeset trước, rồi sau đó mới viết các changeset con. Với ví dụ trên, chúng ta có thể insert một post (changset cha), với nhiều comments (changset con), nhưng ví dụ sau thì không thể:

```elixir
post = %Post{title: "hello world"} |> Ecto.Changeset.changeset

%Comment{body: "excellent article"}
|> Ecto.Changeset.put_assoc(:post, [post])
|> Repo.insert!
```

Ecto 2.0 phá bỏ những rào cản này. Bạn giờ có thể truyền struct vào repository và changeset, và Ecto sẽ tự động tạo ra các changeset cho bạn. Với Ecto 2.0, một post với nhiều comments có thể được insert trực tiếp như sau:

```elixir
Repo.insert! %Post{
  title: "hello world",
  comments: [
    %Comment{body: "excellent article"},
    %Comment{body: "I learned something new"}
  ]
}
```

Bạn cũng có thể insert hoặc update các association theo bất cứ hướng nào, từ cha tới con, hoặc là ngược lại:

```elixir
Repo.insert! %Comment {
    body: "excellent article",
    post: %Post{title: "hello world"}
}
```

Tính năng này không chỉ hữu dụng khi viết các ứng dụng của bạn, mà còn khi test, như chúng ta sẽ thấy sau đây

## Test factories

Rất nhiều dự án phụ thuộc vào các thư viện khác để xây dựng dữ liệu test. Một trong số các thư viện đó được gọi là Factories vì chúng cung cấp các hàm tiện dụng để xây dựng một nhóm các dữ liệu khác nhau. Tuy nhiên, với việc Ecto 2.0 có thể quản lý các cây dữ liệu phức tạp, chúng ta có thể cài đặt những chức năng này mà không cần phụ thuộc vào các thư viên bên thứ ba.

Để bắt đầu, chúng ta tạo ra file `test/support/factory.ex` với nội dung sau:

```elixir
defmodule MyApp.Factory do
  alias MyApp.Repo

  # Factories

  def build(:post) do
    %MyApp.Post{title: "hello world"}
  end

  def build(:comment) do
    %MyApp.Comment{body: "good post"}
  end

  def build(:post_with_comments) do
    %MyApp.Post{
      title: "hello with comments",
      comments: [
        build(:comment, body: "first"),
        build(:comment, body: "second"),
      ]
    }
  end

  def build(:user) do
    %MyApp.User{
      email: "hello#{System.unique_integer}",
      username: "hello#{System.unique_integer}"
    }
  end

  # Convenience API

  def build(factory_name, attributes) do
    factory_name |> build |> struct(attributes)
  end

  def insert!(factory_name, attributes \\ []) do
    Repo.insert! build(factory_name, attributes)
  end
end
```

Module factory của chúng ta định nghĩa bốn "factories" như bốn mệnh đề của hàm `build`: `:post`, `:comment`, `:post_with_comments` và `:user`. Mỗi một mệnh đề định nghĩa một struct với các trường cần thiết cho Database. Trong một số trường hợp, struct được sinh ra cũng cần những trường unique, giống như email và username của user. Chúng ta tạo ra chúng bằng cách gọi hàm của Elixir `System.unique_integer` - bạn cũng có thể gọi `System.unique_integer([:positive])` nếu bạn muốn tạo ra các số dương.

Cuối cùng, chúng ta định nghĩa 2 hàm `build/2` và `insert!/2` để hỗ trợ việc xây dựng các struct với các thuộc tính cụ thể, hoặc để insert dữ liệu trức tiếp vào repository.

Đó là tất cả những việc cần thiết để xây dựng factory. Giờ chúng ta đã sẵn sàng để sử dụng những factory này trong các test của chúng ta. Đầu tiên, hãy mở file "mix.exs" và đảm bảo rằng "test/support/factory.ex" đã được biên dịch:

```elixir
def project do
  [...
   elixirc_paths: elixirc_paths(Mix.env),
  ...]
end

defp elixirc_paths(:test), do: ["lib", "test/support"]
defp elixirc_paths(_), do: ["lib"]
```

Giờ trong bất cứ test nào mà chúng ta muốn sinh dữ liệu, chúng ta có thể import module `MyApp.Factory` và sử dụng nó như sau:

```elixir
import MyApp.Factory

build(:post)
#=> %MyApp.Post(id: nil, title: "hello world", ...)

build(:post, title: "custom title")
#=> %MyApp.Post(id: nil, title: "custom title", ...)

insert!(:post, title: "custom title")
#=> %MyApp.Post(id: ..., title: "custom title")
```

Bằng cách xây dựng các chức năng mà chúng ta cần dựa trên khả năng của Ecto, chúng ta có thể mở rộng và cải thiện các Factory theo bất cứ hướng nào mà chúng ta muốn, thay vì bị phụ thuộc vào giới hạn của các thư viên bên thứ ba.
