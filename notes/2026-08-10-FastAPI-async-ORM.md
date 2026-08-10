# 学习日志｜2026-08-10

## 今日概览

今天学习了如何在 FastAPI 中使用 SQLAlchemy 异步 ORM 操作 MySQL。相比 Day 1 主要处理 HTTP 请求和响应，今天开始进入后端的数据持久化部分：将 Python 类映射为数据库表，通过异步数据库会话执行查询，并利用 FastAPI 的依赖注入统一管理会话和事务。

今天涉及的主要内容：

- ORM 的作用；
- 创建 SQLAlchemy 异步数据库引擎；
- 定义声明式基类和图书模型；
- 在 FastAPI 启动时创建数据表；
- 使用 `async_sessionmaker` 创建会话工厂；
- 使用 `Depends` 为路由注入数据库会话；
- 使用 `yield` 管理会话的使用周期；
- 事务提交、异常回滚；
- 按主键、条件和范围查询数据；
- 区分 Engine、Connection、Session 和 Result。

今天已经完成异步 ORM 查询的基础流程，但目前还没有实现新增、修改、删除、分页和响应模型，代码结构也仍以单文件练习为主。

---

## 知识点一：ORM

ORM，即对象关系映射，是在面向对象程序与关系型数据库之间建立映射的一种技术。

在今天的代码中：

```text
Python 类 Book        ↔ MySQL 表 book
Book.id               ↔ book.id 列
Book.bookname         ↔ book.bookname 列
一个 Book 对象         ↔ book 表中的一行数据
```

使用 ORM 后，可以通过 Python 对象和表达式描述数据库操作。例如：

```python
select(Book).where(Book.price >= 100.00)
```

它表达的是“查询价格大于等于 100 的图书”。SQLAlchemy 会将这个表达式转换为 SQL，再交给数据库执行。

### 当前理解

ORM 并不是取消了 SQL。程序仍然会向数据库发送 SQL，只是开发者可以先使用 Python 类和表达式组织查询。掌握 ORM 的同时仍然需要学习 SQL，因为表设计、复杂查询、索引和性能分析都离不开 SQL 基础。

---

## 知识点二：创建异步数据库引擎

```python
from sqlalchemy.ext.asyncio import create_async_engine

ASYNC_DB_URL = "mysql+aiomysql://<user>:<password>@localhost:3306/<database>?charset=utf8"

async_engine = create_async_engine(
    ASYNC_DB_URL,
    echo=True,
    pool_size=10,
    max_overflow=20,
)
```

连接地址中的主要部分：

| 部分 | 含义 |
| --- | --- |
| `mysql` | 使用 MySQL 数据库 |
| `aiomysql` | 使用异步 MySQL 驱动 |
| `user`、`password` | 数据库账号和密码 |
| `localhost:3306` | 数据库主机与端口 |
| `database` | 需要连接的数据库 |
| `charset=utf8` | 连接字符集配置 |

引擎的主要职责：

- 保存数据库连接配置；
- 管理数据库连接池；
- 在需要时提供连接；
- 协调 SQL 的执行和数据库返回结果。

### 引擎不是什么

Engine 不是某一次请求专用的 Session，也不是一条永远固定的数据库连接。应用通常创建一个长期存在的 Engine，再由不同 Session 在执行数据库操作时通过它取得连接。

### 连接池参数

| 参数 | 作用 |
| --- | --- |
| `echo=True` | 在控制台输出 SQL，方便学习和调试 |
| `pool_size=10` | 连接池中维持的基础连接数量 |
| `max_overflow=20` | 基础连接不足时允许额外创建的连接数量 |

这些数值目前属于练习配置。真实项目需要结合并发量、数据库允许的最大连接数和部署实例数量确定，不能简单认为越大越好。

### 安全注意事项

今天的源代码把数据库账号和密码直接写在连接字符串中。学习阶段虽然方便，但不适合提交到 Git 仓库或用于正式项目。后续应改为从环境变量或本地配置文件读取：

```python
import os

ASYNC_DB_URL = os.getenv("ASYNC_DB_URL")
```

本地配置文件应加入 `.gitignore`。如果代码中的密码也用于其他环境或已经上传到公开位置，应及时更换。

---

## 知识点三：定义 ORM 模型

### 声明基类

```python
from datetime import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    create_time: Mapped[datetime] = mapped_column(
        DateTime,
        insert_default=func.now(),
        default=func.now(),
        comment="创建时间",
    )
    update_time: Mapped[datetime] = mapped_column(
        DateTime,
        insert_default=func.now(),
        default=func.now(),
        onupdate=func.now(),
        comment="更新时间",
    )
```

`Base` 是所有 ORM 模型的共同基类。把创建时间和更新时间写在基类中，子类可以复用这些公共字段。

字段作用：

- `create_time`：记录数据创建时间；
- `update_time`：记录数据更新时间；
- `func.now()`：让数据库或 SQL 表达式产生当前时间；
- `onupdate=func.now()`：数据更新时刷新更新时间。

### 定义图书模型

```python
from sqlalchemy import Float, String


class Book(Base):
    __tablename__ = "book"

    id: Mapped[int] = mapped_column(primary_key=True, comment="书籍 id")
    bookname: Mapped[str] = mapped_column(String(255), comment="书名")
    author: Mapped[str] = mapped_column(String(255), comment="作者")
    price: Mapped[float] = mapped_column(Float, comment="价格")
    publisher: Mapped[str] = mapped_column(String(255), comment="出版社")
```

主要写法：

| 写法 | 作用 |
| --- | --- |
| `__tablename__ = "book"` | 指定对应的数据库表名 |
| `Mapped[int]` | 声明 Python 属性及其预期类型 |
| `mapped_column(...)` | 配置对应的数据库列 |
| `primary_key=True` | 将字段设置为主键 |
| `String(255)` | 最长为 255 的字符串列 |
| `Float` | 浮点数列 |

### 需要注意

价格在实际业务中通常不建议使用二进制浮点数保存，因为可能出现精度问题。后续学习表设计时，可以了解 `DECIMAL`/`Numeric` 类型。今天使用 `Float` 可以先用于理解 ORM 映射。

---

## 知识点四：应用启动时创建数据表

```python
async def create_tables():
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)


@app.on_event("startup")
async def startup_event():
    await create_tables()
```

执行过程：

```text
FastAPI 启动
→ 执行 startup_event
→ 调用 create_tables
→ Engine 提供连接并开启事务
→ 根据 Base.metadata 检查并创建表
```

`Base.metadata` 保存了 ORM 模型对应的表结构信息。`create_all()` 会创建尚不存在的表。

### 重要限制

`create_all()` 通常只负责创建不存在的表。如果表已经存在，后来修改模型字段，它不会像数据库迁移工具一样可靠地修改现有表结构。正式项目通常需要使用 Alembic 管理数据库版本和迁移。

另外，连接地址中指定的 MySQL 数据库本身通常需要事先存在；`create_all()` 创建的是表，不是整个数据库。

当前代码使用 `@app.on_event("startup")` 完成启动处理。能够理解其作用即可；后续接触新版 FastAPI 项目结构时，可以再学习 lifespan 生命周期写法。

---

## 知识点五：创建异步 Session 工厂

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker


AsyncSessionLocal = async_sessionmaker(
    bind=async_engine,
    class_=AsyncSession,
    expire_on_commit=False,
)
```

`async_sessionmaker` 是异步 Session 工厂，本身不是某次数据库会话。路由需要访问数据库时，由工厂创建一个新的 `AsyncSession`。

配置含义：

| 参数 | 含义 |
| --- | --- |
| `bind=async_engine` | 将 Session 工厂绑定到异步引擎 |
| `class_=AsyncSession` | 创建异步 Session，数据库操作通常需要 `await` |
| `expire_on_commit=False` | 提交后不立即让 ORM 对象属性过期 |

`expire_on_commit=False` 并不是“Session 永不过期”，也不是“永远不会重新查询数据库”。它只是在提交事务后保留当前 ORM 对象已经加载的属性，避免访问属性时因为过期而触发重新加载。

---

## 知识点六：使用依赖管理数据库会话

```python
async def get_database():
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

这段依赖项负责：

1. 创建异步 Session；
2. 通过 `yield` 暂时交给路由函数使用；
3. 路由执行完成后继续运行依赖项；
4. 正常情况下提交事务；
5. 发生异常时回滚事务；
6. 离开 `async with` 时关闭 Session。

### 为什么使用 `yield`

普通 `return` 返回结果后函数就结束了，而 `yield` 可以让依赖项在路由执行结束后继续完成提交、回滚和关闭等收尾工作。

### 在路由中注入 Session

```python
from fastapi import Depends


async def get_books_list(
    db: AsyncSession = Depends(get_database),
):
    ...
```

路由不需要反复编写创建 Session、异常回滚和关闭会话等代码，只需声明 `Depends(get_database)`。

### Engine、Session 与连接的关系

```text
应用创建 Engine
→ Engine 管理连接池
→ Session 工厂创建 Session
→ Depends 将 Session 注入路由
→ Session 执行 SQL 时向 Engine 获取连接
→ 数据库返回结果
→ Session 处理 ORM 对象和事务
```

这是今天最重要的整体关系。

### 关于自动提交

今天的依赖项会在每个使用它的路由正常结束后执行 `commit()`，查询接口也会经过这一步。作为事务管理练习可以帮助理解流程，但真实项目是否统一自动提交需要结合项目规范决定。后续学习新增和修改操作时，应重点理解事务边界，而不是把 `commit()` 当作固定模板机械复制。

---

## 知识点七：按主键查询

```python
@app.get("/book/books")
async def get_books_list(
    db: AsyncSession = Depends(get_database),
):
    book = await db.get(Book, 5)
    return book
```

`db.get(Book, 5)` 表示根据 `Book` 的主键查询值为 `5` 的记录。

它适合明确按照主键查找单条数据，返回结果是：

- 找到时：一个 `Book` ORM 对象；
- 未找到时：`None`。

### 当前代码中的问题

路由名和函数名表达的是“图书列表”，但实际只查询主键为 5 的一条记录。后续可以：

- 将接口改成按 ID 查询的语义；或者
- 使用 `select(Book)` 和 `scalars().all()` 真正返回图书列表。

---

## 知识点八：使用 `select` 进行条件查询

### 按图书 ID 查询单条数据

```python
@app.get("/book/get_book/{book_id}")
async def get_book_by_id(
    book_id: int,
    db: AsyncSession = Depends(get_database),
):
    result = await db.execute(
        select(Book).where(Book.id == book_id)
    )
    book = result.scalar_one_or_none()
    return book
```

执行过程：

1. `select(Book)` 构造查询语句；
2. `.where(Book.id == book_id)` 添加查询条件；
3. `await db.execute(...)` 异步执行语句；
4. 得到 SQLAlchemy Result 对象；
5. `scalar_one_or_none()` 提取一个 ORM 对象或 `None`。

`scalar_one_or_none()` 适合结果应该最多只有一条的情况。如果查询结果出现多条，它会报错，而不是随意取第一条。

目前图书不存在时接口会返回空值。结合 Day 1 学过的内容，后续应在找不到图书时抛出 404 异常。

### 按价格范围查询多条数据

```python
@app.get("/book/search_book")
async def get_search_book(
    db: AsyncSession = Depends(get_database),
):
    result = await db.execute(
        select(Book).where(Book.price >= 100.00)
    )
    books = result.scalars().all()
    return books
```

这里使用：

- `Book.price >= 100.00` 构造价格条件；
- `result.scalars()` 从结果中提取 ORM 实体；
- `.all()` 取得所有符合条件的图书。

因为该查询可能返回多条数据，所以使用 `scalars().all()`，而不是 `scalar_one_or_none()`。

### 查询所有图书

手记中记录的写法可以整理为：

```python
result = await db.execute(select(Book))
books = result.scalars().all()
```

也可以写成一行，但学习阶段拆开更容易观察每一步的返回值。

---

## 知识点九：Result 的提取方式

`await db.execute(...)` 返回的是 SQLAlchemy Result 对象，而不是可以直接使用的普通图书列表。

今天接触了几种提取方式：

| 写法 | 适用情况 |
| --- | --- |
| `result.scalars().all()` | 提取全部 ORM 对象 |
| `result.scalars().first()` | 取第一条，没有则返回 `None` |
| `result.scalar_one_or_none()` | 预期最多一条，没有则返回 `None`，多条则报错 |
| `db.get(Book, id)` | 按主键获取单个 ORM 对象 |

选择方式时要先判断业务预期：需要一条、第一条、全部数据，还是明确按照主键查询。

---

## 今日请求流程

以按 ID 查询图书为例：

```text
客户端请求 GET /book/get_book/5
→ FastAPI 匹配路由并解析 book_id
→ Depends 执行 get_database
→ Session 工厂创建 AsyncSession
→ Session 注入路由参数 db
→ 路由构造 select(Book) 查询
→ Session 通过 Engine 获取数据库连接
→ SQLAlchemy 将表达式转换为 SQL
→ MySQL 执行查询并返回结果
→ Result 提取 Book 对象或 None
→ FastAPI 生成响应
→ 依赖项恢复执行并完成事务收尾
→ Session 关闭
```

如果能够脱离笔记讲清这条流程，说明已经理解了今天知识点之间的联系。

---

## 今日实践与发现

今天完成了三类查询练习：

- `db.get()` 按主键查询；
- `select + where + scalar_one_or_none()` 查询单条数据；
- `select + where + scalars().all()` 查询多条数据。

代码注释比单独手记更加详细，已经记录了 Engine、Session、依赖注入和 Result 的关系。这种记录方式可以继续使用，但注释最好重点解释“为什么这样设计”和“容易混淆的概念”，不必为每一行都重复描述表面动作。

当前尚未覆盖：

- 新增图书；
- 修改图书；
- 删除图书；
- 查询分页；
- 图书不存在时返回 404；
- 使用 Pydantic 响应模型；
- 模型和数据库表发生变化后的迁移。

---

## 掌握情况

| 知识点 | 当前状态 | 下一步验收方式 |
| --- | --- | --- |
| ORM 基本概念 | 初步理解 | 用自己的话解释对象、类、表和行的映射 |
| 异步 Engine | 初步使用 | 不看代码重新创建引擎并解释连接池 |
| ORM 模型定义 | 初步使用 | 独立定义另一个模型及表名、主键和字段 |
| 启动时建表 | 初步使用 | 说明 `create_all()` 能做什么、不能做什么 |
| Session 工厂 | 初步理解 | 区分 Engine、工厂和 Session |
| Session 依赖 | 重点巩固 | 独立写出 `yield`、提交和回滚流程 |
| 按主键查询 | 已完成基础使用 | 查询任意路径参数对应的图书 |
| 条件查询 | 已完成基础使用 | 将最低价格改为查询参数 |
| Result 数据提取 | 初步使用 | 根据单条或多条预期选择正确方法 |

当前最需要巩固的是数据库会话依赖。它连接了 FastAPI、事务和 SQLAlchemy，是后续所有增删改查操作都会使用的基础。

---

## 容易遗忘或混淆的内容

- Engine 管理数据库配置和连接池，Session 负责一次工作单元中的 ORM 操作和事务；
- `async_sessionmaker` 是 Session 工厂，不是 Session 对象；
- `yield session` 之后的代码会在路由完成后继续执行；
- 异步 Session 执行数据库操作时通常需要 `await`；
- `execute()` 返回 Result，需要进一步提取 ORM 对象；
- 查询单条和多条数据应选择不同的结果提取方法；
- `db.get()` 主要用于主键查询；
- `create_all()` 不能代替数据库迁移；
- `expire_on_commit=False` 只控制提交后 ORM 属性是否立即过期；
- 数据库密码不应直接写入并提交到源代码；
- ORM 不能代替 SQL、数据库表设计和索引知识。

---

## 今日总结

今天完成了 FastAPI 从“接收请求”到“访问 MySQL 并返回数据”的基础链路。核心不是记住三个查询接口，而是理解以下关系：

```text
模型描述表结构
Engine 管理连接
Session 管理 ORM 操作与事务
Depends 管理 Session 生命周期
select 构造查询
Result 提取最终对象
```

目前已经能够跟随课程实现异步 ORM 建表和查询，下一阶段需要脱离现有代码，独立完成新增图书以及更合理的按 ID 查询，并将 Day 1 的请求体验证、响应模型和异常处理与今天的数据库操作组合起来。

---

## 复习检查

合上代码和笔记后尝试回答：

1. ORM 解决了 Python 对象和关系型数据库之间的什么问题？
2. Engine、Connection、Session 和 Session 工厂分别是什么？
3. 为什么应用通常只创建一个 Engine，却可以为不同请求创建多个 Session？
4. `pool_size` 和 `max_overflow` 分别控制什么？
5. `Mapped[int]` 和 `mapped_column()` 分别起什么作用？
6. `Base.metadata.create_all()` 能做什么，为什么不能替代数据库迁移？
7. 为什么 `get_database()` 使用 `yield` 而不是直接 `return`？
8. 路由发生异常时为什么需要回滚事务？
9. `db.get(Book, 5)` 与 `select(Book).where(...)` 有什么区别？
10. `scalars().all()` 与 `scalar_one_or_none()` 分别适合什么情况？
11. 为什么执行 `db.execute()` 后不能直接把 Result 当作图书列表？
12. 为什么数据库连接字符串不应该直接写进源码？

如果能够不看笔记讲清第 2、7、9、10 题，并独立完成下一步练习，今天的知识就基本达到可应用程度。

---

## 下一步

优先在当前项目中完成以下练习：

1. 将 `/book/books` 改成真正查询全部图书的列表接口；
2. 将最低价格设计为查询参数，而不是固定写死为 `100.00`；
3. 查询指定 ID 的图书时，如果结果为 `None`，抛出状态码为 404 的 `HTTPException`；
4. 为图书返回数据定义 Pydantic 响应模型；
5. 将数据库连接地址移动到环境变量；
6. 尝试不看课程代码重新写出 `get_database()` 依赖项。

完成查询巩固后，再继续学习新增、修改和删除操作。这样可以在进入完整 CRUD 前，先确保异步 Session、依赖注入和 Result 提取方式已经理解。
