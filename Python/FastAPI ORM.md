## FastApi中使用ORM

### 创建模型

安装tortoise

```
pip install tortoise
```

安装数据模型迁移工具

```
pip install aerich==0.6.3
```

如果连接MySQL的话，还需要安装aiomysql

```
pip install aiomysql
```

新建models文件夹（名字不能改），创建模型。

```python
import uvicorn
from tortoise import fields
from tortoise.models import Model


class Student(Model):
    id = fields.IntField(pk=True)
    name = fields.CharField(max_length=255, description="姓名")
    pwd = fields.CharField(max_length=255, description="密码")
    sno = fields.IntField(description="学号")

    # 一对多的关系
    # 外键
    clas = fields.ForeignKeyField("models.Clas", related_name="students")

    # 一对多的关系
    courses = fields.ManyToManyField("models.Course", related_name="students")


# 班级
class Clas(Model):
    id = fields.IntField(pk=True)
    name = fields.CharField(max_length=32, description="班级名称")


# 课程
class Course(Model):
    id = fields.IntField(pk=True)
    name = fields.CharField(max_length=32, description="课程名称")
    teacher = fields.ForeignKeyField("models.Teacher")


# 教师
class Teacher(Model):
    id = fields.IntField(pk=True)
    name = fields.CharField(max_length=32, description="姓名")
    tno = fields.IntField(description="账号")
    pwd = fields.CharField(max_length=32, description="密码")
```

同级目录下新建settings文件，用来放置配置。

```python
TORTOISE_ORM = {
    # 连接的配置
    'connections': {
        'default': {
            # 选择引擎
            'engine': 'tortoise.backends.mysql',  # Mysql or Mariadb
            # 'engine': 'tortoise.backends.asyncpg',  # PostgresSQL
            'credentials': {
                'host': '127.0.0.1',
                'port': '3306',
                'user': 'root',
                'password': '123456',
                'database': 'fastapi',
                'minsize': 1,
                'maxsize': 5,
                'charset': 'utf8mb4',
                'echo': True
            }
        }
    },
    'apps': {
        'models': {
            # 放置自己的models的位置
            # aerich.models 数据模型迁移工具
            'models': ['models', 'aerich.models'],
            'default_connections': 'default'
        }
    },
    'user_tz': False,
    'timezone': 'Asia/Shanghai'
}
```

APP中导入models配置

```python
import uvicorn
from fastapi import FastAPI
from tortoise.contrib.fastapi import register_tortoise
from settings import TORTOISE_ORM

app = FastAPI()


# fastapi一旦运行，这个方法就已经执行，实现监控
register_tortoise(app=app, config=TORTOISE_ORM)

if __name__ == '__main__':
    uvicorn.run('main:app', host='127.0.0.1', port=8010, reload=True, workers=1)
```

### 数据库迁移操作

#### 初始化配置项

首先进入settings目录下，即TORTOISE_ORM配置位置。

```python
aerich init settings.TORTOISE_ORM
```

初始化配置只需要使用一次就可以。

初始化完成后会在当前目录生成一个文件pyproject.toml和一个文件夹migrations。

pyproject.toml	保存配置文件路径；

migrations	存放迁移文件。

#### 初始化数据库

