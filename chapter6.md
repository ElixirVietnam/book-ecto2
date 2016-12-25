# Aggregates và subqueries

Tính năng cuối cùng chúng ta thảo luận về Ecto query là aggregates và subqueries.

## Aggregates

Ecto 2.0 cung cấp một hàm trong các repositories để tính toán các aggregates.

Ví dụ, để tìm số lượng visit trung bình của tất cả các posts:

```elixir
MyApp.Repo.Aggregate(MyApp.Post, :avg, :visits)
#=> #Decimal<1743>
```

Câu query trên sẽ được chuyển thành câu query dưới đây:

```elixir
MyApp.Repo.one(from p in MyApp.Post, select: avg(p.visits))
```

Hàm `aggregate/3` hỗ trợ bất cứ hàm aggregate nào được liệt kê trong [Ecto Query API](https://hexdocs.pm/ecto/Ecto.Query.API.html)

Lúc đầu, có vẻ như việc cài đặt hàm `aggregate/3` là khá đơn giản. Bạn thậm chí sẽ bắt đầu băn khoăn vì sao hàm này không được thêm vào Ecto ngay từ đầu. Tuy nhiên, vấn đề bắt đầu trở nên phức tạp hơn với các câu queries phụ thuộc vào `limit`, `offset` và `distinct`.

Tưởng tượng rằng, thay vì muốn tính giá trị trung bình của tất cả các post, bạn muốn tính giá trị trung bình của top 10 post đầu tiên. Bạn có thể thử như sau:

```elixir
MyApp.Repo.one(from p in MyApp.Post,
  order_by: [desc: :visits],
  limit: 10,
  select: avg(p.visits))
#=> Decimal<1743>
```

Câu query như trên trả về cùng kết quả với câu query trước đó. Lựa chọn `limit: 10` không tạo ra bất cứ ảnh hưởng nào bởi vì nó giới hạn kết quả của câu query aggregate, và câu query này chỉ trả về duy nhất một kết quả. Để lấy được giá trị chính xác, chúng ta cần tìm ra top 10 post trước, sau đó mới áp dụng hàm aggregate.

```elixir
query = from MyApp.Post, order_by: [desc: :visits], limit: 10
MyApp.Repo.aggregate(query, :avg, :visits)
#=> Decimal<4682>
```

Khi `limit`, `offset` hoặc `distinct` được xác định trong câu query, `aggregate/3` sẽ tự động biến câu query đầu vào thành một subquery. Câu query được thực hiện bởi `aggregate/3` ở trên tương đương với câu query dưới đây

```elixir
query = from MyApp.Post, order_by: [desc: :visits], limit: 10
MyApp.Repo.one(from q in subquery(query), select: avg(q.visits))
```

Hãy cùng tìm hiểu kỹ hơn về subquery.

## Subquery

Trong phần trước, chúng ta đã học được rằng một vài câu queries rất khó để biểu diễn mà không có sự hỗ trợ của subquery. Đó là một trong rất nhiều ví dụ dẫn tới việc hỗ trợ subquery được thêm vào Ecto.

Subquery trong Ecto được tạo ra bằng cách gọi `Ecto.Query.subquery/1`. Hàm này nhận bất cứ cấu trúc dữ liệu nào có thể chuyển thành query thông qua `Ecto.Queryable` protocol, và trả về một subquery \(subquery cũng đã cài đặt protocol `Ecto.Queryable`\).

Trong Ecto 2.0, một subquery có thể select cả table `p`, hoặc một trường `p.field`. Tất cả các trường được lựa chọn trong một subquery, có thể được truy cập từ câu query cha. Hãy cùng xem lại câu aggregate chúng ta viết ở phần trước:

```elixir
query = from MyApp.Post, order_by: [desc: :visits], limit: 10
MyApp.Repo.one(from q in subquery(query), select: avg(q.visits))
```

Do `query` không xác định mệnh đề `:select`, nên nó sẽ trả về `select: p` với `p` được xác định bởi schema `MyApp.Post`. Từ việc câu query này trả về tất cả các trường trong `MyApp.Post`, khi chúng ta chuyển nó thành một subquery, tất cả các trường từ `MyApp.Post` sẽ sẵn sàng để truy cập được từ câu query cha, giống như `q.visits`. Thực tế, Ecto sẽ giữ lại các thuộc tính schema giữa tất cả các câu queries. Ví dụ, nếu bạn viết `q.field_doest_not_exist`, câu Ecto query của bạn sẽ không được biên dịch.

Ecto 2.1 cải thiện các câu subqueries bằng cách cho phép một Elixir map có thể được trả về từ một subquery, điều này làm cho các trường map có thể được truy cập trực tiếp từ câu query cha.

Hãy cùng xem ví dụ cuối cùng. Tưởng tượng bạn phải quản lý một thư viện, và có một table sẽ ghi lại mọi thời điểm mà thư viện cho mượn một cuốn sách. Bảng "lendings" sử dụng các index tự tăng \(auto-increment indexes\), và có thể được định nghĩa bằng schema như sau:

```elixir
defmodule Library.Lending do
  use Ecto.schema

  schema "lendings" do
    belongs_to: :book, MyApp.Book              # define book_id
    belongs_to: :visitor, MyApp.Visitor        # define visitor_id
  end
end
```

Giả sử rằng chúng ta muốn lấy tên của tất cả các mọi cuốn sách cùng với tên của người mượn sách cuối cùng. Để làm việc này, chúng ta sẽ cần phải tìm `lending id` cuối cùng của mọi cuốn sách, sau đó join với bảng "books" và bảng "visitors". Với subquery, chúng ta có thể dễ dàng làm như sau:

```elixir
last_lendings =
  from l in MyApp.Lending,
    group_by: l.book_id,
    select: %{book_id: l.book_id, last_lending_id: l.id}

from l in MyApp.Lending,
  join: last in subquery(last_lendings),
    on: last.last_lending_id == l.id,
  join: b in assoc(l, :book),
  join: v in assoc(l, :visitor),
  select: {b.name, v.name}
```

Subquery là một sự cải thiện quan trong của Ecto làm cho việc biểu diễn một loạt các câu query tưởng như không thể trước đây, thành một việc khả thi. Dựa vào đó, chúng ta có thể thêm các tính năng như aggregate - từ đó cung cấp chức năng hữu dụng giúp bảo vệ lập trình viên khỏi những trường hợp ngách.

