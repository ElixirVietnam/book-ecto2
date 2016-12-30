# Multi tenancy với prefix trong truy xuất dữ liệu

Ecto 2.0 giới thiệu tính năng mới trong việc thực hiện câu truy vấn với những prefix khác nhau thông qua việc sử dụng cùng một kết nối tới database. Đối với những hệ quản trị cơ sở dữ liệu như Postgres, mỗi prefix từ Ecto sẽ ánh xạ tới schema trong [Postgres' DDL](https://www.postgresql.org/docs/current/static/ddl-schemas.html). Đối với MySQL, mỗi prefix sẽ là một database khác nhau. 

Các câu truy vấn với prefix có thể hữu dụng trong một số trường hợp. Ví dụ, ứng dụng cho nhiều đối tượng chạy trên Postgres có thể định nghĩa nhiều prefix, thường là mỗi prefix cho mỗi client, trong cùng một database.

Ý tưởng ở đây là prefix sẽ cung cấp việc độc lập dữ liệu giữa những đối tượng người dùng, bảo đảm cho cả toàn cục lẫn phân cấp dữ liệu khi mà truy vấn và thao tác trên một số prefix nhất định.

`Prefix` có thể cũng hữu dụng trong những hệ thống lưu lượng lớn, nơi mà dữ liệu đã được phân chia từ trước. Ví dụ, hệ thống game có thể chia dữ liệu thành từ phần độc lập, và mỗi phần được đặt tên với những prefix khác nhau. Sự phân chia này cho một player có thể lựa chọn ngẫu nhiên hay có tính toán dựa trên thông tin của player.

Trong khi truy vấn thông qua prefix được thiết kế với 2 ngữ cảnh trên, thì cũng có thể được sử dụng trọng một số trường hợp khác, chúng ta sẽ tìm hiểu cụ thể trong chương này. Những ví dụ dưới đây giả sử bạn sử dụng Postgres. Những hệ quản trị cơ sở dữ liệu khác có thể cần những giải pháp khác nhau.


## Global Prefixes

Cùng bắt đầu với một ví dụ đơn giản: ứng dụng của bạn phải truy suất tới một prefix cụ thể khi chạy trên production. Sự giới hạn này vì những điều kiện trong cấu trúc hạ tầng của hệ thống, phân quyền trong databbase và những yêu cầu cái khác.

Để bắt đầu chúng ta sẽ định nghĩa `repository` and `schema`:

```elixir
# lib/repo.ex
defmodule MyApp.Repo do
  use Ecto.Repo, otp_app: :my_app
end

# lib/sample.ex
defmodule MyApp.Sample do
  use Ecto.Schema

  schema "samples" do
    field :name

    timestamps
  end
end
```

Tiếp đến là cấu hình `repository`:

```elixir
config :my_app, MyApp.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "postgres",
  password: "postgres",
  database: "demo",
  hostname: "localhost",
  pool_size: 10
```

và định nghĩa `migration`:

```elixir
# priv/repo/migrations/20160101000000_create_sample.exs
defmodule MyApp.Migrations.CreateSample do
  use Ecto.Migration

  def change do
    create table(:samples) do
      add :name, :string
      timestamps
    end
  end
end
```

Bây giờ tạo database, `migrate` và tạo IEx session:

```elixir
$ mix ecto.create
$ mix ecto.migrate
$ iex -S mix
Interactive Elixir (1.4.0-dev) - press Ctrl+C to exit (type h() ENTER for help)
iex(1) > MyApp.Repo.All MyApp.Sample
[]
```

Chúng ta chưa thực hiện gì lạ cả. Chúng ta tạo một database, thêm vào những thông tin cho nó thông qua migrations và sau đó thực hiện câu truy vấn trong bảng `samples`, với kết quả là chuỗi rỗng.

Mặc định, kết nối tới database trong Postgres ở `public prefix`. Khi chúng ta thực hiện migrations và truy vấn, tất cả đều thực hiện trong `public prefix`. Tuy nhiên tưởng tượng là ứng dụng của bạn yêu cầu chạy trên prefix cụ thể trên production, giả sử là `global_prefix`.

May mắn Postgres cho phép chúng ta thay đổi prefix của kết nối tới database thông qua việc thiết lập "schema search path". Điều tuyệt vời nhất là thay đổi `search path` ngay khi thiết lập kết nối tới database, bảo đảm tất cả câu truy vấn của chúng ta đều thực hiện trên prefix xác định, thông qua kết nối đó.

Để làm vậy, thay đổi cấu hình của database ở `config/config.exs` và thêm vào `:after_connect`. `:after_connect` nhận vào một tuple với module, function và tham số tất cả sẽ đưa vào tiến trình kết nối, ngay khi kết nối tới database được xác lập:

```elixir
config :my_app, MyApp.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "postgres",
  password: "postgres",
  database: "demo_dev",
  hostname: "localhost",
  pool_size: 10,
  after_connect: {Postgrex, :query!, ["SET search_path TO global_prefix"], []}
```

Chạy thử một câu truy vấn giống ở trên:

```elixir
$ iex -S mix
Interactive Elixir (1.4.0-dev) - press Ctrl+C to exit (type h() ENTER for help)
iex(1) > MyApp.Repo.all MyApp.Sample
** (Postgrex.Error) ERROR (undefined_table): relation "samples" does not exist
```

Cùng một câu truy vấn nhưng giờ chúng ta gặp lỗi bởi vì ko có bảng "samples" ở prefix mới. Chúng ta thử sửa lỗi trên bằng cách chạy migrations:

```elixir
$ mix ecto.migrate
** (Postgrex.Error) ERROR (invalid_schema_name): no schema has been selected to create in
```

Bây giờ migration báo là không có schema với tên đó. Lỗi đó bởi vì Postgres sẽ tự động tạo "public" prefix mỗi khi chúng ta tạo database. Nếu chúng ta muốn tạo prefix khác, chúng ta phải tự tạo nó thông qua:

```elixir
$ psql -d demo_dev -c "CREATE SCHEMA global_prefix"
```

Giờ thì chúng ta có thể thực hiện migrate và các câu truy xuất:

```elixir
$ mix ecto.migrate
$ iex -S mix
Interactive Elixir (1.4.0-dev) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> MyApp.Repo.all MyApp.Sample
[]
```

Dữ liệu ở những prefix khác nhau sẽ được tách biệt. Thêm vào bảng "samples" ở một prefix này không thể truy suất bởi một prefix khác trừ khi chúng ta thay đổi prefix trong connection hoặc là sử dụng hỗ trợ từ Ecto cái mà chúng ta sẽ thảo luận sau đây.

## Per-query và per-struct prefixes

Trong khi chúng ta cấu hình để kết nối tới "global_prefix" ở `:after_connect`, thử chạy một số câu truy vấn:

```elixir
iex(1)> MyApp.Repo.all MyApp.Sample
[]
iex(2)> MyApp.Repo.insert %MyApp.Sample{name: "mary"}
{:ok, %MyApp.Sample{...}}
iex(3)> MyApp.Repo.all MyApp.Sample
[%MyApp.Sample{...}]
```

Những truy vấn ở trên thực hiện ở "global_prefix". Điều gì sẽ xảy ra nếu chúng ta thực hiện nó ở "public" prefix? Để làm được vậy, chúng ta sẽ tạo nên query struct, và thiết lập prefix một cách thủ công:

```elixir
iex(4) query = Ecto.Queryable.to_query MyApp.Sample
#Ecto.Query<from s in MyApp.Sample>
iex(5) MyApp.Repo.all %{query | prefix: "public"}
[]
```

Hãy để ý cách mà chúng ta thay đổi prefix khi thực hiện câu truy vấn. Với prefix là "public", kết quả trả về là rỗng.

Ecto 2.1 cũng hỗ trợ `:prefix` trên các thao tác trên repository:

```elixir
iex(6)> MyApp.Repo.all MyApp.Sample
[%MyApp.Sample{...}]
iex(7)> MyApp.Repo.all MyApp.Sample, prefix: "public"
[]
```

Một điểm thú vị từ prefix trong Ecto là thông tin về prefix sẽ giữ lại cùng với struct ở mỗi kết quả trả về sau truy vấn:

```elixir
iex(8)> [sample] = MyApp.Repo.all MyApp.Sample
[%MyApp.Sample{}]
iex(9)> Ecto.get_meta(sample, :prefix)
nil
```

Ví dụ ở trên trả về nil, điều đó nghĩa là không có prefix được thiệt lập bởi Ecto, và kết nối mặc định tới database sẽ được đùng. Trong trường hợp này, "global_prefix" sẽ được sử dụng bởi vì `:after_connect` mà chúng ta thêm vào ở đầu chương.

Bởi vì dữ liệu về prefix được lưu lại trong struct, chúng ta có thể sao chép dữ liệu đó từ một prefix này tới prefix kia. Chúng ta thử sao chép ví dụ ở trên từ "global_prefix" sang "public":

```elixir
iex(10)> public_sample = Ecto.put_meta(sample, prefix: "public")
%MyApp.Sample{}
iex(11)> MyApp.Repo.insert public_sample
{:ok, %MyApp.Sample{}}
iex(12)> [sample] = MyApp.Repo.all MyApp.Sample, prefix: "public"
[%MyApp.Sample{}]
iex(13)> Ecto.get_meta(sample, :prefix)
"public"
```

Chúng ta đã thêm dữ liệu vào cả 2 prefix.

Prefix trong câu truy vấn và struct luôn luôn có tính bắt cầu. Ví dụ, nếu bạn thực hiện `MyApp.Repo.preload(sample, [:some_association])`, thì quan hệ đó sẽ được truy vấn và tải lên với cùng một prefix với `sample`. Nếu `sample` có quan hệ và bạn thực hiện `MyApp.Repo.insert(sample)` hoặc `MyApp.Repo.update(sample` thì dữ liệu của quan hệ sẽ được thêm/cập nhật cùng một prefix với `sample`. Với thiết kế này sẽ đơn giản hoá khi làm việc với nhóm dữ liệu trong cùng một prefix, và đặc biệt vì **dữ liệu khác prefix phải cô lập với nhau**.

## Migration prefixes

Chúng ta đã tìm hiểu làm thế nào để gán global prefix trong Postgres và gán prefix ở mức độ câu truy vấn hay struct. Khi global prefix được gán, nó thay đổi prefix của migration sẽ chạy. Tuy nhiên nó cũng có thể gán prefix thông qua command line hoặc cho mỗi bảng khi chạy migration cho nó.

Ví dụ, tưởng tượng công ty game của bạn bị lỗi ở 128 vùng, lần lượt là "prefix_1", "prefix_2", "prefix_3" cho đến "prefix_128". Bất cứ khi nào bạn cần migrate dữ liệu, bạn cần phải migrate nó cho tất cả 128 prefix. Có 2 cách để thực hiện.

Phương pháp thứ 1 là gọi `mix ecto.migrate` mỗi lần cho mỗi prefix và truyền tham số `--prefix`:

```elixir
$ mix ecto.migrate --prefix "prefix_1"
$ mix ecto.migrate --prefix "prefix_2"
$ mix ecto.migrate --prefix "prefix_3"
...
$ mix ecto.migrate --prefix "prefix_128"
```

Một phương pháp khác là chạy migration đó với nhiều prefix. Ví dụ:

```elixir
defmodule MyApp.Repo.Migrations.CreateSample do
  use Ecto.Migration

  def change do
    for i <- 1..128 do
      prefix = "prefix_#{i}"
      create table(:samples, prefix: prefix) do
        add :name, :string
        timestamps()
      end

      # thực thi câu lệnh với prefix này
      # trước khi chuyển sang prefix khác
      flush()
    end
  end
end
```

## Schema prefixes

Cuối cùng, Ecto 2.1 thêm vào khả năng gán cho một prefix cụ thể khi chạy trên schema. Tưởng tượng bạn đang xây dựng một ứng dụng đa đối tượng. Mỗi đối tượng dữ liệu thuộc về một prefix xác định, như là "client_foo", "client_bar". Ứng dụng của bạn có thể vẫn phụ thuộc vào một tập các bảng được chia sẽ trên toàn đối tượng. Một trong các bảng có thể được xác định trước cho đối tượng nào đó với prefix. Giả sử bạn muốn lưu dữ liệu với prefix "main".

```elixir
defmodule Mapp.Mapping do
  use Ecto.Schema

  @schema_prefix "main"
  schema "mappings" do
    field :client_id, :integer
    timestamps
  end
end
```

Giờ chạy `MyApp.Repo.all MyApp.Mapping` sẽ tự động chạy trên prefix "main", mặc dù giá trị được gán ở `:after_connect` là khác. Tương tự sẽ xảy ra cho `insert`, `update` và những thao tác khác, `@schema_prefix` được sử dụng trừ khi `:prefix` được xác định rõ thông qua `Ecto.put_schema/2` hay truyền vào `:prefix` trên các thao tác đối với repository.

Các câu truy vấn luôn chạy trên cùng một prefix. Ví dụ, nếu `MyApp.Mapping` trên prefix "main" phụ thuộc vào schema tên `MyApp.Other` trên prefix "another" thì câu truy vấn bắt đầu bằng `MyApp.Mapping` sẽ luôn luôn chạy trên prefix "main". Với thiết kế này, thì việc thực hiện các câu truy vấn join bảng khác prefix là không thể. Nếu dữ liệu thuộc về những prefix khác nhau, thì tốt nhất là không nên liên kết chúng, để **giữ dữ liệu độc lập trên những prefix khác nhau**.

## Tổng kết

Ecto 2.0 cung cấp nhiều công cụ để thao tác với câu truy vấn có prefix. Những công cụ này được cải thiện nhiều hơn nữa trong Ecto 2.1, cho phép developer có thể cấu hình prefix với những cấp độ ưu tiên khác nhau:

```elixir
global prefixes > schema prefix > query/struct prefixes
```

Điều này cho phép delopever có thể dùng trong nhiều ngữ cảnh khác nhau, từ những yêu của production cho đến những ứng dụng đa đối tượng. Hành trình của chúng ta trong việc khám phá câu cấu trúc truy vấn đã gần kết thúc. Tiếp theo và là chương truy vấn cuối cùng là aggregates và subqueries.