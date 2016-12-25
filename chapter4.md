# Query động

Khi xây dựng các query trong Ecto, chúng ta có thể sử dụng keyword syntax:

```elixir
import Ecto.Query

from p in Post,
  where: p.author == "José" and p.category == "Elixir",
  where: p.published_at > ^minimum_date,
  order_by: [desc: p.published_at]
```

hoặc cú pháp dựa vào pipe như sau:

```elixir
import Ecto.Query

Post
|> where([p], p.author == "José" and p.category == "Elixir")
|> where([p], p.published_at > ^minimum_date)
|> order_by([p], desc: p.published_at)
```

Trong khi nhiều lập trình viên thích cú pháp dựa vào toán tử pipe, việc phải lặp đi lặp lại binding `p` khiến cho nó khá là rườm rà so với cú pháp dùng keyword syntax. Thêm vào đó, cách tiếp cận sinh query tại thời điểm biên dịch của Ecto khiến cho việc xây dựng các query động trở nên khó khăn. Tưởng tượng một ứng dụng web cung cấp chức năng tìm kiếm các top post. Người dùng có thể xác định nhiều tiêu chí để tìm kiếm: category của post, tên tác giả, khoảng thời gian post được đăng, ...

Với Ecto 1.0, cách duy nhất để viết chức năng này là sử dụng hàm `Enum.reduce/3`

```elixir
def filter(params) do
  Enum.reduce(params, Post, &filter/2)
end

defp filter({"author", author}, query) do
  where(query, [p], p.author == ^author)
end
defp filter({"category", category}, query) do
  where(query, [p], p.category == ^category)
end
defp filter({"published_at", minimum_date}, query) do
  where(query, [p], p.published_at == ^minimum_date)
end
defp filter({"order_by", "published_at_asc"}, query) do
  order_by(query, [p], desc: p.published_at)
end
defp filter({"order_by", "published_at_asc"}, query) do
  order_by(query, [p], asc: p.published_at)
end
defp filter(_ignore_unknow, query) do
  query
end
```

Trong khi đoạn code ở trên làm việc tốt, nó tạo nên sự kết dính giữa quá trình xử lý các tham số với quá trình sinh ra các query. Nó là một cách cài đặt khá rườm rà trong khi cũng khá là khó để test bởi vì kết quả của việc filter và xử lý các parameters được lưu lại trực tiếp vào trong cấu trúc của các query.

Ecto 2.0 đưa ra một cách tiếp cận tôt hơn, cho phép chúng ta gộp các xử lý parameters lại thành một cấu trúc dữ liệu, rồi sau đó mới truyền cấu trúc dữ liệu này để xây dựng query.

## Tập trung vào cấu trúc dữ liệu

Ecto 2.0 cung cấp một API đơn giản hơn cho cả 2 loại query: keyword query vào pipe query.

```elixir
from p in Post,
  where: [auther: "José", category: "Elixir"],
  where: p.published_at > ^minimum_date,
  order_by: [desc: :published_at]
```

và

```elixir
Post
|> where(author: "José", category: "Elixir")
|> where([p], p.published_at > ^minimum_date)
|> order_by(desc: :published_at)
```

Chú ý cách chúng ta loại bỏ `p` selector trong phần lớn các biểu thức. Với Ecto 2.0, tất cả các hàm khởi tạo, từ `select` và `order_by` tới `where` và `group_by`, đều chấp nhận các input là các cấu trúc dữ liệu. Các cấu trúc dữ liệu này có thể được xác định tại thời điểm biên dịch \(compile-time\) như ở trên hoặc là xác định động trong khi chạy \(runtime\) như dưới đây

```elixir
where = [author: "José", category: "Elixir"]
order_by = [desc: :published_at]

Post
|> where(^where)
|> where([p], p.published_at > ^minimum_date)
|> order_by(^order_by)
```

Lợi ích của việc nội suy \(interpolating\) các cấu trúc dữ liệu như trên đó là chúng ta có thể phân tách quá trình xử lý các parameters khỏi quá trình sinh query. Tuy nhiên không phải tất cả các biểu thức có thể chuyển thành dạng cấu trúc dữ liệu. Cụ thể với cậu lệnh `where` có thể chuyênn hoá dạng key-value thành so sánh bằng `key == value`, nhưng các só sánh khác như `p.published_at > ^mininum_date` vẫn chưa được hỗ trợ.

Phiên bản tiếp theo Ecto 2.1 sẽ giải quyết những vấn đề này.

## Macro động

Cho những trường hợp chúng ta không thể dựa vào các cấu trúc dữ liệu, nhưng vẫn mong muốn xây dựng các câu query động, Ecto 2.1 sẽ bao gồm macro `Ecto.Query.dynamic/2`.

Để hiểu `dynamic` macro làm việc như nào, chúng ta hãy cùng nhau viết lại hàm `filter/1` ở đầu chương bằng cách sử dụng cả các cấu trúc dữ liệu và `dynamic` macro. Chú ý rằng các ví dụ dưới đây yêu cầu phiên bản Ecto 2.1:

```elixir
def filter(params) do
  Post
  |> order_by(^filter_order_by(params["order_by"]))
  |> where(^filter_where(params))
  |> where(^filter_published_at(params["published_at"]))
end

def filter_order_by("published_at_desc"), do: [desc: published_at]
def filter_order_by("published_at"),      do: [asc: published_at]
def filter_order_by(_),                   do: []

def filter_where(params) do
  for key <- [:author, :category],
      value = params[Atom.to_string(params)],
      do: {key, value}
end

def filter_published_at(date) when is_binary(data),
  do: dynamic([p], p.published_at > ^date)
def filter_published_at(date) when is_binary(data),
  do: true
```

`dynamic` macro cho phép chúng ta xây dựng những biểu thức động mà sau đó có thể thêm vào \(interpolated\) vào trong câu query, các biểu thức `dynamic` có thể thêm vào vào chính các biểu thức `dynamic` khác, điều này cho phép các lập trình viên có thể xây dựng những biểu thức động phức tạp.

Bởi vì chúng ta có thể chia nhỏ bài toán thành các hàm nho hơn nhận vào các cấu trúc dữ liệu thông thường, chúng ta có thể sử dụng các công cụ có sẵn trong Elixir để làm việc với các dữ liệu này. Để xử lý tham số `order_by`, có thể cách tốt nhất là sử dụng các pattern matching trên những `order_by` parameter. Để xây dựng các mệnh đề `where`, chúng ta có thể duyệt qua một list các khoá đã biết, và chuyển hoá chúng thành định dạng phù hợp với Ecto. Cho những điều kiện phức tạp, chúng ta dùng `dynamic` macro.

Việc testing sẽ trên nên đơn giản hơn nhiều vì chúng ta có thể test từng hàm riêng biết, thâm chí cả khi sử dụng các query động:

```elixir
test "filter published at based on the given date" do
  assert inspect(filter_published_at("2010-04-17")) ==
         "dynamic([p], p.published_at > ^\"2010-04-17\")"
  assert inspect(filter_published_at(nil)) ==
         "true"
end
```

Trong khi cuối cùng, một vài lập trình viên sẽ vẩn cảm thấy thoải mái hơn với cách sử dụng `Enum.reduce/3`, tuy nhiên Ecto 2.0 và các phiên bản sau cho chúng ta được quyền lựa chọn cách tiếp cận nào là phù hợp nhất.

