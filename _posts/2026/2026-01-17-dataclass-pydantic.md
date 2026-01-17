---
layout: blog
istop: false
title: "dataclass 与 Pydantic"
category: 知识归纳
background: red
background-image: https://s2.loli.net/2026/01/17/MgXCK7oerT5GJ8a.png
tags:
- dataclass
- pydantic
- python
---

# 前言

python中，常需要定义一些“数据容器”——主要用来存储数据，而不是实现复杂的业务逻辑。比如配置文件、API请求参数等。传统做法时定义一个类，然后写一堆重复的代码：

```python
# 传统方式：繁琐且容易出错
class Person:
    def __init__(self, name, age, email):
        self.name = name
        self.age = age
        self.email = email
    
    def __repr__(self):
        return f"Person(name={self.name}, age={self.age}, email={self.email})"
    
    def __eq__(self, other):
        if not isinstance(other, Person):
            return False
        return self.name == other.name and self.age == other.age and self.email == other.email
```

为了解决这些问题，Python社区提供了两个优秀的解决方案：**dataclass**（标准库）和 **Pydantic**（第三方库）。其核心功能对比如下：

| 功能         | dataclass                                   | Pydantic             |
| ------------ | ------------------------------------------- | -------------------- |
| 自动生成方法 | ✅ \_\_init\_\_, \_\_repr\_\_, \_\_eq\_\_ 等 | ✅ 同左               |
| 类型注解     | ✅ 仅作为提示                                | ✅ 运行时验证         |
| 数据验证     | ❌ 无                                        | ✅ 强大的验证器       |
| 类型转换     | ❌ 无                                        | ✅ 自动转换           |
| JSON序列化   | ⚠️ 需手动实现                                | ✅ 内置支持           |
| 性能         | ⚡ 快（无验证开销）                          | 🐢 较慢（有验证开销） |
| 依赖         | ✅ 标准库                                    | ⚠️ 需安装第三方库     |
| 适用场景     | 内部数据结构                                | 配置、API、外部数据  |

# dataclass

## 基础用法

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

p1 = Point(3, 4)
p2 = Point(3, 4)

print(p1)  # 输出: Point(x=3, y=4)
print(p1 == p2)  # 输出: True
```

## dataclass 的默认行为

- \_\_init\_\_ 方法：自动生成并根据类的属性定义初始化方法。
- \_\_repr\_\_ 方法：自动生成一个可读性很强的字符串表示，用于打印对象。
- \_\_eq\_\_ 方法：自动生成比较方法，使得数据类的实例可以通过 == 进行比较。
- \_\_hash\_\_ 方法：如果所有字段都是不可变的，自动生成一个 \_\_hash\_\_ 方法，使对象可用于集合等需要哈希值的场景。

## `field` 函数

`dataclass` 提供了 `field` 函数，可以定制每个字段的行为。例如设置默认值、指定字段是否参与比较、定义字段的默认工厂函数等。

```py
from dataclasses import dataclass, field

@dataclass
class Product:
    name: str
    price: float
    tags: list[str] = field(default_factory=list)  # 使用默认工厂创建空列表,确保每个 Product 实例都有一个独立的空列表作为默认值，而不是共享同一个列表对象。

    def apply_discount(self, percentage: float):
        self.price *= (1 - percentage)

# 创建产品实例
product = Product(name="Laptop", price=1000.0)
product.apply_discount(0.1)

print(product)  # 输出: Product(name='Laptop', price=900.0, tags=[])
```

## 高级特性

### `frozen`与`order`

如果希望创建不可修改的实例，可以使用 `frozen=True` 参数将数据类设为“冻结”的，实例将变为不可变对象，不能修改其属性。

```python
from dataclasses import dataclass

@dataclass(frozen=True) # 不可变对象
class Coordinate:
    latitude: float
    longitude: float

coordinate = Coordinate(40.7128, 74.0060)
# coordinate.latitude = 41.0  # 会抛出错误: FrozenInstanceError

@dataclass(order=True)  # 支持比较运算
class Score:
    value: int
    
scores = [Score(85), Score(92), Score(78)]
print(sorted(scores))  # 按 value 排序
```

### \_\_post_init\_\_

dataclasses 提供了一个特殊方法 \_\_post_init\_\_。会在 \_\_init\_\_ 方法执行后自动调用，适用于需要在初始化后执行额外逻辑的场景，例如属性验证、派生属性计算或动态设置默认值。

```py
from dataclasses import dataclass, field

@dataclass
class Config:
    name: str
    timestamp: str = field(init=False)	 # __init__ 方法不会初始化该字段

    def __post_init__(self): 
        from datetime import datetime
        self.timestamp = datetime.now().isoformat()

config = Config(name="test")
print(config.timestamp)  # 自动生成的时间戳
```

### 转换为字典/元组

```python
from dataclasses import dataclass, asdict, astuple

@dataclass
class Book:
    title: str
    author: str
    year: int

book = Book("Python编程", "张三", 2023)

# 转为字典
print(asdict(book))
# {'title': 'Python编程', 'author': '张三', 'year': 2023}

# 转为元组
print(astuple(book))
# ('Python编程', '张三', 2023)
```

# Pydantic

## 安装

```text
pip install pydantic
```

## 基础用法

```python
from pydantic import BaseModel, ValidationError

class Student(BaseModel):
    name: str
    age: int
    grade: float
    is_active: bool = True

# ✅ 正常创建
student = Student(name="张三", age=20, grade=85.5)
print(student)	# name='张三' age=20 grade=85.5 is_active=True

# ✅ 自动类型转换
student2 = Student(name="李四", age="22", grade="90.0")
print(student2.age)  # 22 (int) - 从字符串自动转换
print(type(student2.age))  # <class 'int'>

# ❌ 验证失败
try:
    student = Student(name=123, age="不是数字", grade=85.5)
except ValidationError as e:
    print(e)
```

## 字段约束

| 约束                 | 说明                        | 适用类型        |
| -------------------- | --------------------------- | --------------- |
| gt/ge/lt/le          | 大于/大于等于/小于/小于等于 | int, float      |
| min_lenth/max_length | 最小长度/最大长度           | str, list, dict |
| pattern              | 正则表达式匹配              | str             |
| allow_inf_nan        | 是否允许无穷大或 NaN        | float           |
| EmailStr             | 自动验证邮箱格式            |                 |

```python
from pydantic import BaseModel, Field, field_validator, EmailStr
from typing import Annotated

class User(BaseModel):
    # 字段级验证
    username: str = Field(min_length=3, max_length=20, pattern=r'^[a-zA-Z0-9_]+$')
    age: Annotated[int, Field(gt=0, lt=150)]  # 0 < age < 150
    email: EmailStr  # 自动验证邮箱格式
    score: float = Field(ge=0.0, le=100.0)  # 0 <= score <= 100
    
    # 自定义验证器
    @field_validator('username')
    @classmethod
    def username_must_be_lowercase(cls, v: str) -> str:
        if not v.islower():
            raise ValueError('用户名必须是小写字母')
        return v

# ✅ 验证通过
user = User(username="zhangsan", age=25, email="zhangsan@example.com", score=85.5)

# ❌ 验证失败
try:
    user = User(username="ab", age=200, email="invalid", score=150)
except ValidationError as e:
    print(e)
    # username: 至少3个字符
    # age: 必须小于150
    # email: 邮箱格式错误
    # score: 必须小于等于100
```

## JSON序列化

```python
from pydantic import BaseModel
from pathlib import Path
from datetime import datetime

class Experiment(BaseModel):
    name: str
    data_path: Path
    created_at: datetime
    parameters: dict

exp = Experiment(
    name="实验001",
    data_path="/data/exp001",
    created_at="2024-01-15T10:30:00",
    parameters={"learning_rate": 0.001, "epochs": 100}
)

# 转为字典
print(exp.model_dump())
# {
#     'name': '实验001',
#     'data_path': PosixPath('/data/exp001'),
#     'created_at': datetime.datetime(2024, 1, 15, 10, 30),
#     'parameters': {'learning_rate': 0.001, 'epochs': 100}
# }

# 转为 JSON 字符串
print(exp.model_dump_json(indent=2))
# {
#   "name": "实验001",
#   "data_path": "/data/exp001",
#   "created_at": "2024-01-15T10:30:00",
#   "parameters": {
#     "learning_rate": 0.001,
#     "epochs": 100
#   }
# }

# 从 JSON 加载
json_str = '{"name": "实验002", "data_path": "/data/exp002", "created_at": "2024-01-16T11:00:00", "parameters": {}}'
exp2 = Experiment.model_validate_json(json_str)
print(exp2.name)  # 实验002
```

## 配置选项

```python
from pydantic import BaseModel, ConfigDict

class StrictModel(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,  # 自动去除字符串首尾空格
        validate_assignment=True,   # 赋值时也验证
        frozen=True,                # 不可变对象
        extra='forbid'              # 禁止额外字段
    )
    
    name: str
    age: int

# ✅ 自动去除空格
model = StrictModel(name="  张三  ", age=25)
print(model.name)  # "张三"

# ❌ 不可变
# model.age = 30  # ValidationError

# ❌ 禁止额外字段
try:
    model = StrictModel(name="李四", age=30, extra_field="不允许")
except ValidationError as e:
    print(e)  # Extra inputs are not permitted
```

# 参考

[(Python 数据类（dataclasses）：简化类定义和数据管理 - 知乎](https://zhuanlan.zhihu.com/p/1890503834708714486)

[(Python数据类完全指南：Pydantic vs Dataclass - 知乎](https://zhuanlan.zhihu.com/p/1970136264511562403)