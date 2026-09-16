# Chuyên Đề 21 — Dữ Liệu Lớn: Arrow, Parquet, Polars, DuckDB

## 21.1 Vì sao Pandas không đủ

Pandas:

- Dùng chung một luồng (trừ vài phép toán).
- Object dtype cho chuỗi — tốn bộ nhớ, chậm.
- Không lazy — mỗi bước tạo bản sao.
- File lớn hơn RAM là vấn đề.

Ba công cụ thay thế bổ sung nhau:

- **Arrow**: định dạng cột trong bộ nhớ, chuẩn chung.
- **Parquet**: định dạng file cột trên đĩa.
- **Polars**: dataframe đa luồng, lazy.
- **DuckDB**: SQL in-process trên Arrow/Parquet.

## 21.2 Apache Arrow

```bash
pip install pyarrow
```

```python
import pyarrow as pa
import pyarrow.compute as pc

bang = pa.table({
    "ten": ["An", "Bình", "Cường"],
    "tuoi": [20, 25, 22],
    "diem": [8.0, 9.5, 7.0],
})

print(bang.schema)
print(bang.num_rows, bang.num_columns)
print(bang.column("tuoi").to_pylist())
```

### Vì sao nhanh

- Lưu theo cột → cache CPU tốt, SIMD.
- Zero-copy giữa thư viện (Pandas, Polars, DuckDB, PyTorch).
- Không serialize khi chuyển giữa các bư��c.

```python
# Arrow → Pandas, zero-copy cho kiểu đơn giản
df = bang.to_pandas()

# Pandas → Arrow
bang2 = pa.Table.from_pandas(df)
```

### Compute kernel

```python
pc.sum(bang.column("tuoi"))            # tổng
pc.mean(bang.column("diem"))
pc.filter(bang, pc.greater(bang.column("tuoi"), 20))
```

## 21.3 Parquet

```python
import pyarrow.parquet as pq

# Ghi
pq.write_table(bang, "data.parquet", compression="zstd")

# Đọc
bang = pq.read_table("data.parquet")

# Chỉ đọc vài cột
bang = pq.read_table("data.parquet", columns=["ten", "diem"])
```

### Vì sao Parquet tốt

- Nén tốt hơn CSV 5-10 lần.
- Đọc theo cột — chỉ đọc cột cần.
- Lưu schema và kiểu.
- Hỗ trợ predicate pushdown (lọc ở tầng file).

### Dataset nhiều file

```python
import pyarrow.dataset as ds

dataset = ds.dataset("du-lieu/", format="parquet")
bang = dataset.to_table(filter=ds.field("tuoi") > 20)
```

Partition theo thư mục (`nam=2024/thang=01/`) giúp lọc nhanh.

### CSV vs Parquet

```python
import time

t0 = time.perf_counter()
pq.write_table(bang, "x.parquet")
t_pq = time.perf_counter() - t0

import pandas as pd
t0 = time.perf_counter()
bang.to_pandas().to_csv("x.csv", index=False)
t_csv = time.perf_counter() - t0

print(f"parquet {t_pq:.3f}s, csv {t_csv:.3f}s")
```

Trong thực tế Parquet nhỏ hơn ~5x và đọc nhanh hơn ~3-10x.

## 21.4 Polars

```bash
pip install polars
```

### Eager

```python
import polars as pl

df = pl.DataFrame({
    "ten": ["An", "Bình", "Cường", "Dũng"],
    "tuoi": [20, 25, 22, 30],
    "luong": [800.0, 1500.0, 1000.0, 2000.0],
})

print(df.filter(pl.col("tuoi") > 21))
print(df.select(["ten", pl.col("luong") * 1.1]))
```

### Lazy — điểm mạnh chính

```python
ket_qua = (
    pl.scan_parquet("du-lieu/*.parquet")
      .filter(pl.col("tuoi") > 21)
      .group_by("nhom")
      .agg([
          pl.col("luong").mean().alias("luong_tb"),
          pl.len().alias("so_nguoi"),
      ])
      .sort("luong_tb", descending=True)
      .collect()
)
```

`scan_*` không đọc dữ liệu. Polars xây query plan, tối ưu, rồi mới chạy. Predicate và projection được đẩy xuống tầng đọc file.

### Streaming

```python
ket_qua = (
    pl.scan_parquet("du-lieu/*.parquet")
      .group_by("nhom")
      .agg(pl.col("luong").sum())
      .collect(streaming=True)
)
```

Xử lý theo batch, không nạp hết vào RAM.

### Biểu thức

```python
df.with_columns([
    (pl.col("luong") / pl.col("tuoi")).alias("luong_tren_tuoi"),
    pl.col("ten").str.to_uppercase().alias("ten_hoa"),
    pl.when(pl.col("tuoi") >= 25).then(pl.lit("già")).otherwise(pl.lit("trẻ")).alias("nhom"),
])
```

### Join

```python
left = pl.DataFrame({"id": [1, 2], "ten": ["A", "B"]})
right = pl.DataFrame({"id": [1, 2], "diem": [8, 9]})

left.join(right, on="id", how="inner")
```

### So sánh Polars và Pandas

| Khía cạnh | Pandas | Polars |
|-----------|--------|--------|
| Đa luồng | Hạn chế | Có |
| Lazy | Không | Có |
| Chuỗi | Object (chậm) | Arrow (nhanh) |
| API | `df.method` + `[]` | Biểu thức `pl.col` |
| Hệ sinh thái | Rất rộng | Đang lớn |
| Dữ liệu > RAM | Khó | Streaming |

Chuyển đổi dễ:

```python
df_polars = pl.from_pandas(df_pandas)
df_pandas = df_polars.to_pandas()
```

## 21.5 DuckDB — SQL in-process

```bash
pip install duckdb
```

```python
import duckdb

# Query trực tiếp file Parquet, không cần nạp
ket_qua = duckdb.sql("""
    SELECT nhom, AVG(luong) AS luong_tb, COUNT(*) AS so_nguoi
    FROM 'du-lieu/*.parquet'
    WHERE tuoi > 21
    GROUP BY nhom
    ORDER BY luong_tb DESC
""").fetchdf()

print(ket_qua)
```

### Query Pandas/Polars/Arrow

```python
import pandas as pd
df = pd.DataFrame({"a": [1, 2, 3]})

duckdb.sql("SELECT SUM(a) FROM df").fetchone()
```

DuckDB tự nhận dataframe trong scope.

### Ghi Parquet partitioned

```python
duckdb.sql("""
    COPY (SELECT * FROM 'input.parquet' WHERE nam >= 2020)
    TO 'output/' (FORMAT PARQUET, PARTITION_BY (nam, thang))
""")
```

### DuckDB + Polars

```python
df = duckdb.sql("SELECT * FROM 'data.parquet'").pl()
```

`.pl()` trả Polars, `.arrow()` trả Arrow, `.df()` trả Pandas.

## 21.6 Chọn công cụ nào

| Tình huống | Công cụ |
|------------|---------|
| Dữ liệu < 1GB, phân tích nhanh | Pandas |
| Dữ liệu 1-100GB, pipeline | Polars lazy |
| SQL quen thuộc, ad-hoc | DuckDB |
| Cần lưu trữ hiệu quả | Parquet |
| Trao đổi giữa thư viện | Arrow |
| Stream real-time | Polars streaming / Arrow Flight |

Kết hợp thường dùng: **Parquet trên đĩa + Polars hoặc DuckDB để query**.

## 21.7 ETL pipeline mẫu

```python
from pathlib import Path
import polars as pl

def extract(thu_muc: Path) -> pl.LazyFrame:
    return pl.scan_parquet(thu_muc / "*.parquet")

def transform(lf: pl.LazyFrame) -> pl.LazyFrame:
    return (
        lf.filter(pl.col("trang_thai") == "hoan_thanh")
          .with_columns(
              pl.col("ngay").str.to_date("%Y-%m-%d"),
              (pl.col("so_tien") * pl.col("ty_gia")).alias("so_tien_vnd"),
          )
          .group_by(pl.col("ngay").dt.month().alias("thang"))
          .agg([
              pl.col("so_tien_vnd").sum().alias("tong"),
              pl.len().alias("so_giao_dich"),
          ])
    )

def load(lf: pl.LazyFrame, dich: Path) -> None:
    (
        lf.collect(streaming=True)
          .write_parquet(dich / "tong-hop.parquet", compression="zstd")
    )

def chay(nguon: Path, dich: Path) -> None:
    dich.mkdir(parents=True, exist_ok=True)
    load(transform(extract(nguon)), dich)
```

### Kiểm tra chất lượng dữ liệu

```python
def kiem_tra(lf: pl.LazyFrame) -> dict[str, int]:
    return (
        lf.select([
            pl.col("id").null_count().alias("id_null"),
            pl.col("id").is_duplicated().sum().alias("id_trung"),
            (pl.col("so_tien") < 0).sum().alias("so_tien_am"),
        ])
        .collect()
        .to_dicts()[0]
    )
```

## 21.8 Đọc file lớn hơn RAM với Pandas

Nếu buộc phải dùng Pandas:

```python
import pandas as pd

chunks = pd.read_csv("big.csv", chunksize=100_000)
tong = 0
for chunk in chunks:
    tong += chunk["so_tien"].sum()
```

### Giảm bộ nhớ

```python
dtypes = {
    "id": "int32",
    "tuoi": "int8",
    "danh_muc": "category",
    "so_tien": "float32",
}
df = pd.read_csv("data.csv", dtype=dtypes)
```

`category` cho chuỗi lặp nhiều giảm bộ nhớ 10-100x.

## 21.9 Bài tập

1. Chuyển một file CSV 1GB sang Parquet, so sánh kích thước và thời gian đọc.
2. Dùng Polars lazy tính top 10 nhóm theo tổng, đọc từ nhiều file Parquet.
3. Dùng DuckDB query cùng dữ liệu, so sánh tốc độ với Polars.
4. Viết ETL: đọc Parquet → lọc → ghi Parquet partition theo tháng.
5. Đo bộ nhớ khi dùng `category` so với object dtype trên 1 triệu dòng.
6. Dùng Polars streaming xử lý file 10GB (tạo giả nếu cần), đo RAM đỉnh.

## 21.10 Ghi chú kỹ thuật

- Parquet + Zstd thường tốt hơn Snappy về tỉ lệ nén, chậm hơn chút khi ghi.
- Polars dùng Arrow memory format — chuyển đổi với Pandas gần như zero-copy.
- DuckDB có thể query trực tiếp file Parquet mà không cần load — cực tiện cho ad-hoc.
- `scan_parquet` với glob đọc metadata trước, chỉ đọc cột cần.
- Partition theo cột có cardinality vừa phải; quá nhiều thư mục nhỏ làm chậm.
- Không dùng CSV cho dữ liệu lớn nếu có thể chọn Parquet.
- Kiểm tra `df.estimated_size()` (Polars) hoặc `df.memory_usage(deep=True)` (Pandas) trước khi xử lý.

Stashed.