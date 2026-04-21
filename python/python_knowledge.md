# Python 知识点总结

## 1. 包别名机制

```python
sys.modules[__name__] = importlib.import_module("kimi_cli")
```

这行代码让 `kimi_code` 成为 `kimi_cli` 的别名。

当用户执行 `import kimi_code` 时，Python 实际导入的是 `kimi_cli` 模块。

**使用场景：**
- 项目重命名后保留旧包名以兼容已有代码
- 提供更简短或更直观的包名别名

---

## 2. importlib 模块

Python 标准库，提供程序化导入模块的能力。

```python
import importlib

# 动态导入
mod = importlib.import_module("json")  # 等价于 import json

# 导入子模块
path = importlib.import_module("os.path")

# 重新加载模块
importlib.reload(my_module)
```

**与 `import` 语句的区别：**

| 方式 | 特点 |
|------|------|
| `import json` | 静态导入，模块名必须硬编码 |
| `importlib.import_module("json")` | 动态导入，模块名可以是字符串变量 |

---

## 3. @classmethod 装饰器

将方法绑定到**类**而非实例，第一个参数是类本身（约定叫 `cls`）。

```python
class Person:
    count = 0

    def __init__(self, name):
        self.name = name
        Person.count += 1

    # 普通方法 — 第一个参数是 self（实例）
    def say_hi(self):
        print(f"Hi, I'm {self.name}")

    # 类方法 — 第一个参数是 cls（类）
    @classmethod
    def get_count(cls):
        return cls.count

    # 常见用途：工厂方法
    @classmethod
    def from_string(cls, name_str):
        return cls(name_str)
```

**常见用途：**

| 用途 | 示例 |
|------|------|
| 工厂方法 | `dict.fromkeys()`、`datetime.now()` |
| 访问/修改类属性 | 跟踪实例数量、共享状态 |
| 继承友好 | 子类调用时 `cls` 指向子类 |

---

## 4. @staticmethod 装饰器

将方法定义为静态方法，不需要实例化就能调用，不访问 `self` 或 `cls`。

```python
class MathUtils:
    @staticmethod
    def add(a, b):
        return a + b

    @staticmethod
    def multiply(a, b):
        return a * b

# 直接用类名调用
result = MathUtils.add(1, 2)  # 3

# 也可以实例化后调用
utils = MathUtils()
utils.multiply(2, 3)  # 6
```

### 三种方法对比

| 装饰器 | 第一个参数 | 调用方式 | 用途 |
|--------|-----------|---------|------|
| 普通方法 | `self` (实例) | `obj.method()` | 操作实例数据 |
| `@staticmethod` | 无 | `Class.method()` | 工具函数 |
| `@classmethod` | `cls` (类本身) | `Class.method()` | 工厂方法、类变量 |

### 适用场景

- 工具函数（与类相关但不需要实例状态）
- 作为命名空间组织相关函数
- 不需要访问实例或类状态的函数

---

## 5. Typer 模块

用于构建 CLI 应用的 Python 库，基于 `click` 构建，使用类型注解自动生成命令行参数。

```python
import typer

app = typer.Typer()

@app.command()
def hello(name: str):
    print(f"Hello {name}")

@app.command()
def goodbye(name: str, formal: bool = False):
    if formal:
        print(f"Goodbye Ms. {name}.")
    else:
        print(f"Bye {name}!")

if __name__ == "__main__":
    app()
```

**类型注解映射：**

```python
def process(
    name: str,                    # 必需字符串参数
    count: int = 1,               # 可选 int，默认 1
    verbose: bool = False,        # --verbose 标志
    path: Path = ".",             # 路径参数
    mode: Literal["a", "b"] = "a" # 枚举选择
):
    ...
```

---

## 5. @app.callback(invoke_without_command=True)

定义在所有命令之前执行的回调函数，用于处理全局选项。

```python
@app.callback(invoke_without_command=True)
def main(version: bool = typer.Option(False, "--version", "-v")):
    if version:
        print("MyApp v1.0.0")
        raise typer.Exit()
```

**invoke_without_command 参数：**

| 调用方式 | `True` | `False`（默认） |
|---------|--------|----------------|
| `myapp --version` | ✅ 执行 callback | ❌ 报错：缺少命令 |
| `myapp run` | ✅ 先 callback 再 run | ✅ 先 callback 再 run |
| `myapp` | ✅ 执行 callback | ❌ 报错或显示帮助 |

---

## 6. async/await 语法

Python 异步编程语法，用于编写并发代码。

```python
import asyncio

# 定义协程
async def fetch_data():
    await asyncio.sleep(1)
    return "结果"

# 运行协程
asyncio.run(fetch_data())

# 并发执行
async def main():
    results = await asyncio.gather(
        download("a.com"),
        download("b.com"),
        download("c.com"),
    )
```

**关键点：**

| 概念 | 说明 |
|------|------|
| `async def` | 定义协程函数 |
| `await` | 等待协程完成，期间让出控制权 |
| `asyncio.run()` | 运行协程的入口点 |
| `asyncio.gather()` | 并发运行多个协程 |

**适合场景：** 网络请求、文件 I/O、大量等待时间的任务

---

## 7. @dataclass(frozen=True, slots=True)

### frozen=True

创建不可变对象：

```python
@dataclass(frozen=True)
class Point:
    x: int
    y: int

p = Point(1, 2)
p.x = 10  # ❌ FrozenInstanceError
```

- 实例不可修改
- 可哈希，能用作字典键或集合元素

### slots=True

使用 `__slots__` 优化（Python 3.10+）：

```python
@dataclass(slots=True)
class Point:
    x: int
    y: int

p = Point(1, 2)
p.z = 3  # ❌ AttributeError
```

| 特性 | 无 slots | 有 slots |
|------|---------|---------|
| 内存占用 | 较大 | 较小 |
| 属性访问 | 稍慢 | 更快 |
| 动态属性 | ✅ 可添加 | ❌ 禁止 |

### kw_only=True

强制所有字段只能通过关键字参数传入（Python 3.10+）：

```python
@dataclass(kw_only=True)
class Config:
    host: str
    port: int = 8080

# ✅ 正确：使用关键字参数
c = Config(host="localhost", port=9000)

# ❌ 错误：位置参数被禁止
c = Config("localhost")  # TypeError
```

**解决默认值顺序问题：**

```python
# 没有 kw_only — 默认值字段必须在后面
@dataclass
class Bad:
    name: str           # 无默认值
    age: int = 18       # 有默认值
    score: int          # ❌ 错误：无默认值字段不能在有默认值之后

# 使用 kw_only — 顺序随意
@dataclass(kw_only=True)
class Good:
    name: str           # 无默认值
    age: int = 18       # 有默认值
    score: int          # ✅ 可以：关键字参数顺序不限
```

**组合使用场景：** 配置类、值对象、不可变数据结构

---

## 8. @property 装饰器

将方法变成**只读属性**，访问时像访问字段一样，无需调用括号。

### 基本用法

```python
class Circle:
    def __init__(self, radius):
        self.radius = radius

    @property
    def area(self):
        return 3.14 * self.radius ** 2

c = Circle(5)
print(c.area)    # 78.5  ← 像访问属性，无需 c.area()
# c.area = 100   # ❌ 报错：不可赋值
```

### getter/setter/deleter

```python
class Person:
    def __init__(self, name):
        self._name = name

    @property
    def name(self):
        """getter"""
        return self._name.upper()

    @name.setter
    def name(self, value):
        """setter"""
        if len(value) < 2:
            raise ValueError("名字太短")
        self._name = value

    @name.deleter
    def name(self):
        """deleter"""
        del self._name

p = Person("Alice")
print(p.name)     # ALICE（getter 生效）
p.name = "Bob"    # setter 生效
del p.name        # deleter 生效
```

### 优势

| 用途 | 说明 |
|------|------|
| **封装** | 隐藏内部实现，只暴露接口 |
| **验证** | setter 中添加校验逻辑 |
| **计算属性** | 动态计算，如 `area` |
| **向后兼容** | 字段改成方法无需修改调用方 |

### 对比普通方法

```python
# 普通方法
c.area()      # 需要括号

# @property
c.area        # 直接访问，更简洁
```

---

## 9. @override 装饰器

Python 3.12 新增的类型提示装饰器，用于**显式标记方法重写父类方法**。

### 基本用法

```python
from typing import override

class Animal:
    def speak(self) -> str:
        return "..."

class Dog(Animal):
    @override
    def speak(self) -> str:
        return "Woof"
```

### 作用

| 功能 | 说明 |
|------|------|
| **类型检查** | 静态分析工具验证父类是否真的有该方法 |
| **代码可读性** | 明确表明这是重写，不是新方法 |
| **防止错误** | 父类没有该方法时，IDE/检查器会警告 |

### 错误示例

```python
class Dog(Animal):
    @override
    def speek(self):  # ❌ 拼写错误，父类没有 speek
        return "Woof"
```

### 与其他语言的对比

| 语言 | 关键字/装饰器 |
|------|--------------|
| Java | `@Override` |
| C# | `override` |
| Kotlin | `override` |
| Python 3.12+ | `@override` |

### 注意

- Python 3.12+ 才内置支持
- 早期版本可用 `typing_extensions.override`
- 纯类型提示，运行时不强制检查