# Tạo transaction với Ecto.Multi

Ecto dựa vào các Database transaction khi muốn nhiều hoạt động được thực hiện cùng với nhau. Transaction có thể được thực thi thông qua hàm `Repo.transaction`:

```elixir
Repo.transaction(fn ->
  mary |> Ecto.Changeset.change(balance: mary.balance - 10) |> Repo.update!
  john |> Ecto.Changeset.change(balance: mary.balance + 10) |> Repo.update!
end)
```

Khi chúng ta mong muốn cả hai hoạt động được thành công cùng nhau như ví dụ ở trên, transaction là lựa chọn hiển nhiên. Tuy nhiên, transaction sẽ trở nên khá phức tạp nếu các hoạt động này có thể thất bại:

```elixir
Repo.transaction(fn ->
  case mary |> Ecto.Changeset.change(balance: mary.balance - 10) |> Repo.update! ->
    {:ok, mary} ->
      case john |> Ecto.Changeset.change(balance: mary.balance + 10) |> Repo.update! ->
        {:ok, john} ->
          {mary, john}
        {:error, changeset} ->
          Repo.rollback({:john, changeset})
      end
    {:error, changeset} ->
      Repo.rollback({:mary, changeset})
  end
end)
```

Nói cách khác, transaction trong Ecto có thể bị lồng nhau. Ví dụ, tưởng tượng các transaction ở trên bị di chuyển vào một hàm khác, `transfer_money(mary, john, 10)`, và bên cạnh việc chuyển tiền, chúng ta cũng muốn lưu lại giao dịch này:

```elixir
Repo.transaction(fn ->
  case transfer_money(many, john, 10) ->
    {:ok, {mary, john}} ->
      Repo.insert!(%Transfer{from: mary.id,, to: john.id, amount: 10})
    {:error, {who, changeset}} ->
      Repo.rollback(who, changeset)
  end
end)
```

Đoạn code ở trên chạy trong một transaction, và sau đó gọi hàm `transfer_money/3` cũng chạy trong một transaction. Đoạn code này có thể làm việc bởi vì Ecto chuyển hoá bất cứ transaction lồng (nested transaction) thành một savepoint một cách tự động. Trong trường hợp transaction ở trong bị thất bại, nó sẽ rollback về một điểm savepoint cụ thể.

Trong khi transaction lồng có thể giúp code dễ đọc hơn bằng cách chia nhỏ các transaction lớn thành nhiều transaction con nhỏ hơn, vẫn có một vài vấn đề về tính rườm rà khi phải xử lý transaction dựa vào việc transaction đó có thành công hay không. Hơn thế nữa, việc lồng nhau khá là giới hạn, vì tất cả các hoạt động sẽ phải thực hiện trong một transaction chủ ở bên ngoài.

Một cách tiếp cận tốt (declarative) hơn đó là **định nghĩa tất cả các hoạt động mà chúng ta muốn thực hiện trong một transaction tách biệt với việc thực thi transaction đó**. Với cách này, chúng ta có thể tạo nên hoạt động của transaction mà không cần phải quan tâm tới ngữ cảnh thực thi hay là kịch bản thành công/thất bại của từng hoạt động riêng lẻ. Đó chính xác là cách mà `Ecto.Multi` sẽ cho phép chúng ta xây dựng.

## Tạo transaction với các cấu trúc dữ liệu

Hãy cùng viết lại đoạn code ở trên với `Ecto.Multi`. Đoạn code đầu tiên để chuyển tiền giữa mary và john có thể viết như sau:

```elixir
Ecto.Multi.new
|> Ecto.Multi.update(:mary, Ecto.Changeset.change(mary, balance: mary.balance - 10))
|> Ecto.Multi.update(:john, Ecto.Changeset.change(john, balance: john.balance + 10))
```

`Ecto.Multi` là một cấu trúc dữ liệu cho phép chúng ta định nghĩa các hoạt động nào phải được thực hiện cùng nhau mà không cần quan tâm về việc chúng sẽ được thực hiện ở đâu, và thực hiện như nào. `Ecto.Multi` có phần lớn các API giống với `Ecto.Repo`, với sự khác biệt là mỗi một hoạt động phải có một tên cụ thể. Trong ví dụ trên, chúng ta đã định nghĩa 2 hoạt động cập nhật, với các tên `:mary`, `:john`. Như chúng ta sẽ thấy, các tên này rất quan trọng khi xử lý kết quả của những hoạt động này.

Từ việc `Ecto.Multi` chỉ là một cấu trúc dữ liệu, chúng ta có thể truyền nó như là tham số tới các hàm khác, cũng như trả về nó trong một hàm. Giả sử multi được tạo ra ở đoạn code trên được di chuyển vào một hàm `transfer_money(mary, john, 10)`, chúng ta có thể thêm một hoạt động để lưu lại giao dịch như sau:

```elixir
transfer_money(mary, john, 10)
|> Ecto.Multi.insert(:transfer, %Transfer(from: mary.id, to: john.id, amount: 10})
```

Điều này có thể coi là đơn giản hơn so với cách tiếp cận sử dụng transaction lồng mà chúng ta đã thấy trước đây. Một khi tất cả các hoạt động được định nghĩa ở trong một multi, chúng ta có thể gọi tới hàm `Repo.transaction` để thực thi nó như sau:

```elixir
transfer_money(mary, john, 10)
|> Ecto.Multi.insert(:transfer, %Transfer(from: mary.id, to: john.id, amount: 10})
|> Repo.transaction
|> case do
  {:ok, %{mary: mary, john: john, transfer: transfer}} ->
    # handle success case
  {:error, name, value, rolled_back_change} ->
    # handle failure case
end
```

Nếu tất cả các hoạt động trong multi đều thành công, hàm `Repo.transaction` sẽ trả về `{:ok, map}` trong đó `map` là một map có khoá là tên của tất cả các hoạt động, và value của khoá là giá trị được trả về khi hành động với khoá đó thành công. Nếu bất cứ hành động nào thất bại, transaction sẽ phải được roll back và hàm `Repo.transaction` trả về `{:error, name, value, rolled_back_changes}` trong đó `name` là tên của hoạt động thất bại, `value` là giá trị trả về của hoạt động đó, `rolled_back_changeset` là map của các hoạt động thành công khác được thực thi trước hành động thất bại.

Nói cách khác, `Ecto.Multi` tự quản lý hết các luồng điều khiển cho chúng ta, trong khi nó phân tách định nghĩa transaction với cách transaction này được thực thi, điều này cho phép chúng ta có thể tạo ra các hành động một cách rất đơn giản.

## Testing

Một lợi điểm của việc sử dụng `Ecto.Multi` đó là chúng ta có thể duyệt qua tất cả các hoạt động trong multi và sử dụng các dữ liệu này để viết test. Ví dụ, chúng ta có thể test như sau:

```elixir
test "transfers from mary to john" do
  multi = transfer_money(mary, john, 10)
  assert [{:mary, {:update, mary_changeset, _}},
          {:john, {:update, john_changeset, _}}] = Ecto.Multi.to_list(multi)
  assert mary_changeset.changes.balance == mary.balance - 10
  assert john_changeset.changes.balance == john.balance + 10
end
```

## Các giá trị phụ thuộc

Bên cạnh các hoạt động như `insert`, `update` và `delete`. `Ecto.Multi` cũng cung cấp các hàm để quản lý các kịch bản phức tạp hơn. Ví dụ, `prepend` và `append` có thể được dùng để gộp các multi với nhau. Tổng quát hơn, hàm `Ecto.Multi.run/3` có thể sử dụng để định nghĩa bất cứ hoạt động nào phụ thuộc vào kết quả của các hoạt động trước đó trong multi.

Chúng ta hãy cùng xem xét một ví dụ thực tế bằng cách xem lại vấn đề đã được nêu lên từ chương trước. Chúng ta muốn thay đổi một post trong khi truyền vào một list các tags được biểu diễn bằng một chuỗi được phân tách bởi các dấu phảy. Đến cuối chương trước, chúng ta đã xây dựng một giải pháp để có thể thêm vào bất cứ tag nào chưa tồn tại và sau đó lấy ra tất cả các tag bằng hai câu query.

```elixir
defmodule MyApp.Post do
  use Ecto.Schema

  # Schema is the same
  schema "posts" do
    field :title
    field :body
    many_to_many :tags, MyApp.Tag, join_through: "posts_tags"
    timestamps()
  end

  # Changeset is the same
  def changeset(struct, params \\ %{}) do
    struct
    |> Ecto.Changeset.cast(params, [:title, :body])
    |> Ecto.Changeset.put_assoc(:tags, parse_tags(params))
  end

  # Parse tags has slightly changed
  defp parse_tags(params) do
    (params["tags"] || "")
    |> String.split(",")
    |> Enum.map(&String.trim/1)
    |> Enum.reject(& &1 == "")
    |> insert_and_get_all
  end

  defp insert_and_get_all([]) do
    []
  end
  defp insert_and_get_all(names) do
    maps = Enum.map(names, &%{name: &1})
    Repo.insert_all MyApp.Tag, maps, on_conflict: :nothing
    Repo.all from t in MyApp.Tag, where: t.name in ^names
  end
end
```

Mặc dù hàm `insert_and_get_all/1` là idempotent, cho phép chúng ta có thể chạy nó nhiều lần và vẫn có cùng kết quả, tuy nhiên do nó không chạy trong một transaction, nên bất cứ thất bại nào trong khi cố gắng thay đổi post cha, có thể dẫn tới trường hợp các tag được tạo ra mà không có bất cứ post nào được liên kết với chúng.

Hãy cùng sửa lại lỗi ở trên bằng cách sử dụng `Ecto.Multi`. Bắt đầu bằng việc chia logic vào 2 modules `Post` và `Tag`, và giữ chúng khỏi bị side-effects:

```elixir
defmodule MyApp.Post do
  use Ecto.Schema

  schema "posts" do
    add :title
    add :body
    many_to_many :tags, MyApp.Tag, join_through: "posts_tags"
    timestamps()
  end

  def changeset(struct, tags, params) do
    struct
    |> Ecto.Changeset.cast(params, [:title, :body])
    |> Ecto.Changeset.put_assoc(:tags, tags)
  end
end

defmodule MyApp.Tag do
  use Ecto.Schema

  schema "tags" do
    add :name
    timestamps()
  end

  def parse(tags) do
    (tags || "")
    |> String.split(",")
    |> Enum.map(&String.trim/1)
    |> Enum.reject(& &1 == "")
  end
end
```

Giờ đây, bất cứ khi nào chung ta muốn thêm một post với nhiều tags, chúng ta có thể tạo ra một multi bao lấy các hoạt động này:

```elixir
def insert_or_update_post_with_tags(post, params) do
  Ecto.Multi.new
  |> Ecto.Multi.run(:tags, &insert_and_get_all_tags(&1, params))
  |> Ecto.Multi.run(:post, &insert_or_update_post(&1, post, params))
  |> Repo.transaction
end

defp insert_and_get_all_tags(_changes, params) do
  case MyApp.Tag.parse(params["tags"]) do
    [] ->
      []
    tags ->
      maps = Enum.map(names, &%{name: &1})
      Repo.insert_all(MyApp.Tag, maps, on_conflict: :nothing)
      Repo.all(from t in MyApp.Tag, where: t.name in ^names)
  end
end

defp insert_or_update_post(%{tags: tags}, post, params) do
  Repo.insert_or_update MyApp.Post.changeset(post, tags, params)
end
```

Trong ví dụ trên, chúng ta đã sử dụng `Ecto.Multi.run/3` hai lần với hai lý do khác nhau:

1. Trong `Ecto.Multi.run(:tags, ...)`, chúng ta dùng `run/3` vì muốn thực hiện cả 2 hành động `insert_all` và `all`, trong khi multi hỗ trợ API `Ecto.Multi.insert_all/4`, nó không hỗ trợ API `Ecto.Multi.all/3`. Bất cứ khi nào chúng ta muốn thực hiện một hành động mà `Ecto.Multi` chưa hỗ trợ, chúng ta có thể fallback về hàm `run/3`

2. Trong `Ecto.Multi.run(:post, ...)`, chúng ta sử dụng `run/3` vì chúng ta cẩn truy cập vào giá trị của hoạt động trước đó. Tham số đầu tiên của `run/3` là một map với kết quả của các hoạt động trước. Để lấy tags được trả về từ bước trước, chúng ta đơn giản chỉ cần sử dụng pattern matching trên `%{tags: tags)`.

Trong khi `run/3` khá là tiện dụng khi cần phải thực hiện những API mà `Ecto.Multi` chưa hỗ trợ trực tiếp, nó có một điểm dở đó là các hoạt động định nghĩa bởi `Ecto.Multi.run/3` là mờ (opaque), và do đó, chung không thể nào test được bằng cách dùng `Ecto.Multi.to_list/1` như chúng ta dùng ở phần trước. Mặc dù vậy, `Ecto.Multi` vẫn cho phép chúng ta có thể giảm thiểu rất nhiều những đoạn code rườm rà khi làm việc với transaction.
