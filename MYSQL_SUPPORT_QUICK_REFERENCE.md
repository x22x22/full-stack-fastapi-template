# MySQL 支持快速参考指南 / MySQL Support Quick Reference

## 快速答案 / Quick Answer

**问：本项目是否可以支持接入 MySQL 数据库？**
**Q: Can this project support MySQL database integration?**

**答：是的，完全可以！/ A: Yes, absolutely!**

---

## 核心要点 / Key Points

### ✅ 兼容性 / Compatibility
- SQLModel (基于 SQLAlchemy) / SQLModel (based on SQLAlchemy) ✅
- Alembic 迁移工具 / Alembic migration tool ✅
- 应用层代码 95% 无需修改 / 95% of application code needs no changes ✅

### ⚠️ 需要修改 / Changes Required
- 数据库驱动：psycopg → mysqlclient/pymysql
- UUID 处理：PostgreSQL UUID → CHAR(36)/BINARY(16)
- 配置文件：支持数据库类型选择
- Docker 配置：MySQL 镜像和健康检查

---

## 最小修改清单 / Minimal Change Checklist

### 1️⃣ 依赖包 / Dependencies
```toml
# backend/pyproject.toml
# 删除 / Remove:
"psycopg[binary]<4.0.0,>=3.1.13"

# 添加 / Add:
"mysqlclient>=2.2.0"  # 推荐 / Recommended
```

### 2️⃣ 配置 / Configuration
```python
# backend/app/core/config.py
from pydantic import MySQLDsn

DATABASE_TYPE: Literal["postgresql", "mysql"] = "mysql"
MYSQL_SERVER: str = "localhost"
MYSQL_PORT: int = 3306
MYSQL_USER: str = "root"
MYSQL_PASSWORD: str = "password"
MYSQL_DB: str = "app"
```

### 3️⃣ 环境变量 / Environment Variables
```env
# .env
DATABASE_TYPE=mysql
MYSQL_SERVER=localhost
MYSQL_PORT=3306
MYSQL_DB=app
MYSQL_USER=root
MYSQL_PASSWORD=changethis
```

### 4️⃣ Docker Compose
```yaml
# docker-compose.mysql.yml
services:
  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DB}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
```

### 5️⃣ UUID 模型 / UUID Models
```python
# backend/app/models.py
import sqlalchemy

class User(UserBase, table=True):
    id: uuid.UUID = Field(
        default_factory=uuid.uuid4,
        primary_key=True,
        sa_type=sqlalchemy.types.String(36)  # MySQL compatible
    )
```

---

## 快速启动 MySQL / Quick Start with MySQL

### 方式 1：新项目 / Method 1: New Project
```bash
# 1. 修改 .env
echo "DATABASE_TYPE=mysql" >> .env
echo "MYSQL_SERVER=localhost" >> .env
echo "MYSQL_PORT=3306" >> .env
echo "MYSQL_DB=app" >> .env
echo "MYSQL_USER=root" >> .env
echo "MYSQL_PASSWORD=changethis" >> .env
echo "MYSQL_ROOT_PASSWORD=changethis" >> .env

# 2. 使用 MySQL docker-compose
docker compose -f docker-compose.yml -f docker-compose.mysql.yml up -d

# 3. 运行迁移
docker compose exec backend alembic upgrade head
```

### 方式 2：现有项目迁移 / Method 2: Migrate Existing Project
```bash
# 1. 导出 PostgreSQL 数据
docker compose exec db pg_dump -U postgres app > backup.sql

# 2. 转换为 MySQL 格式（需要手动调整）
# 使用工具如 pgloader 或手动修改 SQL

# 3. 切换到 MySQL
# ... 按照方式 1 的步骤操作

# 4. 导入数据
docker compose exec db mysql -u root -p app < converted_data.sql
```

---

## 三种实施方案对比 / Three Implementation Approaches

| 特性 / Feature | 方案A: 完全MySQL<br/>Full MySQL | 方案B: 双数据库支持 ⭐<br/>Dual Support ⭐ | 方案C: 分支管理<br/>Branch Approach |
|----------------|------------------------------|----------------------------------|---------------------------|
| **实施难度**<br/>Complexity | 简单 / Simple | 中等 / Medium | 中等 / Medium |
| **灵活性**<br/>Flexibility | 低 / Low | 高 / High ⭐ | 中 / Medium |
| **维护成本**<br/>Maintenance | 低 / Low | 中 / Medium | 高 / High |
| **推荐场景**<br/>Best For | MySQL only | 生产环境<br/>Production ⭐ | 试验<br/>Experimental |

⭐ **推荐方案 B / Recommended: Option B**

---

## 工作量估算 / Effort Estimation

| 阶段 / Phase | 工作内容 / Tasks | 时间 / Time |
|-------------|-----------------|------------|
| 📋 准备 / Prep | 分析、备份、测试<br/>Analysis, backup, testing | 1-2 天 / days |
| 🔧 核心修改 / Core | 依赖、配置、UUID<br/>Dependencies, config, UUID | 2-3 天 / days |
| 🗃️ 迁移 / Migration | Alembic 文件修改<br/>Alembic file updates | 1-2 天 / days |
| ✅ 测试 / Testing | 测试套件、性能<br/>Test suite, performance | 2-3 天 / days |
| 🚀 部署 / Deploy | 文档、上线<br/>Docs, deployment | 1 天 / day |

**总计 / Total**: 7-11 工作日 / working days

---

## 常见问题 / FAQ

### Q1: 会丢失数据吗？/ Will data be lost?
**A**: 不会，只要正确执行迁移和备份。/ No, with proper migration and backups.

### Q2: 性能会受影响吗？/ Will performance be affected?
**A**: 略有不同，但对大多数应用影响不大。/ Slightly different, but minimal impact for most apps.

### Q3: 需要改多少代码？/ How much code needs changing?
**A**: 主要是配置文件，应用代码改动 < 5%。/ Mainly config files, <5% application code.

### Q4: 可以同时支持两种数据库吗？/ Can both databases be supported?
**A**: 可以！通过环境变量切换。/ Yes! Switch via environment variables.

### Q5: UUID 怎么处理？/ How to handle UUIDs?
**A**: MySQL 用 CHAR(36) 或 BINARY(16) 存储。/ MySQL uses CHAR(36) or BINARY(16) storage.

---

## 风险提示 / Risk Warnings

### ⚠️ 高风险 / High Risk
- UUID 类型转换需要仔细处理 / UUID type conversion needs careful handling
- 现有迁移文件需要重写 / Existing migrations need rewriting

### ✅ 低风险 / Low Risk
- 基本 CRUD 操作无影响 / Basic CRUD operations unaffected
- SQLModel 自动处理大部分差异 / SQLModel handles most differences

---

## 技术支持 / Technical Support

### 详细文档 / Detailed Documentation
- 📄 中文完整版 / Chinese Full Version: `MYSQL_SUPPORT_ANALYSIS.md`
- 📄 英文完整版 / English Full Version: `MYSQL_SUPPORT_ANALYSIS_EN.md`

### 相关链接 / Related Links
- [SQLModel 官方文档 / SQLModel Docs](https://sqlmodel.tiangolo.com/)
- [Alembic 官方文档 / Alembic Docs](https://alembic.sqlalchemy.org/)
- [MySQL 8.0 文档 / MySQL 8.0 Docs](https://dev.mysql.com/doc/)

---

## 立即开始 / Get Started Now

```bash
# 1. 查看详细分析 / Read detailed analysis
cat MYSQL_SUPPORT_ANALYSIS.md  # 中文 / Chinese
cat MYSQL_SUPPORT_ANALYSIS_EN.md  # English

# 2. 根据方案实施 / Implement based on chosen approach
# 参考完整文档中的详细步骤 / Refer to detailed steps in full documentation
```

---

**结论 / Conclusion**: 
✅ **完全支持 MySQL，实施简单，风险可控！**
✅ **Full MySQL support, simple implementation, manageable risks!**
