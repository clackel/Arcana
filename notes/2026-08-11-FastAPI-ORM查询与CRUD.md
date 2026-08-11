# 学习日志｜2026-08-11

## 今日概览

今天在 Day 2 已完成的 FastAPI 异步 ORM 基础上，继续学习 SQLAlchemy 查询表达式与完整 CRUD 操作。数据库引擎、ORM 模型、异步 Session 工厂和 `get_database()` 依赖仍沿用昨天的代码，本篇不再重复整理。

今日新增内容：

- 使用 `LIKE` 进行模糊查询；
- 使用“与”“或”组合多个查询条件；
- 使用 `in_()` 查询指定 ID 集合；
- 使用聚合函数统计数据；
- 使用 `offset()` 和 `limit()` 实现分页；
- 使用 Pydantic 模型接收新增与更新请求；
- 将请求数据转换为 ORM 对象并新增记录；
- 按主键查询后更新记录；
- 按主键查询后删除记录；
- 数据不存在时返回 404 异常。

今天已经把 Day 2 的“查询数据库”扩展成了基本完整的图书 CRUD 流程。

---

## 知识点一：模糊查询

SQLAlchemy 可以使用 `like()` 构造 SQL 的 `LIKE` 查询。

```python
select(Book).where(Book.author.like("曹%"))
```

这里的 `%` 可以匹配任意长度的字符，包括零个字符。因此，`曹%` 表示查询所有以“曹”开头的作者。

```python
select(Book).where(Book.author.like("曹_"))
```

`_` 只匹配一个字符，所以 `曹_` 表示匹配恰好由“曹”和另一个字符组成的作者名。

| 通配符 | 含义 | 示例 |
| --- | --- | --- |
| `%` | 匹配任意长度字符 | `曹%` |
| `_` | 只匹配一个字符 | `曹_` |

### 容易混淆的地方

`%` 与 `_` 都是 `LIKE` 模式中的特殊通配符。它们不是 Python 自身的匹配规则，而是最终生成的 SQL 查询条件。

---

## 知识点二：组合查询条件

### 同时满足多个条件

```python
select(Book).where(
    (Book.author.like("曹%")) & (Book.price >= 150)
)
```

`&` 表示“与”，对应 SQL 中的 `AND`。查询结果必须同时满足：

- 作者以“曹”开头；
- 价格大于等于 150。

### 满足任意一个条件

```python
select(Book).where(
    (Book.author.like("曹%")) | (Book.price >= 100)
)
```

`|` 表示“或”，对应 SQL 中的 `OR`。只要满足其中一个条件就可以返回。

### 为什么条件外面要加括号

Python 运算符存在优先级。组合 SQLAlchemy 条件时给每个条件加括号，可以避免表达式被错误解析，也能让代码语义更清晰。

今天的手记标题提到了“非”，但代码中实际练习的是“与”和“或”。“非”条件尚未真正实践，之后可以再学习 `not_()` 或 `~` 的写法。

---

## 知识点三：包含查询

需求是：给出一组图书 ID，返回数据库中 ID 位于这组数据中的图书。

```python
id_list = {1, 3, 5, 7}

result = await db.execute(
    select(Book).where(Book.id.in_(id_list))
)
books = result.scalars().all()
```

`Book.id.in_(id_list)` 对应 SQL 的 `IN` 查询，可以理解为：

```sql
WHERE id IN (1, 3, 5, 7)
```

查询可能返回多条记录，所以使用 `result.scalars().all()` 提取所有 ORM 对象。

### 当前代码情况

目前 ID 集合固定写在函数内部。实际接口中通常会由客户端传入查询参数，再由 FastAPI 完成类型转换与校验。

---

## 知识点四：聚合查询

聚合函数用于对多条数据进行统计计算。今天的手记中记录了：

- `func.count()`：统计数量；
- `func.max()`：求最大值；
- `func.min()`：求最小值；
- `func.sum()`：求和；
- `func.avg()`：求平均值。

今天实际实现的是图书数量统计：

```python
@app.get("/book/count")
async def get_count(
    db: AsyncSession = Depends(get_database),
):
    result = await db.execute(
        select(func.count(Book.id))
    )
    num = result.scalar()
    return num
```

这里查询结果只有一个统计值，所以使用 `result.scalar()` 提取单个数值，而不是使用 `scalars().all()` 获取对象列表。

### 当前掌握范围

目前只有 `count()` 真正运行在接口中，其他聚合函数属于已经了解但尚未实际练习的内容。

---

## 知识点五：分页查询

分页的两个核心量：

- `offset`：跳过多少条记录；
- `limit`：最多返回多少条记录。

当前接口：

```python
@app.get("/book/get_books")
async def get_book_list(
    page: int = 2,
    page_size: int = 3,
    db: AsyncSession = Depends(get_database),
):
    skip = (page - 1) * page_size

    result = await db.execute(
        select(Book).offset(skip).limit(page_size)
    )
    books = result.scalars().all()
    return books
```

页码与跳过数量的关系：

```text
skip = (page - 1) × page_size
```

假设每页 3 条：

| 页码 | skip | 返回范围 |
| ---: | ---: | --- |
| 1 | 0 | 从第1条开始 |
| 2 | 3 | 跳过前3条 |
| 3 | 6 | 跳过前6条 |

### 当前代码需要改进

1. `page` 默认值目前为 2，常规分页一般默认从第 1 页开始。
2. 没有限制 `page >= 1`，传入 0 或负数会得到错误的 `skip`。
3. 没有限制 `page_size` 的最小值和最大值。
4. 查询没有指定 `order_by()`，数据库返回顺序不一定稳定，可能导致翻页时出现重复或遗漏。

后续可以使用 `Query` 增加校验，并按主键排序：

```python
page: int = Query(1, ge=1)
page_size: int = Query(10, ge=1, le=100)
```

查询语句可以加入：

```python
select(Book).order_by(Book.id).offset(skip).limit(page_size)
```

---

## 知识点六：使用 Pydantic 接收图书数据

新增图书时，客户端需要在请求体中提交书名、作者、价格和出版社。

```python
class BookBase(BaseModel):
    bookname: str
    author: str
    price: float
    publisher: str
```

`BookBase` 是请求数据模型，作用包括：

- 接收客户端提交的 JSON；
- 检查字段是否齐全；
- 验证字段的数据类型；
- 将 JSON 转换为 Python 对象；
- 在接口文档中展示请求结构。

它与 SQLAlchemy 的 `Book` 模型职责不同：

| 模型 | 主要职责 |
| --- | --- |
| `BookBase` | 定义和校验接口请求数据 |
| `Book` | 映射并操作数据库中的 `book` 表 |

Pydantic 模型服务于 API 数据交换，ORM 模型服务于数据库持久化。两者字段可能相似，但不能把它们当作同一个模型。

---

## 知识点七：新增图书

新增操作的核心步骤：

```text
接收请求体
→ 创建 ORM 对象
→ 将对象加入 Session
→ 提交事务
```

当前实现：

```python
@app.post("/book/add_book")
async def add_book(
    book: BookBase,
    db: AsyncSession = Depends(get_database),
):
    book_obj = Book(**book.__dict__)
    db.add(book_obj)
    return book_obj
```

### 请求数据转换为 ORM 对象

`Book(**book.__dict__)` 会把 Pydantic 对象中的字段展开，传给 ORM 模型构造函数。

当前写法能够帮助理解数据转换。在 Pydantic v2 中，更推荐使用公开方法：

```python
book_obj = Book(**book.model_dump())
```

相比直接访问 `__dict__`，`model_dump()` 的意图更明确，也更符合 Pydantic 提供的接口。

### 为什么 `db.add()` 不使用 `await`

`db.add(book_obj)` 只是把对象加入当前 Session，由 Session 跟踪这条待新增数据，本身不会立即执行异步数据库 I/O，因此不需要 `await`。

事务提交仍由 Day 2 定义的 `get_database()` 依赖在路由执行结束后完成。如果后续需要在返回前获得数据库生成的 ID 或时间字段，可以继续学习 `flush()` 和 `refresh()`。

---

## 知识点八：更新图书

更新操作采用“先查找，再赋值”的流程：

```text
接收图书 ID 和请求体
→ 按主键查询 ORM 对象
→ 不存在则抛出 404
→ 修改对象属性
→ 提交事务
```

请求模型：

```python
class BookUpdate(BaseModel):
    bookname: str
    author: str
    price: float
    publisher: str
```

路由中的关键逻辑：

```python
db_book = await db.get(Book, book_id)

if db_book is None:
    raise HTTPException(
        status_code=404,
        detail="Book not found",
    )

db_book.bookname = data.bookname
db_book.author = data.author
db_book.price = data.price
db_book.publisher = data.publisher
```

被查询出来的 `db_book` 已经由 Session 管理。修改它的属性后，SQLAlchemy 会跟踪变化，事务提交时生成对应的 `UPDATE` SQL，因此不需要再次调用 `db.add()`。

### PUT 与当前模型

当前接口使用 `PUT`，而 `BookUpdate` 的四个字段全部必填，因此它表达的是整体更新。如果之后希望只修改价格或出版社，可以再学习可选字段与 `PATCH` 局部更新。

---

## 知识点九：删除图书

删除操作同样需要先确认数据是否存在：

```text
接收图书 ID
→ 按主键查询
→ 不存在则抛出 404
→ 将 ORM 对象标记为删除
→ 提交事务
```

关键代码：

```python
db_book = await db.get(Book, book_id)

if db_book is None:
    raise HTTPException(
        status_code=404,
        detail="Book not found",
    )

await db.delete(db_book)
return {"message": "Book deleted"}
```

与 `db.add()` 不同，异步 Session 的 `delete()` 使用 `await`。真正的删除结果会在事务提交后写入数据库。

更新与删除接口都使用了 Day 1 学习的 `HTTPException`，说明之前的异常处理知识已经开始与数据库操作结合。

---

## 今日 CRUD 流程

| 操作 | HTTP方法 | 核心Session操作 | 数据来源 |
| --- | --- | --- | --- |
| 查询 | `GET` | `execute()`、`get()` | 路径/查询参数 |
| 新增 | `POST` | `add()` | 请求体 |
| 更新 | `PUT` | 查询后修改对象属性 | 路径参数 + 请求体 |
| 删除 | `DELETE` | `delete()` | 路径参数 |

今天最重要的变化是：Day 2 的 Session 不再只执行 `SELECT`，而是开始作为一个工作单元跟踪新增、修改和删除，并在依赖项结束时统一提交事务。

---

## 今日实践与发现

今天已经完成：

- 两种 `LIKE` 模式的编写；
- AND、OR组合条件的编写；
- 使用 `IN` 查询一组图书；
- 统计图书总数；
- 根据页码和每页数量查询；
- 新增、整体更新和删除图书；
- 在更新和删除时处理“图书不存在”。

当前代码仍有以下待完善项：

- 模糊查询和 ID 集合仍固定写在函数内部，尚未改为客户端参数；
- `max`、`min`、`sum`、`avg` 只记录了概念，尚未实际运行；
- 分页缺少参数校验和稳定排序；
- 新增接口尚未处理返回数据库生成字段的问题；
- 新增和更新请求模型尚未增加字段范围校验；
- 返回数据尚未统一使用响应模型；
- Day 2 遗留的数据库连接配置问题仍需处理。

---

## 掌握情况

| 知识点 | 当前状态 | 下一步验收方式 |
| --- | --- | --- |
| `LIKE`模糊查询 | 初步使用 | 将作者模式改为查询参数 |
| AND、OR组合条件 | 初步使用 | 根据多个可选条件动态筛选 |
| `IN`查询 | 已完成基础使用 | 从客户端接收多个 ID |
| 聚合查询 | 初步了解 | 分别计算最高价和平均价 |
| 分页查询 | 已完成基础流程 | 加入校验、排序和总数 |
| Pydantic请求模型 | 已完成基础使用 | 增加价格与字符串长度校验 |
| 新增操作 | 初步使用 | 创建后返回完整数据库记录 |
| 更新操作 | 已完成基础流程 | 独立重写并测试不存在情况 |
| 删除操作 | 已完成基础流程 | 验证提交后记录确实消失 |
| CRUD整体关系 | 需要巩固 | 脱离课程重新实现一套接口 |

今天的内容已经从单一知识点进入完整业务操作阶段。是否真正掌握，主要看能否关掉视频后重新实现，而不是接口是否曾经运行成功。

---

## 容易遗忘或混淆的内容

- `%` 匹配任意长度字符，`_` 只匹配一个字符；
- 使用 `&` 和 `|` 组合表达式时，单个条件最好加括号；
- `in_()` 对应 SQL 的 `IN`；
- 聚合结果是数值，可以使用 `scalar()` 提取；
- `offset` 是跳过数量，不是页码；
- 分页查询必须有稳定排序，才能避免跨页数据不稳定；
- Pydantic 模型负责 API 校验，ORM 模型负责数据库映射；
- Pydantic v2优先使用 `model_dump()` 导出数据；
- `db.add()` 不需要 `await`，`AsyncSession.delete()` 需要 `await`；
- 查询出的 ORM 对象被 Session 跟踪，修改属性后无需再次 `add()`；
- 当前事务由依赖项统一提交，路由中没有直接写 `commit()`；
- `PUT` 更偏向整体更新，`PATCH`更偏向局部更新；
- 更新和删除之前需要处理目标数据不存在的情况。

---

## 今日总结

今天在 Day 2 建立的异步 ORM 基础上完成了两个扩展：一是查询能力从简单条件提升到模糊、组合、包含、聚合和分页；二是数据操作从只读查询扩展到新增、更新和删除。

目前图书接口已经具备 CRUD 雏形，但仍属于课程练习代码。下一步应重点补充参数校验、稳定分页、响应模型和更合理的接口路径，并尝试脱离视频独立重写。完成这一步后，FastAPI 与 SQLAlchemy 的基础数据操作才算形成闭环。

---

## 复习检查

合上代码后尝试回答：

1. `LIKE` 中 `%` 和 `_` 有什么区别？
2. SQLAlchemy中如何表示 AND、OR和 IN 条件？
3. 为什么组合查询条件时需要注意括号？
4. 聚合查询为什么使用 `scalar()` 提取结果？
5. 第 4 页、每页 10 条时，`offset` 应该是多少？
6. 为什么分页查询最好添加 `order_by()`？
7. Pydantic请求模型与 SQLAlchemy ORM 模型有什么区别？
8. 如何把 Pydantic v2 对象转换为创建 ORM 对象所需的字典？
9. 为什么 `db.add()` 不需要 `await`？
10. 为什么更新查询得到的 ORM 对象后不需要再次调用 `add()`？
11. 当前代码中的新增、更新和删除在什么位置提交事务？
12. PUT整体更新和 PATCH局部更新有什么区别？
13. 更新或删除不存在的数据时，为什么应该返回 404？

如果能够不看代码回答第 5、7、10、11 题，并独立完成下一步练习，今天的内容就基本达到了可以应用的程度。

---

## 下一步

优先完善当前图书接口：

1. 将分页默认页改为第 1 页，并用 `Query` 限制页码和每页数量；
2. 为分页查询加入 `order_by(Book.id)`；
3. 返回图书列表的同时提供总数量；
4. 将最低价格、作者关键字和 ID 列表改为客户端可传入的参数；
5. 在 `BookBase` 和 `BookUpdate` 中增加字符串长度与价格范围校验；
6. 使用 `model_dump()` 替换对 `__dict__` 的直接访问；
7. 为新增、更新和查询接口定义响应模型；
8. 脱离课程代码，重新实现一遍图书 CRUD。

这些完善完成后，再考虑拆分项目目录或学习局部更新，避免在基础 CRUD 尚未巩固时继续堆叠新技术。
