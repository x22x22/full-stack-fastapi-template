# MySQL 数据库支持可行性分析报告

## 一、执行摘要

本项目 **完全可以支持接入 MySQL 数据库**。项目使用的技术栈（SQLModel/SQLAlchemy）天然支持多种数据库后端，包括 MySQL。但需要进行一些配置调整和代码修改。

## 二、当前技术栈分析

### 2.1 数据库相关组件

| 组件 | 当前配置 | MySQL 兼容性 |
|------|---------|-------------|
| **数据库** | PostgreSQL 17 | ✅ 可替换为 MySQL |
| **ORM** | SQLModel 0.0.21 | ✅ 完全支持 MySQL |
| **数据库驱动** | psycopg 3.1.13 | ⚠️ 需要更换为 MySQL 驱动 |
| **迁移工具** | Alembic 1.12.1 | ✅ 完全支持 MySQL |
| **连接 URL** | PostgresDsn | ⚠️ 需要更改为 MySQLDsn |

### 2.2 SQLModel/SQLAlchemy 数据库兼容性

SQLModel 基于 SQLAlchemy，天然支持以下数据库：
- ✅ PostgreSQL
- ✅ MySQL / MariaDB
- ✅ SQLite
- ✅ Oracle
- ✅ Microsoft SQL Server

## 三、PostgreSQL 特定功能识别

### 3.1 代码中的 PostgreSQL 依赖

#### 位置 1: `backend/app/core/config.py` (第61-69行)
```python
@computed_field
@property
def SQLALCHEMY_DATABASE_URI(self) -> PostgresDsn:
    return PostgresDsn.build(
        scheme="postgresql+psycopg",
        username=self.POSTGRES_USER,
        password=self.POSTGRES_PASSWORD,
        host=self.POSTGRES_SERVER,
        port=self.POSTGRES_PORT,
        path=self.POSTGRES_DB,
    )
```
**影响**: 硬编码使用 PostgreSQL DSN 和 psycopg 驱动

#### 位置 2: `backend/app/alembic/versions/d98dd8ec85a3_*.py` (第11, 23-28行)
```python
from sqlalchemy.dialects import postgresql

# 在迁移中使用
op.execute('CREATE EXTENSION IF NOT EXISTS "uuid-ossp"')
op.add_column('user', sa.Column('new_id', postgresql.UUID(as_uuid=True), ...))
```
**影响**: 使用 PostgreSQL 特定的 UUID 扩展和类型

### 3.2 Docker Compose 配置

#### 位置: `docker-compose.yml` (第3-20行)
```yaml
db:
  image: postgres:17
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
```
**影响**: 容器镜像和健康检查命令特定于 PostgreSQL

## 四、MySQL 支持实施方案

### 4.1 必需修改项

#### 修改 1: 更新依赖包 (`backend/pyproject.toml`)

**当前配置**:
```toml
"psycopg[binary]<4.0.0,>=3.1.13",
```

**MySQL 配置** (选择其一):
```toml
# 选项 1: mysqlclient (推荐，性能最佳)
"mysqlclient>=2.2.0",

# 选项 2: PyMySQL (纯 Python 实现)
"pymysql>=1.1.0",

# 选项 3: asyncmy (异步支持)
"asyncmy>=0.2.9",
```

#### 修改 2: 数据库配置 (`backend/app/core/config.py`)

**需要添加**:
```python
from pydantic import MySQLDsn
import os

class Settings(BaseSettings):
    # 添加数据库类型选择
    DATABASE_TYPE: Literal["postgresql", "mysql"] = "postgresql"
    
    # MySQL 配置参数
    MYSQL_SERVER: str | None = None
    MYSQL_PORT: int = 3306
    MYSQL_USER: str | None = None
    MYSQL_PASSWORD: str | None = None
    MYSQL_DB: str | None = None
    
    # 保留 PostgreSQL 配置
    POSTGRES_SERVER: str
    POSTGRES_PORT: int = 5432
    POSTGRES_USER: str
    POSTGRES_PASSWORD: str = ""
    POSTGRES_DB: str = ""
    
    @computed_field
    @property
    def SQLALCHEMY_DATABASE_URI(self) -> PostgresDsn | MySQLDsn:
        if self.DATABASE_TYPE == "mysql":
            return MySQLDsn.build(
                scheme="mysql+mysqlclient",  # 或 mysql+pymysql
                username=self.MYSQL_USER,
                password=self.MYSQL_PASSWORD,
                host=self.MYSQL_SERVER,
                port=self.MYSQL_PORT,
                path=self.MYSQL_DB,
            )
        else:
            return PostgresDsn.build(
                scheme="postgresql+psycopg",
                username=self.POSTGRES_USER,
                password=self.POSTGRES_PASSWORD,
                host=self.POSTGRES_SERVER,
                port=self.POSTGRES_PORT,
                path=self.POSTGRES_DB,
            )
```

#### 修改 3: UUID 处理策略

MySQL 5.7+ 没有原生 UUID 类型，需要使用以下策略之一：

**策略 A: 使用 CHAR(36) 存储 UUID 字符串**
```python
# 在 models.py 中
import uuid
from sqlmodel import Field

class User(UserBase, table=True):
    # MySQL: 存储为 CHAR(36)
    # PostgreSQL: 存储为 UUID
    id: uuid.UUID = Field(
        default_factory=uuid.uuid4,
        primary_key=True,
        sa_type=sqlalchemy.types.String(36)  # 对于 MySQL
    )
```

**策略 B: 使用 BINARY(16) 存储 UUID 二进制** (性能更好)
```python
from sqlalchemy.dialects.mysql import BINARY
from sqlalchemy import TypeDecorator

class GUID(TypeDecorator):
    impl = BINARY(16)
    cache_ok = True
    
    def process_bind_param(self, value, dialect):
        if value is None:
            return value
        return value.bytes
    
    def process_result_value(self, value, dialect):
        if value is None:
            return value
        return uuid.UUID(bytes=value)
```

#### 修改 4: Alembic 迁移文件

需要修改或创建 MySQL 兼容的迁移文件：

```python
# 替代 PostgreSQL UUID 扩展
def upgrade():
    # PostgreSQL
    if op.get_bind().dialect.name == 'postgresql':
        op.execute('CREATE EXTENSION IF NOT EXISTS "uuid-ossp"')
        op.add_column('user', sa.Column('id', postgresql.UUID(as_uuid=True), ...))
    # MySQL
    elif op.get_bind().dialect.name == 'mysql':
        op.add_column('user', sa.Column('id', sa.String(36), ...))
```

#### 修改 5: Docker Compose 配置

**创建新文件**: `docker-compose.mysql.yml`

```yaml
services:
  db:
    image: mysql:8.0
    restart: always
    command: --default-authentication-plugin=mysql_native_password
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u${MYSQL_USER}", "-p${MYSQL_PASSWORD}"]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 10s
    volumes:
      - app-db-data:/var/lib/mysql
    env_file:
      - .env
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DB}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}

  adminer:
    # Adminer 同时支持 PostgreSQL 和 MySQL，无需修改
    image: adminer
    # ... 保持不变
```

#### 修改 6: 环境变量 (`.env`)

**添加 MySQL 配置**:
```env
# 数据库类型选择
DATABASE_TYPE=mysql  # 或 postgresql

# MySQL 配置
MYSQL_SERVER=localhost
MYSQL_PORT=3306
MYSQL_DB=app
MYSQL_USER=root
MYSQL_PASSWORD=changethis
MYSQL_ROOT_PASSWORD=changethis
```

### 4.2 可选修改项

#### 数据类型兼容性调整

| 场景 | PostgreSQL | MySQL 对应 |
|------|-----------|-----------|
| 文本字段 | TEXT | TEXT 或 LONGTEXT |
| JSON 字段 | JSONB | JSON (MySQL 5.7+) |
| 数组字段 | ARRAY | JSON 或建立关联表 |
| 全文搜索 | tsvector | FULLTEXT INDEX |

如果代码中使用了这些 PostgreSQL 特有类型，需要进行相应调整。

## 五、实施步骤建议

### 阶段 1: 准备工作（1-2天）
1. ✅ 完成可行性分析（本文档）
2. 备份现有数据库
3. 在开发环境测试 MySQL 连接

### 阶段 2: 核心修改（2-3天）
1. 修改 `pyproject.toml` 添加 MySQL 驱动
2. 更新 `backend/app/core/config.py` 支持多数据库
3. 处理 UUID 类型兼容性
4. 创建 MySQL 专用 Docker Compose 配置

### 阶段 3: 迁移调整（1-2天）
1. 审查所有 Alembic 迁移文件
2. 修改 PostgreSQL 特定代码使其数据库无关
3. 创建新的 MySQL 迁移或条件迁移

### 阶段 4: 测试验证（2-3天）
1. 在 MySQL 环境运行全部测试套件
2. 验证数据迁移正确性
3. 性能基准测试
4. 文档更新

### 阶段 5: 部署（1天）
1. 更新部署文档
2. 提供数据库切换指南
3. 生产环境迁移计划

**预计总时间**: 7-11 个工作日

## 六、风险评估

### 高风险项
- ⚠️ **UUID 类型不兼容**: 需要仔细处理，可能影响现有数据
- ⚠️ **现有迁移文件**: PostgreSQL 特定语法需要重写

### 中风险项
- ⚠️ **性能差异**: MySQL 和 PostgreSQL 在某些查询上性能特征不同
- ⚠️ **全文搜索**: 如果使用了 PostgreSQL 特定的全文搜索功能

### 低风险项
- ✅ **基本 CRUD 操作**: SQLModel 完全兼容
- ✅ **Alembic 迁移工具**: 天然支持多数据库
- ✅ **应用层代码**: 大部分代码无需修改

## 七、推荐方案

### 方案 A: 完全切换到 MySQL
**适用场景**: 决定统一使用 MySQL
- **优点**: 配置简单，维护成本低
- **缺点**: 放弃 PostgreSQL 的高级功能

### 方案 B: 同时支持两种数据库（推荐）
**适用场景**: 需要灵活性，用户可自行选择
- **优点**: 最大灵活性，通过环境变量切换
- **缺点**: 维护成本较高，需要确保两种数据库下都能正常工作

### 方案 C: 保持 PostgreSQL，提供 MySQL 分支
**适用场景**: 主要使用 PostgreSQL，MySQL 作为备选
- **优点**: 主线代码保持简洁
- **缺点**: 需要维护两个分支

## 八、技术债务考虑

如果选择支持 MySQL，建议：
1. 避免使用数据库特定功能
2. 在 CI/CD 中同时测试两种数据库
3. 使用 SQLAlchemy 的跨数据库类型
4. 文档明确说明各数据库的差异

## 九、结论

**答案：是的，本项目完全可以支持接入 MySQL 数据库。**

主要原因：
1. ✅ SQLModel/SQLAlchemy 天然支持 MySQL
2. ✅ Alembic 完全兼容 MySQL 迁移
3. ✅ 应用层代码使用 ORM，数据库无关
4. ⚠️ 需要处理少量 PostgreSQL 特定功能（主要是 UUID）
5. ⚠️ 需要更新配置文件和部署脚本

**实施难度**: 中等
**预计工作量**: 7-11 个工作日
**风险等级**: 中低（大部分是配置修改，代码改动较少）

建议采用**方案 B（同时支持两种数据库）**，通过环境变量 `DATABASE_TYPE` 控制，既保持了向后兼容，又提供了灵活性。
