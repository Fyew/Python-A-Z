# Chương 14 — Data, ML, Bảo Mật, CPython Internals

## 14.1 NumPy

```bash
pip install numpy
```

```python
import numpy as np

a = np.array([1, 2, 3])
b = np.arange(12).reshape(3, 4)

print(a * 2)              # vector hóa
print(b.T)                # chuyển vị
print(b.sum(axis=0))      # tổng theo cột
print(b.mean(), b.std())

c = np.array([[1], [2], [3]])
print(b + c)              # broadcasting

print(a[a > 1])           # boolean indexing
```

### Vì sao nhanh

NumPy lưu mảng liên tục trong bộ nhớ, phép toán viết bằng C. Vector hóa tránh vòng lặp Python.

```python
import timeit

setup = "import numpy as np; a = np.arange(1000000)"
print(timeit.timeit("a * 2", setup=setup, number=100))

setup2 = "a = list(range(1000000))"
print(timeit.timeit("[x * 2 for x in a]", setup=setup2, number=100))
```

NumPy nhanh hơn hàng chục lần.

### Các hàm quan trọng

```python
np.zeros((3, 4))
np.ones((2, 2))
np.eye(3)
np.random.default_rng(42).normal(0, 1, 1000)
np.linspace(0, 1, 11)

np.concatenate([a, a])
np.stack([a, a])
np.split(b, 3, axis=0)

np.where(a > 1, a, 0)
np.clip(a, 0, 2)
np.argmax(a), np.argsort(a)
```

## 14.2 Pandas

```bash
pip install pandas
```

```python
import pandas as pd

df = pd.DataFrame({
    "ten": ["Nam", "Lan", "Minh", "Hoa"],
    "tuoi": [25, 30, 22, 28],
    "luong": [1000, 1500, 800, 1200],
})

print(df.head())
print(df.describe())
print(df[df["tuoi"] > 23])
print(df.groupby("ten")["luong"].mean())
df["luong_tang"] = df["luong"] * 1.1
df.to_csv("out.csv", index=False)
```

### Đọc dữ liệu

```python
df = pd.read_csv("data.csv")
df = pd.read_json("data.json")
df = pd.read_excel("data.xlsx")
df = pd.read_sql("SELECT * FROM users", conn)
```

### Xử lý thiếu

```python
df.isna().sum()
df.dropna(inplace=True)
df["tuoi"] = df["tuoi"].fillna(df["tuoi"].mean())
df["nhom"] = df["nhom"].fillna("khác")
```

### Biến đổi

```python
df.rename(columns={"ten": "name"}, inplace=True)
df["tuoi_doi"] = df["tuoi"] * 2
df["nhom"] = df["tuoi"].apply(lambda x: "già" if x > 60 else "trẻ")

df.sort_values("luong", ascending=False)
df.nlargest(3, "luong")
```

### Merge và join

```python
left = pd.DataFrame({"id": [1, 2], "ten": ["A", "B"]})
right = pd.DataFrame({"id": [1, 2], "diem": [8, 9]})

pd.merge(left, right, on="id", how="inner")
pd.concat([left, left], ignore_index=True)
```

### Pivot và groupby

```python
df.groupby("nhom").agg({"luong": ["mean", "max", "count"]})
df.pivot_table(values="luong", index="nhom", columns="gioi_tinh", aggfunc="mean")
```

### Polars — thay thế nhanh hơn

```bash
pip install polars
```

```python
import polars as pl

df = pl.read_csv("data.csv")
result = (
    df.filter(pl.col("tuoi") > 23)
      .group_by("nhom")
      .agg(pl.col("luong").mean().alias("luong_tb"))
)
print(result)
```

Polars dùng Arrow, đa luồng, nhanh hơn Pandas cho dữ liệu lớn.

## 14.3 Machine Learning với scikit-learn

```bash
pip install scikit-learn
```

```python
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, classification_report

X, y = load_iris(return_X_y=True)
X_tr, X_te, y_tr, y_te = train_test_split(X, y, test_size=0.2, random_state=42)

model = RandomForestClassifier(n_estimators=100, random_state=42)
model.fit(X_tr, y_tr)
pred = model.predict(X_te)

print("Độ chính xác:", accuracy_score(y_te, pred))
print(classification_report(y_te, pred))
```

### Pipeline

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("clf", RandomForestClassifier()),
])
pipe.fit(X_tr, y_tr)
```

### Cross-validation

```python
from sklearn.model_selection import cross_val_score

scores = cross_val_score(model, X, y, cv=5)
print(scores.mean(), scores.std())
```

### Grid search

```python
from sklearn.model_selection import GridSearchCV

param_grid = {"n_estimators": [50, 100, 200], "max_depth": [None, 5, 10]}
grid = GridSearchCV(RandomForestClassifier(), param_grid, cv=5)
grid.fit(X, y)
print(grid.best_params_)
```

### Lưu model

```python
import joblib

joblib.dump(model, "model.joblib")
model = joblib.load("model.joblib")
```

## 14.4 PyTorch cơ bản

```bash
pip install torch
```

```python
import torch
import torch.nn as nn

model = nn.Sequential(
    nn.Linear(4, 16),
    nn.ReLU(),
    nn.Linear(16, 3),
)

optim = torch.optim.Adam(model.parameters(), lr=1e-3)
loss_fn = nn.CrossEntropyLoss()

x = torch.randn(32, 4)
y = torch.randint(0, 3, (32,))

for epoch in range(100):
    optim.zero_grad()
    out = model(x)
    loss = loss_fn(out, y)
    loss.backward()
    optim.step()
```

### Dataset và DataLoader

```python
from torch.utils.data import Dataset, DataLoader

class MyDataset(Dataset):
    def __init__(self, X, y):
        self.X, self.y = X, y

    def __len__(self):
        return len(self.X)

    def __getitem__(self, i):
        return self.X[i], self.y[i]

loader = DataLoader(MyDataset(x, y), batch_size=8, shuffle=True)
```

### GPU

```python
device = "cuda" if torch.cuda.is_available() else "cpu"
model.to(device)
x = x.to(device)
```

## 14.5 Bảo mật

### Hash mật khẩu

```bash
pip install argon2-cffi bcrypt
```

```python
from argon2 import PasswordHasher

ph = PasswordHasher()
hashed = ph.hash("matkhau123")
ph.verify(hashed, "matkhau123")

try:
    ph.verify(hashed, "sai")
except Exception:
    print("Sai mật khẩu")
```

Argon2 là khuyến nghị OWASP hiện tại. bcrypt cũng tốt.

```python
import bcrypt
hashed = bcrypt.hashpw(b"matkhau123", bcrypt.gensalt(rounds=12))
bcrypt.checkpw(b"matkhau123", hashed)
```

Không dùng MD5/SHA cho mật khẩu — quá nhanh.

### So sánh thời gian hằng

```python
import hmac
hmac.compare_digest(a, b)   # chống timing attack
```

### Sinh token an toàn

```python
import secrets
token = secrets.token_urlsafe(32)
otp = secrets.randbelow(1_000_000)
```

`random` không dùng cho bảo mật — có thể đoán seed.

### Biến môi trường

```python
import os
from dotenv import load_dotenv

load_dotenv()
API_KEY = os.environ["API_KEY"]
```

Không hardcode secret. Không commit `.env`.

### SQL injection

```python
# SAI
cur.execute(f"SELECT * FROM users WHERE name='{ten}'")

# ĐÚNG
cur.execute("SELECT * FROM users WHERE name=?", (ten,))
```

### Deserialize an toàn

```python
# NGUY HIỂM với dữ liệu không tin cậy
import pickle
data = pickle.loads(user_input)   # có thể thực thi code

# AN TOÀN
import json
data = json.loads(user_input)
```

### subprocess an toàn

```python
# SAI
import os
os.system(f"ls {path}")

# ĐÚNG
import subprocess
subprocess.run(["ls", path], check=True)
```

`shell=True` tái tạo lỗ hổng injection.

### Path traversal

```python
from pathlib import Path

BASE = Path("/srv/uploads").resolve()

def safe_path(user_path: str) -> Path:
    p = (BASE / user_path).resolve()
    if not p.is_relative_to(BASE):
        raise ValueError("ngoài thư mục cho phép")
    return p
```

### Nén zip an toàn — chống zip slip

```python
import zipfile
from pathlib import Path

def extract_an_toan(zip_path: str, dest: str):
    dest_p = Path(dest).resolve()
    with zipfile.ZipFile(zip_path) as z:
        for name in z.namelist():
            target = (dest_p / name).resolve()
            if not target.is_relative_to(dest_p):
                raise ValueError(f"zip slip: {name}")
            z.extract(name, dest_p)
```

## 14.6 CPython Internals

### Mô hình đối tượng

Mọi giá trị là `PyObject*` với refcount và con trỏ type.

```c
typedef struct _object {
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
} PyObject;
```

### Reference counting

```python
import sys

a = []
print(sys.getrefcount(a))   # 2: biến a + tham số tạm

b = a
print(sys.getrefcount(a))   # 3
```

Khi refcount về 0, object giải phóng. Chu trình được GC theo thế hệ xử lý.

```python
import gc
gc.collect()
gc.get_count()
gc.get_threshold()
```

### Bytecode

```python
import dis

def cong(a, b):
    return a + b

dis.dis(cong)
# LOAD_FAST a
# LOAD_FAST b
# BINARY_OP +
# RETURN_VALUE
```

3.11+ có specialized adaptive bytecode — tăng tốc sau vài lần chạy.

### GIL

GIL là mutex bảo vệ refcount và cấu trúc nội bộ. Nhả khi:
- Chờ I/O.
- Một số hàm C (numpy release GIL).
- `time.sleep`.

### `__slots__` và bộ nhớ

```python
import sys

class A:
    def __init__(self, x): self.x = x

class B:
    __slots__ = ("x",)
    def __init__(self, x): self.x = x

a, b = A(1), B(1)
print(sys.getsizeof(a.__dict__))   # ~104 byte
# B không có __dict__
print(sys.getsizeof(b))            # ~48 byte
```

### Small integer cache

Python cache số nguyên −5..256.

```python
a = 256
b = 256
a is b    # True

a = 257
b = 257
a is b    # thường False
```

Không bao giờ dựa vào `is` cho số.

## 14.7 C Extension

`mymodule.c`:

```c
#define PY_SSIZE_T_CLEAN
#include <Python.h>

static PyObject* cong(PyObject* self, PyObject* args) {
    long a, b;
    if (!PyArg_ParseTuple(args, "ll", &a, &b))
        return NULL;
    return PyLong_FromLong(a + b);
}

static PyMethodDef Methods[] = {
    {"cong", cong, METH_VARARGS, "Cộng hai số nguyên"},
    {NULL, NULL, 0, NULL}
};

static struct PyModuleDef module = {
    PyModuleDef_HEAD_INIT, "mymodule", NULL, -1, Methods
};

PyMODINIT_FUNC PyInit_mymodule(void) {
    return PyModule_Create(&module);
}
```

`pyproject.toml`:

```toml
[build-system]
requires = ["setuptools"]
build-backend = "setuptools.build_meta"

[project]
name = "mymodule"
version = "0.1.0"
```

`setup.py`:

```python
from setuptools import setup, Extension

setup(
    name="mymodule",
    version="0.1.0",
    ext_modules=[Extension("mymodule", ["mymodule.c"])],
)
```

```bash
pip install -e .
python -c "import mymodule; print(mymodule.cong(2, 3))"
```

## 14.8 Cython

```bash
pip install cython
```

`fast.pyx`:

```cython
def tong(int n):
    cdef long s = 0
    cdef int i
    for i in range(n):
        s += i
    return s
```

Biên dịch với `cythonize` — thường nhanh hơn Python thuần nhiều lần cho vòng lặp số.

## 14.9 Rust với PyO3

```bash
pip install maturin
maturin init --bindings pyo3
```

`src/lib.rs`:

```rust
use pyo3::prelude::*;

#[pyfunction]
fn cong(a: i64, b: i64) -> i64 {
    a + b
}

#[pymodule]
fn mymodule(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(cong, m)?)?;
    Ok(())
}
```

```bash
maturin develop
```

Rust cho hiệu năng tương đương C, an toàn bộ nhớ.

## 14.10 Bài tập

### Bài 1 — Phân tích dữ liệu CSV
Đọc CSV, tính thống kê, vẽ biểu đồ với matplotlib.

### Bài 2 — Model phân loại
Train model trên dataset tùy chọn, đánh giá cross-validation.

### Bài 3 — Password manager
Lưu Argon2 hash, CLI thêm/kiểm tra.

### Bài 4 — C extension
Viết C extension tính Fibonacci, so sánh tốc độ với Python.

### Bài 5 — Đo refcount
Viết script minh họa refcount tăng/giảm.

### Bài 6 — Dis bytecode
Dùng `dis` phân tích một hàm phức tạp.

## 14.11 Ghi chú kỹ thuật

- NumPy/Pandas release GIL trong nhiều phép toán — có thể dùng thread cho song song.
- Polars nhanh hơn Pandas cho dataset lớn; Pandas có hệ sinh thái rộng hơn.
- PyTorch dynamic graph dễ debug; TensorFlow tốt cho production serving.
- Hash mật khẩu cần cost cao đủ chậm — điều chỉnh theo phần cứng.
- C extension phải quản lý refcount cẩn thận — sai là rò rỉ hoặc crash.
- `gc` có thể tắt tạm cho hiệu năng, nhưng rủi ro rò rỉ chu trình.

Stashed.