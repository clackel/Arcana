# 数据库配置

### 创建异步引擎

```python
async_engine = create_async_engine(
    ASYNC_DATABASE_URL, # 数据库地址
    echo=True, # 输出SQ日志
    pool_size = 10,
    max_overflow = 20,
)
```

### 创建异步会话工厂

```python
AsyncSessionLocal = async_sessionmaker(
    bind = async_engine,
    class_ = AsyncSession,
    expire_on_commit = False,
)
```
