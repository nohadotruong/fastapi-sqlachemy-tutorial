FastAPI khi kết hợp với **SQLAlchemy 2.0** (ORM) và **Alembic** (Migration Tool) là bộ cờ tiêu chuẩn để xây dựng Backend Python mạnh mẽ, chuẩn hóa và dễ mở rộng.

---

## 1. Môi trường & Luồng hoạt động

* **SQLAlchemy:** Định nghĩa cấu trúc các bảng (Model) trong Python và thực hiện các thao tác CRUD tới database.
* **Alembic:** Quản lý phiên bản cơ sở dữ liệu (Database Migration). Khi bạn sửa Model trong Python, Alembic giúp tự động phát hiện và sinh các file migration để đồng bộ vào Database.

```
[SQLAlchemy Model (Python)]  ──(alembic revision --autogenerate)──>  [File Migration]  ──(alembic upgrade head)──>  [Database]

```

---

## 2. Kiến thức cơ bản về FastAPI + SQLAlchemy

### Khởi tạo Engine, Session và Base (`database.py`)

Trong SQLAlchemy 2.0, bạn nên dùng `DeclarativeBase` để định nghĩa Model và `sessionmaker` để tạo phiên bản làm việc với DB.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker

# 1. Chuỗi kết nối Database (ví dụ SQLite hoặc PostgreSQL)
SQLALCHEMY_DATABASE_URL = "sqlite:///./app.db"
# Với PostgreSQL: "postgresql://user:password@localhost:5432/dbname"

# 2. Tạo Engine
engine = create_engine(
    SQLALCHEMY_DATABASE_URL, 
    connect_args={"check_same_thread": False} # Chỉ dùng tham số này cho SQLite
)

# 3. Tạo SessionLocal factory
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# 4. Base class để các Model kế thừa
class Base(DeclarativeBase):
    pass

# 5. Dependency injection lấy DB session trong FastAPI
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

```

### Định nghĩa Model (`models.py`)

Sử dụng cú pháp Mapped type hints mới của SQLAlchemy 2.0:

```python
from typing import Optional
from sqlalchemy import String, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from database import Base

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    email: Mapped[str] = mapped_column(String(100), unique=True)
    
    # Quan hệ 1-N với Item
    items: Mapped[list["Item"]] = relationship(back_populates="owner")

class Item(Base):
    __tablename__ = "items"

    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    title: Mapped[str] = mapped_column(String(100))
    description: Mapped[Optional[str]] = mapped_column(String(255), nullable=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    owner: Mapped["User"] = relationship(back_populates="items")

```

### Tạo API Endpoint (`main.py`)

Inject `get_db` bằng `Depends` để tương tác với DB trong mỗi Request:

```python
from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.orm import Session
from database import get_db
import models

app = FastAPI()

@app.get("/users/{user_id}")
def read_user(user_id: int, db: Session = Depends(get_db)):
    user = db.query(models.User).filter(models.User.id == user_id).first()
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user

```

---

## 3. Tổng hợp lệnh Alembic chuẩn

Để Alembic hoạt động, trước tiên bạn cần khởi tạo môi trường Alembic trong dự án.

### Khởi tạo môi trường

```bash
# Khởi tạo thư mục alembic (chỉ chạy 1 lần duy nhất đầu dự án)
alembic init alembic

```

> **Lưu ý cấu hình quan trọng sau khi init:**
> 1. Trỏ URL Database trong `alembic.ini` (hoặc cấu hình động trong `alembic/env.py`).
> 2. Trong `alembic/env.py`, import `Base` từ `database.py` và gán:
> `target_metadata = Base.metadata`
> 
> 

---

### Các lệnh làm việc thường dùng (Migration Workflow)

#### 1. Tạo file Migration tự động (Autogenerate)

Mỗi khi bạn thêm/sửa/xóa bảng hoặc trường trong `models.py`, hãy chạy lệnh này để Alembic so sánh `models.py` với Database hiện tại và tạo ra file mã lệnh migration:

```bash
alembic revision --autogenerate -m "Mô tả thay đổi (ví dụ: create users table)"

```

#### 2. Cập nhật Database lên phiên bản mới nhất (`upgrade`)

Áp dụng các file migration chưa được thực thi vào Database:

```bash
# Áp dụng tất cả migration đến bản mới nhất
alembic upgrade head

# Tiến lên 1 phiên bản kế tiếp
alembic upgrade +1

# Tiến đến một mã commit/revision cụ thể
alembic upgrade <revision_id>

```

#### 3. Revert / Hoàn tác thay đổi (`downgrade`)

Quay ngược Database về trạng thái cũ nếu bản migration vừa rồi bị lỗi hoặc cần khôi phục:

```bash
# Lùi lại 1 phiên bản gần nhất
alembic downgrade -1

# Trở về trạng thái ban đầu (xóa toàn bộ bảng do Alembic quản lý)
alembic downgrade base

# Lùi về một mã commit/revision cụ thể
alembic downgrade <revision_id>

```

#### 4. Xem lịch sử Migration (`history` & `current`)

```bash
# Kiểm tra phiên bản DB hiện tại đang ở revision nào
alembic current

# Xem toàn bộ lịch sử các file migration đã tạo
alembic history --verbose

```

---

## 4. Tóm tắt các bước workflow phát triển hàng ngày

1. **Cập nhật Model:** Chỉnh sửa file models.py.
Thêm class mới hoặc sửa các thuộc tính/cột trong file Python model.


2. **Sinh file Migration:** Chạy lệnh alembic revision.
Gõ `alembic revision --autogenerate -m "add phone number to users"` để sinh file migration mới trong thư mục `alembic/versions/`.


3. **Review mã lệnh Migration:** Kiểm tra file vừa tạo.
Mở file mới tạo trong `alembic/versions/` ra để kiểm tra xem hàm `upgrade()` và `downgrade()` đã chính xác chưa.


4. **Thực thi Upgrade:** Đồng bộ vào Database.
Gõ `alembic upgrade head` để áp dụng cấu trúc mới vào Database.
