# Quan hệ nhiều nhiều và upserts

Trong chương trước, chúng ta đã học về `many_to_many` và cách để ánh xạ dữ liệu từ bên ngoài tới các thực thể liên kết bằng hàm `Ecto.Changeset.cast_assoc/3`. Để làm chuyện đó, chúng ta phải tuân theo các luật định nghĩa bởi hàm `cast_assoc/3`. Điều này đôi khi không phải là mong muốn của lập trình viên, cũng như trong một số tình huống là bất khả thi.

Trong chương này, chúng ta sẽ tìm hiểu về `Ecto.Changeset.put_assoc/4` và khám phá một số ví dụ. Chúng ta cũng sẽ nói về tính năng upsert đi kèm với Ecto 2.1.

## put_assoc và cast_assoc

Tưởng tượng chúng ta đang xây dựng một ứng dụng blog, mỗi một post có thể có nhiều tag. Không chỉ vậy, một tag có thể thuộc về nhiều post. Đây là tình huống cổ điển để sử dụng liên kết `many_to_many`. Chúng ta có thể tổ chức Database như sau:

```elixir
create table(:posts) do
  add :title
  add :body
  timestamps()
end

create table(:tags) do
  add :name
  timestamps()
end

create unique_index(:tags, [:name])

create table(:posts_tags, primary_key: false) do
  add :post_id, references(:posts)
  add :tag_id, references(:tags)
end
```

Chú ý rằng chúng ta thêm vào một "unique index" cho tag name bởi vì chúng ta không muốn có các tag bị lặp trong Database. Việc thêm index kiểm tra tính duy nhất ở mức Database thay vì ở mức ứng dụng là rất quan trọng, vì sẽ luôn luôn có trường hợp 2 tag với cùng tên sẽ được kiểm tra và cùng được insert vào Database một lúc, việc này sẽ dẫn tới chuyện chúng ta có các thực thể trùng nhau.

Giờ hãy cùng nhau tưởng tượng rằng chúng ta muốn người dùng điền vào các tags như một danh sách các từ được ngăn cách bởi dấu phẩy, ví dụ như: "elixir,erlang,ecto". Một khi dữ liệu này được gửi tới server, chúng ta sẽ chia nhỏ nó thành nhiều tag và liên kết chúng với post hiện tại, đồng thời tạo ra các tag chưa có trong Database.

Trong khi các điều kiện trên khá là hợp lý, nó lại tạo ra các vấn đề khác với `cast_assoc/3`. Nhớ rằng hàm `cast_assoc/3` được thiết kế để nhận vào các tham số và so sánh chúng với các dữ liệu liên kết với struct của chúng ta. Để làm điều này, Ecto yêu cầu các tag cần phải được gửi như là một list các map. Tuy nhiên ở đây, chúng ta mong muốn tag sẽ được gửi lên như một chuỗi được ngăn cách bởi các dấu phảy.

Hơn thế nữa `cast_assoc/3` phụ thuộc vào trường primary key cho mỗi tag để có thể quyết định nó nên insert, update hoặc là delete. Lại một lần nữa, bởi vì người dùng đơn giản chỉ gửi lên một xâu, chúng ta không thể có thông tin về ID.

Khi chúng ta không thể sử dụng `cast_assoc/3`, đó chính là lúc sử dụng `put_assoc/4`. Với `put_assoc/4`, chúng ta đưa cho Ecto các struct hoặc các changeset thay vì các tham số, điều này giúp chúng ta có khả năng kiểm soát dữ liệu theo cách chúng ta muốn. Hãy cùng định nghĩa schema và hàm changeset cho một post để có thể nhận vào các tags như một chuỗi:

```elixir
defmodule MyApp.Post do
  use Ecto.schema

  schema "posts" do
    add :title
    add :body
    many_to_many :tags, MyApp.Tag, join_through: "posts_tags"
    timestamps()
  end

  def changeset(struct, params \\ %{}) do
    struct
    |> Ecto.Changeset.cast(struct, [:title, :body])
    |> Ecto.Changeset.put_assoc(:tags, parse_tags(params)
  end

  defp parse_tags(params) do
    (params["tags"] || "")
    |> String.split(",")
    |> Enum.map(&String.trim/1)
    |> Enum.reject(& &1 == "")
    |> Enum.map(&get_or_insert_tag/1)
  end

  defp get_or_insert_tag(name) do
    Repo.get_by(MyApp.Tag, name: name) ||
      Repo.insert!(MyApp.Tag, %Tag{name: name})
  end
end
```

Trong hàm changeset ở trên, chúng ta đã di chuyển tất cả mọi việc liên quan tới xử lý tags sang một hàm mới `parse_tags/1`, hàm này sẽ kiểm tra tham số, chia nhỏ nó thành các phần dựa vào `String/split/2`, sau đó loại bỏ đi các dấu cách ở đầu và cuối mỗi phần tử với `String.trim/1`, xoá đi các phần tử rỗng và cuối cùng, kiểm tra xem tag có tồn tại trong Database hay không, nếu chưa có thì tạo mới.

Hàm `parse_tags/1` sẽ trả về một list các `MyApp.Tag` mà sau đó sẽ được truyền cho `put_assoc/3`. Bằng việc gọi `put_assoc/3`, chúng ta nói với Ecto rằng, list này sẽ là các tags được liên kết với post hiện tại. Trong trường hợp có một tag đã từng được liên kết với post, nhưng không có trong tag hiện tại, Ecto sẽ tự động xoá liên kết của tag đó với post, và xoá tag khỏi Database.

Và đó là tất cả những gì chúng ta cần để sử dụng `many_to_many` với `put_assoc/3`. `put_assoc/3` làm việc với `has_many`, `belongs_to` và tất cả các kiểu liên kết khác. Tuy nhiên, code của chúng ta chưa sẵn sàng để lên production. Hãy cùng xem vì sao.

## Constraints và race coditions

Nhớ rằng, chúng ta đã thêm vào một unique index cho cột `:name` khi tạo bảng "tags". Chúng ta làm như vậy để bảo vệ Database khỏi các tag trùng lặp.

Bằng việc thêm unique index và sau đó sử dụng `get_by` với `insert!` thể lấy ra hoặc tạo mới một tag, chúng ta đã tạo ra một bug tiền tàng trong ứng dụng. Nếu như 2 posts được tạo ra đồng thời với cùng một tags, sẽ có trường hợp chúng ta kiểm tra tag có tồn tại hay không cùng lúc, dẫn đến cả 2 việc kiểm tra này  đều nghĩ rằng chưa có tag nào trong Database. Khi điều này xảy ra, chỉ một trong hai quá trình sẽ thành công, trong khi cái còn lại sẽ thất bại. Đó chính là "race condition": code của bạn sẽ sai theo thời gian, khi mà thoả mãn nó thoả mãn một vài điều kiện rất đặc biệt.

Rất nhiều lập trình viên có xu hướng suy nghĩ là những lỗi như vậy sẽ không gặp trong thực tế, hoặc là nó xảy ra, nó cũng không quan trọng lắm. Tuy nhiên trên thực tế, chúng dẫn tới rất nhiều trải nghiệm xấu cho người dùng. Tôi đã từng nghe một ví dụ về một công ty game, trong game, nếu một người chơi có thể chơi nhiều nhiệm vụ, và trong mỗi nhiệm vụ, bạn phải lựa chọn một nhân vật khách từ một người chơi khác từ một danh sách trong nhiệm vụ của bạn. Đến cuối nhiệm vụ, bạn có quyền lựa chọn thêm nhân vật khác như là bạn của mình.

Thông thường, toàn bộ danh sách khách là ngẫu nhiên, nhưng theo thời gian, các người chơi bắt đầu phàn nàn rằng đôi khi những tài khoản cũ, thường xuyên không active, được hiển thị trong danh sách khách mời của họ. Để cải thiện tình hình, các lập trình viên trong game bắt đầu sắp xếp danh sách khách theo thứ tự thời gian active. Điều đó có nghĩa là, nếu bạn mới chơi gần đây, sẽ có khả năng cao là bạn sẽ nằm trong danh sách khách của người khác.

Tuy nhiên, khi họ thay đổi game như vậy, rất nhiều lỗi đã xảy ra, và người chơi, và người chơi trở nên rất giận dữ trong diễn đàn. Đó là bởi vì khi họ sắp xếp danh sách người chơi theo độ active, ngay khi có 2 người chơi cùng đăng nhập, nhân vật của họ có thể xuất hiện trong danh sách của nhau. Nếu 2 người chơi này chọn nhân vật của người kia, người đầu tiên có thể thêm người còn lại là bạn trong danh sách ở cuối nhiệm vụ, nhưng một lỗi sẽ bị hiện ra với người thứ hai khi người này thêm người thứ nhất vào danh sách bạn của mình, vì mối quan hệ giữa hai người đã tồn tại trong Database. Không chỉ vậy, cả quá trình đã hoàn thành của nhiệm vụ cũng bị mất, bởi vì server không thể lưu lại kết quả của nhiệm vụ vào Database. Không ngạc nhiên người chơi bắt đầu phàn nàn.

Nói tóm lại: chúng ta phải để tâm tới race condition.

May mắn thay, Ecto cung cấp một cơ chế để có thể xử lý những lỗi xảy ra từ Database.

## Kiểm tra cho constraint errors

Chúng ta hãy viết lại hàm `get_or_insert_tag(name)` với việc chú ý tới race condition:

```elixir
defp get_or_insert_tag(name) do
  %MyApp.Tag{}
  |> Ecto.Changeset.change(name: name)
  |> Ecto.Changeset.unique_constraint(:name)
  |> Repo.insert
  |> case do
    {:ok, tag} -> tag
    {:error_, _} -> Repo.get_by!(MyApp.Tag, name: name)
  end
end
```

Thay vì insert tag trực tiếp, chúng ta tạo ra một changeset để cho phép chúng ta sử dụng hàm `unique_constraint`. Giờ nếu như hành động `Repo.insert` thất bại bởi vì unique index của `:name`, Ecto sẽ không raise Exception, mà trả về một tuple `{:error, changest}`. Do đó, nếu `Repo.insert` thành công thì nghĩa là tag đã được lưu lại, còn nếu không tức là tag đã tồn tại trong Database, lúc này chúng ta sẽ lấy tag ra bằng hàm `Repo.get_by!`.

Trong khi cơ chế trên fix được race condition, nó lại là một hành động khá tốn kém: chúng ta cần phải gọi hai câu query cho mỗi tag đã tồn tại trong Database: một câu insert (thất bại), và một câu get. Dựa vào độ phổ biến của các trường hợp, chúng ta có thể viết lại nó như sau:

```elixir
defp get_or_insert_tag(name) do
  Repo.get_by(MyApp.Tag, name: name) || maybe_insert_tag(name)
end

defp maybe_insert(name) do
  %MyApp.Tag{}
  |> Ecto.Changeset.change(name: name)
  |> Ecto.Changeset.unique_constraint(:name)
  |> Repo.insert
  |> case do
    {:ok, tag} -> tag
    {:error, _} -> Repo.get_by!(MyApp.Tag, name: name)
  end
end
```

Đoạn code trên sẽ gọi một 1 query cho mỗi tag đã tồn tại, 2 query cho mỗi tag mới, và có thể là 3 query cho trường hợp race condition. Mặc dù cách giải quyết như vậy là tốt hơn so với cách cũ, nhưng Ecto 2.1 còn cung cấp một lựa chọn tốt hơn.

## Upserts

Ecto 2.1 hỗ trợ tính năng "upsert" - tức là "update hoặc insert". Ý tưởng là chúng ta sẽ cố gắng insert một bản ghi và trong trường hợp nó bị xung đột với một bản ghi đã tồn tại trong Database, ví dụ vì unique index, chúng ta có thể lựa chọn cách mà chúng ta muốn Database hành động bằng cách hoặc là raise Exception (cách mặc định), hoặc là bỏ qua hành động insert (và không có lỗi), hoặc là cập nhật sự xung đột với các bản ghi trong Database.

`upsert` trong Ecto 2.1 được thực hiện với lựa chọn `:on_conflict`. Hãy cùng viết lại `get_or_insert_tag(name)` một lần nữa, nhưng lần này chung ta sử dụng lựa chọn `:on_conflict`. Cũng cần nhớ rằng chức năng "upsert" là một tính năng của PostgreSQL 9.5, vì thế hãy chắc chắn là bạn đã có bản cập nhật này:

```elixir
defp get_or_insert_tag(name) do
  Repo.insert!(%MyApp.Tag{name: name}, on_conflict: :nothing)
end
```

Chúng ta cố gắng thêm một tag với `name` vào Database, và nếu như tag đã tồn tại, chúng ta nói với Ecto rằng đó không phải là lỗi, và trả về tag mà chúng ta đã truyền vào như một tham số nếu như nó đã có trong Database. Trong khi cách làm ở trên là một giải pháp nâng cấp so với những giải pháp trước đây, nhưng nó vẫn thực hiện 1 câu query cho mỗi một tag, nếu có 10 tags được gửi lên, nó sẽ thực hiện 10 câu query. Liệu chúng ta có thể cải tiến nó không?

## Upserts và insert_all

Ecto 2.1 không chỉ thêm vào lựa chọn `:on_conflict` vào `Repo.insert/2` mà còn thêm nó vào cả API `Repo.insert_all/3`. Điều đó có nghĩa là chúng ta có thể xây dựng một câu query có thể insert tất cả các tags còn thiếu và một câu query khác lấy ra tất cả những tag cùng một lúc. Hãy cùng xem `Post` scheam sẽ như nào với những thay đổi trên:

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
    |> Ecto.Changeset.cast(struct, [:title, :body])
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

Thay vì cố gắng get và insert từng tag một, đoạn code trên làm việc với tất cả các tags một lúc, đầu tiên nó build một list các maps, sau đó truyền maps này cho `insert_all`, cuối cùng lấy ra tất cả các tags đã có trong Database. Bất kể bao nhiêu tag được gửi lên đi nữa, chúng ta cũng chỉ cần thực hiện 2 thao tác (trừ trường hợp không có tag nào được gửi lên, chúng ta sẽ trả về một list rỗng ngay lập tức). Giải pháp này khả thi trong Ecto 2.1 là nhờ có lựa chọn `:on_conflict` đã đảm bảo rằng `insert_all` không thất bại nếu có một tag đã tồn tại.

Cuối cùng, nhớ rằng chúng không chưa hề sử dụng trấnction trong bất kỳ ví dụ nào trong cuốn sách. Quyết định này là có lý do. Từ việc lấy ra hoặc insert các tag là một hoạt động không thay đổi (idempotent) theo nghĩa chúng ta có thể lặp lại nó rất nhiều lần, nhưng kết quả vẫn luôn luôn là như nhau. Do đó, nếu chúng ta thất bại trong việc lưu trữ post vào database bởi vì một lỗi ở lúc kiểm trả, người dùng vẫn được tự do để submit lại form của họ, và chúng ta chỉ cần get hoặc insert tag một lần nữa. Bất lợi của cách làm này là tag có thể sẽ được tạo ra mặc dù việc tạo ra post thất bại. Điều này dẫn tới một vài tags sẽ không liên kết với bất cứ post nào. Trong trường hợp bạn không muốn điều này xảy ra, toàn bộ hành động này nên được bao trong một transaction, hoặc là được mô hình hoá với `Ecto.Multi` như chúng ta sẽ thảo luận trong chương kế tiếp.
