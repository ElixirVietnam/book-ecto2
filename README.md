# Ecto là gì?

Ecto là **một tập các công cụ** để làm việc với các **nguồn dữ liệu từ bên ngoài** trong Elixir. Nguồn dữ liệu có thể là các Database \(như PostgreSQL, MySQL\), hay các API \(ví dụ như Github API\).

Ecto cung cấp bốn công cụ chính để làm việc với dữ liệu:

* `Ecto.Repo` - Là kho \(Repository\) tương tác với nguồn dữ liệu. Thông qua Repo, chúng ta có thể thực hiện các thao tác: tạo mới, cập nhật, xoá hoặc tìm kiếm các ban ghi. Một Repository sẽ cần một Adapter để liên lạc với nguồn dữ liệu. Bản thân thư viện Ecto cung cấp sẵn các Adapters để làm việc với các cơ sở dữ liệu phổ thông \(cụ thể là MySQL và PostgreSQL\)

* `Ecto.Query` - Cấu trúc dữ liệu để mô tả các câu Query chúng ta cần để tương tác với Database. Các Query này được mô tả bằng cú pháp của Elixir trong khi vẫn giữ được các đặc tính dễ đọc đê hiểu của SQL.

* `Ecto.Schema` - Schema được dùng để ánh xạ bất cứ nguồn dữ liệu nào tới một cấu trúc dữ liệu của Elixir \(cụ thể là Elixir Struct\). Chúng ta thường sử dụng Schema để ánh xạ một bảng trong Database vào một Struct trong Elixir. Tuy nhiên việc sử dụng Schema là rất linh hoạt, và hoàn toàn không hề bị giới hạn trong phạm vi của cơ sở dữ liệu.

* `Ecto.Changset` - Changeset cung cấp một cách để lập trình viên có thể lọc, chuẩn hoá dữ liệu từ các nguồn bên ngoài, cũng như cơ chế để theo dõi những sự thay đổi về dữ liệu trước khi chúng được áp dụng vào một bản ghi.





