# Concurrent tests với SQL Sandbox

Chương cuối của chúng ta sẽ nói về một trong những tính năng quan trọng nhất của Ecto 2.0: concurent SQL sanbox. Với sức mạnh của Elixir có thể tận dụng tất cả các tài nguyên còn rảnh rỗi của máy tính, khả năng chạy các test song song để nói chuyện với Database có thể giúp lập trình viên tăng tốc thời gian chạy bộ test lên 2x, 4x, 8x hoặc hơn nữa dựa vào số lượng nhân của máy.

Bất cứ khi nào bạn khởi động một Ecto Repository trong cây supervision bằng cách gọi `supervisor(MyApp.Repo, [])`, Ecto sẽ khởi tạo một supervisor với một connection pool. Connection pool này sẽ chứa nhiều connection tới Database. Khi bạn muốn thực hiện một hoạt động với Database, Ecto sẽ tự động lấy ra một connection từ trong pool, thực thi hoạt động của bạn, và sau đó đẩy connection trở về pool.

Điều đó nghĩa là, khi viết test sử dụng connection pool mặc định của Ecto (và không phải là SQL sandbox), mỗi lần bạn chạy một câu query, bạn có khả năng sẽ lấy ra một connection khác từ trong pool. Điều này không tốt cho test bởi vì chúng ta muốn tất cả các hoạt động trong một test có thể sử dụng chung một connection.

Không chỉ vậy, chúng ta cũng muốn phân tách dữ liệu giữa các test với nhau. Nếu tôi thêm một bản ghi vào bảng "users" trong test A, thì test B không nên thấy dữ liệu này trong khi thực hiện query vào cùng bảng "users" đó.

Ecto 1.0 giải quyết bài toán đầu tiên bằng cách đơn giản bắt các test chỉ có thể có một connection trong pool. Với bài toán thứ hai về chuyện phân tách dữ liệu, có thể được giải quyết bằng cách không cho phép các test chạy đồng thời với nhau!

## Explicit checkouts

Ecto 2.0 giải quyết các bài toán trên theo một cách khác. Ý tưởng chính đó là chúng ta cho phép một pool có thể có nhiều connection, nhưng thay vì việc các connection được lấy ra một cách thụ động (implicit) mỗi khi chúng ta chạy query, thì giờ đây các connection sẽ phải được lấy ra một chủ động (explicit) bởi các test. Cách làm này đảm bảo rằng mỗi khi một connection được sử dụng trong một test, nó sẽ luôn luôn là cùng một connection.

Một khi connection được lấy ra một cách chủ động, test sẽ là chủ (own) của connection đó cho tới khi test kết thúc.

Hãy bắt đầu bằng cách cài đặt Database của chúng ta dùng `Ecto.Adapters.SQL.Sandbox` pool. Bạn có thể set những lựa chọn trong `config/config.exs` (hoặc tốt hơn là `config/test.exs`):

```elixir
config :my_app, Repo,
  pool: Ecto.Adapters.SQL.Sandbox
```

Mặc định, sandbox pool sẽ được bật với chế độ `:automatic`, đây chính xác là chế Ecto làm việc mà không có sandbox pool. Nói cách khác, mặc định SQL sandbox sẽ bị tắt (disable). Điều này cho phép chúng ta có thể cài đặt Database, ví dụ để chạy migration hoặc cài đặt `test/test_helper.exs` như bạn vẫn làm thông thường (hay nói cách khác, việc upgrade lên Ecto 2.0 sẽ không làm thay đổi code của bạn).

Trước khi test của chúng ta bắt dầu, chúng ta cần phải chuyển sang chế độ `:manual`, lúc đó mỗi connection sẽ được lấy ra chủ động. Chúng ta làm việc đó bằng cách gọi tới hàm `mode/2`, thường là ở cuối file `test/test_helper.exs` như sau:

```elixir
# At the end of your test_helper.exs
# Set the pool mode to manual for explicit checkouts
Ecto.Adapters.SQL.Sandbox.mode(MyApp.Repo, :manual)
```

Nếu bạn chỉ đơn giản thêm một dòng code ở trên và bạn không thay đổi các test của bạn để lấy cá các connection chủ động từ pool, tất cả các test của bạn sẽ thất bại. Để giải quyết chuyện này, bạn có thể lấy ra connection chủ động cho từng test, và để tránh việc lặp lại code, hãy cùng định nghĩa một `ExUnit.CaseTemplate` để làm việc đó tự động trong hàm `setup` như sau:

```elixir
defmodule MyApp.RepoCase do
  use ExUnit.CaseTemplate

  setup do
    # Explicitly get a connection before each test
    :ok = Ecto.Adapters.SQL.Sandbox.checkout(MyApp.Repo)
  end
end
```

Giờ đây trong các test, thay vì gọi `use ExUnit.Case`, bạn có thể viết `use MyApp.RepoCase, async: true`. Bằng việc làm theo các bước ở trên, chúng ta có thể có nhiều test chạy đồng thời, mỗi test sẽ xác định một Database transaction.

Tuy nhiên, nếu bạn thắc mắc, làm thế nào Ecto có thể đảm bảo các dữ liệu sinh ra trong một test không ảnh hưởng tới các test khác?

## Transactions

Điểm thứ hai bên cạnh việc lấy ra chủ động của SQL Sandbox đó là ý tưởng chạy mỗi một connection được lấy ra chủ động trong một transaction. Mỗi khi bạn chạy `Ecto.Adapters.SQL.Sandbox.checkout(MyApp.Repo)` trong một test, nó không chỉ lấy ra một connection, mà còn đảm bảo rằng connection đó mở ra một transaction trong Database. Theo cách này, bất cứ việc insert, update hoặc delete nào bạn thực hiện trong test cũng sẽ chỉ ảnh hưởng tới test đó thôi.

Thêm vào đó, cuối mỗi test, bạn có thể tự động rollback lại transaction, từ đó dễ dàng loại bỏ những thay đổi trong Database mà bạn đã thực hiện trong test. Điều này đảm bảo một test sẽ không ảnh hưởng tới các test chạy đồng thời cùng nó, hoặc là các test chạy sau nó.

Trong khi cách tiếp cận sử dụng nhiều connections với transaction làm việc trong rất nhiều trường hợp, nó cũng có một số giới hạn liên quan tới cách mà các Database engine quản lý transaction và thực hiện việc kiểm soát đồng thời (concurrent control). Ví dụ trong khi cả PostgreSQL và MySQL hỗ trợ SQL Sandbox, chỉ có PostgreSQL hỗ trợ các test đồng thời khi chạy với SQL Sandbox. Do đó không nên sử dụng `async: true` với MySQL nếu không bạn sẽ bị deadlocks.

Cũng có trường hợp khi bị rơi vào deadlock khi chạy test với PostgreSQL vì các tài nguyên chia sẻ như Database indexes. Những trường hợp này đã được mô tả chi tiết trong [tài liệu của `Ecto.Adapters.SQL.Sandbox`](https://hexdocs.pm/ecto/Ecto.Adapters.SQL.Sandbox.html), trong mục FAQ.

## Ownership

Bất cứ khi nảo một test chủ động lấy ra connection từ SQL Sandbox pool, chúng ta có thể nói test đó "sở hữu" connection đó. Và nhớ rằng, nếu như một test hoặc bất cứ process nào, không lấy ra connection từ pool một cách chủ động, test đó sẽ bị lỗi với nội dung nói là nó không có connection tới Database.

Hãy cùng xem một ví dụ:

```elixir
use MyApp.RepoCase, async: true

test "create two posts, one sync, another async" do
  task = Task.async(fn ->
    Repo.insert!(%Post{title: "async"})
  end)
  assert %Post{} = Repo.insert!(%Post{title: "sync"})
  assert %Post{} = Task.await(task)
end
```

Test ở trên có thể bị lỗi như sau:

```
 ** (RuntimeError) cannot find ownership process for #PID<0.35.0>
```

Một khi chúng ta tạo ra một `Task`, sẽ không có connection nào được gắn cho task process đó, do đó nó sẽ bị thất bại khi chạy test.

Trong khi một lớn thời gian chúng ta muốn các process khác nhau sẽ sở hữu các database connection khác nhau, đôi khi một test có thể muốn tương tác với nhiều process, tất cả cùng sử dụng chung một connection để chúng có thể cùng thuộc về một transaction.

Sandbox module đưa ra 2 giải pháp để giải quyết chuyện này, thông qua "allowances" hoặc chạy trong chế độ chia sẻ

## Allowances

Nếu một process chủ động lấy sở hữu một connection, process này có thể cũng cho phép các process khác được sử dụng connection đó. Hệ quả là nó cho phép nhiều process có thể cùng làm việc với một connection cùng lúc. Hãy cùng xem xét ví dụ sau:

```elixir
test "create two post, one sync, one async" do
  parent = self()
  task = Task.async(fn ->
    Ecto.Adapters.SQL.Sandbox.allow(Repo, parent, self())
    Repo.insert!(%Post{title: "async"})
  end)
  assert %Post{} = Repo.insert!(%Post{title: "sync"})
  assert %Post{} = Task.await(task)
end
```

Bằng cách gọi `allow/3`, chúng ta gán connection của process cha (chính là connection của test) một cách tường minh với task này.

Bởi vì allowances sử dụng cơ chế tường mình, lợi điểm của nó là bạn vãn có thể chạy các test trong chế độ async. Bất lợi là bạn sẽ phải điểu khiển và cho phép mọi process còn một cách tường minh, điều này không phải luôn luôn là khả thi. Trong những trường hợp như vậy, bạn có thể sẽ cần dùng tới chế độ chia sẻ

## Shared mode

Chế độ chia sẻ (shared mode) cho phép một process có thể chia sẻ connection của nó với bất cứ process nào một cách tự động, mà không cần dựa vào cơ chế cho phép tường minh.

Hãy cùng thay đổi ví dụ trên bằng chế độ chia sẻ:

```elixir
test "create two post, one sync, one async" do
  Ecto.Adapters.SQL.Sandbox.mode(Repo, {:shared, self()})
  task = Task.async(fn ->
    Repo.insert!(%Post{title: "async"})
  end)
  assert %Post{} = Repo.insert!(%Post{title: "sync"})
  assert %Post{} = Task.await(task)
end
```

Bằng việc gọi `mode({:shared, self()})`, bất cứ process nào cần phải nói chuyện với Database giờ sẽ sử dụng chung connection với test process.

Lợi ích của chế độ chia sẻ đó là chỉ cần gọi một hàm, bạn sẽ đảm bảo tất cả các process và hoạt động tiếp theo sẽ sử dụng chung một connection, mà không cần phải cho phép nó một cách tường mình. Nhược điểm đó là các test không thể nào chạy cùng nhau trong chế độ chia sẻ.

## Tổng kết

Trong chương này, chúng ta đã học về sức mạnh của SQL sandbox và cách mà nó nâng tầm transaction, và một cơ chế sở hữu với việc lấy ra một cách tường minh để chạy các test đồng thời khi chúng cần phải tương tác với Database.

Chúng ta cũng đã thảo luận hai cơ chế để chia sẻ sở hữu:

+ Sử dụng allowances - yêu cầu phải cho phép tường minh bằng hàm `allow/3`. Test có thể chạy đồng thời

+ Sử dụng chế độ chia sẻ - không yêu cầu cho phép tường minh. Test không thể chạy đồng thời

Xuyên suốt cuốn sách này, chúng ta đã học về Ecto như một tập các công cụ để làm việc trong lĩnh vực của bạn, các chương cuối cùng chỉ ra rằng Ecto cung các các công cụ tốt hơn để tương tác với Database, như Ecto.Multi dựa vào tính chất hàm của Elixir cũng như SQL Sandbox dựa vào sức mạnh xử lý song song đằng sau máy ảo Erlang.

Chúng tôi hy vọng bạn học được nhiều điều từ chuyến hành trình này, và bạn đã sẵn sàng để viết các ứng dụng nhanh, gọn gàng và dễ bảo trì.
