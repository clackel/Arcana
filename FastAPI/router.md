# Router

### 创建应用实例

```python
from fastapi import FastAPI
app = FastAPI()
```

### 创建路由

```python
@app.get("/")
async def root():
    return
```
