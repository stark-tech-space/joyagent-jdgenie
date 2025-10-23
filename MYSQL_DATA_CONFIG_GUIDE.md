# MySQL 自定义数据配置指南

> 完整指南：如何配置自己的 MySQL 数据库，让 JoyAgent-JDGenie 的 DataAgent 智能分析你的业务数据

## 目录

- [概述](#概述)
- [前置准备](#前置准备)
- [配置步骤](#配置步骤)
- [DataAgent 配置详解](#dataagent-配置详解)
- [示例：电商数据配置](#示例电商数据配置)
- [高级功能](#高级功能)
- [常见问题](#常见问题)

---

## 概述

JoyAgent-JDGenie 的 **DataAgent** 模块支持直接查询和分析你的 MySQL 数据库。通过简单配置，你可以：

✅ **自然语言查询数据** - "查询最近三个月销售额"
✅ **智能分析** - 趋势分析、异常检测、相关性分析
✅ **自动生成可视化** - 图表、报表自动生成
✅ **业务知识注入** - 添加业务说明，提升理解准确性

---

## 前置准备

### 1. 环境要求

- ✅ MySQL 5.7+ 或 8.0+
- ✅ 数据库有表结构和数据
- ✅ 有数据库访问权限（至少 SELECT 权限）

### 2. 需要准备的信息

```
数据库连接信息：
- 主机地址：如 127.0.0.1
- 端口：默认 3306
- 数据库名：如 my_business_db
- 用户名：如 root
- 密码：如 mypassword

业务信息：
- 表的业务含义
- 字段的业务含义
- 常用查询场景
```

---

## 配置步骤

### 步骤 1：配置数据库连接

编辑 `genie-backend/src/main/resources/application.yml`：

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/my_business_db?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: root
    password: mypassword
```

**参数说明：**

| 参数 | 说明 | 示例 |
|------|------|------|
| `url` | 数据库连接地址 | `jdbc:mysql://127.0.0.1:3306/sales_db` |
| `username` | 数据库用户名 | `root` 或 `readonly_user` |
| `password` | 数据库密码 | 你的密码 |
| `useSSL` | 是否使用 SSL | `false`（本地开发）<br>`true`（生产环境） |
| `serverTimezone` | 时区 | `UTC` 或 `Asia/Shanghai` |

**连接其他主机的数据库：**

```yaml
# 连接远程服务器
url: jdbc:mysql://192.168.1.100:3306/my_database

# 连接云数据库
url: jdbc:mysql://rds.example.com:3306/my_database?useSSL=true
```

### 步骤 2：配置 DataAgent

在同一个 `application.yml` 文件中配置 DataAgent：

```yaml
autobots:
  data-agent:
    agent-url: https://joyagent-llm.datamunger.io  # 或你的服务地址

    # 数据库配置（与上面的 datasource 保持一致）
    db-config:
      type: mysql              # 数据库类型：mysql/h2/clickhouse
      host: 127.0.0.1          # 主机地址
      port: 3306               # 端口
      schema: my_business_db   # 数据库名
      username: root           # 用户名
      password: mypassword     # 密码

    # 向量数据库配置（可选，用于语义搜索）
    qdrantConfig:
      enable: false            # 暂时关闭，后续需要时开启
      embeddingUrl: http://embedding-service.local
      host: 127.0.0.1
      port: 6333
      apiKey: your-api-key

    # Elasticsearch 配置（可选，用于全文搜索）
    es-config:
      enable: false            # 暂时关闭
      host: 127.0.0.1:9200
      user: elastic
      password: password
```

### 步骤 3：配置数据模型（Model-List）

这是**最关键的一步**！定义你的表和字段的业务含义。

```yaml
autobots:
  data-agent:
    model-list:
      # 第一个表：订单表
      - name: 订单数据表                    # 表的中文名
        id: t_orders                       # 表的标识（表名或别名）
        type: table                        # 类型：table 或 sql
        content: orders                    # 实际的表名
        remark: 订单基础信息表，包含订单状态、金额、时间等
        business-prompt: |                 # 业务提示（重要！）
          order_date 是订单创建时间，日维度数据。
          如果要按月统计，使用 DATE_FORMAT(order_date, '%Y-%m')。
          total_amount 是订单总金额，单位为元。
          status 字段：pending=待支付, paid=已支付, shipped=已发货, completed=已完成, cancelled=已取消
        ignore-fields: created_by,updated_by,deleted_at  # 忽略的字段
        default-recall-fields: order_id,order_date,total_amount,status  # 默认召回字段
        analyze-suggest-fields: status,payment_method    # 建议用于分析的字段
        analyze-forbid-fields: customer_phone,customer_email  # 禁止用于分析的敏感字段
        sync-value-fields: status,payment_method         # 需要同步枚举值的字段
        column-alias-map: |                             # 字段别名映射（JSON 格式）
          {
            "order_date": "订单时间,下单时间,创建时间",
            "total_amount": "订单金额,总金额,金额",
            "customer_id": "客户ID,会员ID,用户ID",
            "status": "订单状态,状态"
          }

      # 第二个表：产品表
      - name: 产品信息表
        id: t_products
        type: table
        content: products
        remark: 产品基础信息，包含产品名称、价格、分类等
        business-prompt: |
          price 是产品价格，单位为元。
          category 是产品分类，包括：电子产品、服装、食品、图书等。
          stock 是库存数量。
        ignore-fields: created_at,updated_at
        default-recall-fields: product_id,product_name,price,category
        analyze-suggest-fields: category,price_range
        sync-value-fields: category
        column-alias-map: |
          {
            "product_name": "产品名称,商品名称,名称",
            "price": "价格,单价,售价",
            "category": "分类,类别,品类"
          }
```

**配置项详解：**

| 配置项 | 必填 | 说明 | 示例 |
|--------|------|------|------|
| `name` | ✅ | 表的中文名称 | "订单数据表" |
| `id` | ✅ | 表的唯一标识 | "t_orders" |
| `type` | ✅ | 类型 | "table" 或 "sql" |
| `content` | ✅ | 实际表名或 SQL | "orders" |
| `remark` | ✅ | 表的业务说明 | "订单基础信息表..." |
| `business-prompt` | 🔥 **重要** | 业务规则说明 | 时间格式、枚举值含义、计算规则等 |
| `ignore-fields` | ❌ | 要忽略的字段 | "password,secret_key" |
| `default-recall-fields` | ❌ | 默认查询的字段 | "id,name,status" |
| `analyze-suggest-fields` | ❌ | 建议用于分析的字段 | "status,category" |
| `analyze-forbid-fields` | ❌ | 禁止分析的敏感字段 | "phone,email,id_card" |
| `sync-value-fields` | ❌ | 需要同步枚举值的字段 | "status,type" |
| `column-alias-map` | 🔥 **重要** | 字段的中文别名 | JSON 格式映射 |

### 步骤 4：初始化数据库表

JoyAgent 需要一些系统表来存储配置和运行数据。

**执行初始化 SQL：**

```bash
# 连接到你的数据库
mysql -h127.0.0.1 -P3306 -uroot -p

# 选择数据库
USE my_business_db;

# 执行初始化脚本
SOURCE /path/to/joyagent-jdgenie/genie-backend/src/main/resources/db/schema.sql;
```

**或者手动创建：**

```sql
CREATE TABLE `chat_model_info` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `code` varchar(50) NOT NULL COMMENT '模型编码',
  `type` varchar(10) NOT NULL COMMENT '模型类型TABLE,SQL',
  `name` varchar(100) DEFAULT NULL COMMENT '模型名称',
  `content` text NOT NULL COMMENT '模型内容，表或者sql',
  `use_prompt` text COMMENT '模型使用说明',
  `business_prompt` text COMMENT '模型业务限定提示词',
  `yn` tinyint(2) NOT NULL DEFAULT '1' COMMENT '是否有效',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据模型表信息';

CREATE TABLE `chat_model_schema` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `model_code` varchar(200) NOT NULL COMMENT '模型编码',
  `column_id` varchar(1000) NOT NULL COMMENT '字段唯一ID',
  `column_name` varchar(200) NOT NULL COMMENT '字段中文名',
  `column_comment` varchar(1000) NOT NULL COMMENT '字段描述',
  `few_shot` text COMMENT '值枚举逗号分隔',
  `data_type` varchar(20) DEFAULT NULL COMMENT '字段值类型',
  `synonyms` varchar(300) DEFAULT NULL COMMENT '同义词',
  `vector_uuid` varchar(400) DEFAULT NULL COMMENT '向量库数据id',
  `default_recall` tinyint(2) NOT NULL DEFAULT '0' COMMENT '默认召回',
  `analyze_suggest` tinyint(2) NOT NULL DEFAULT '0' COMMENT '分析建议',
  `yn` tinyint(2) NOT NULL DEFAULT '1' COMMENT '是否有效',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='数据模型字段信息';
```

### 步骤 5：重启服务

**如果使用 Docker：**

```bash
# 停止旧容器
sudo docker stop genie-app
sudo docker rm genie-app

# 重新构建镜像（包含新配置）
sudo docker build -t genie:latest .

# 启动容器（使用 host 网络访问本地 MySQL）
sudo docker run -d --network host --name genie-app genie:latest

# 检查状态
sudo docker ps | grep genie-app
sudo docker logs genie-app -f
```

**如果本地运行：**

```bash
# 重新编译后端
cd genie-backend
sh build.sh

# 重启所有服务
cd ..
sh Genie_start.sh
```

### 步骤 6：测试连接

访问前端（http://localhost:3000 或 3002），尝试查询：

```
查询订单表的前 10 条数据
```

如果返回数据，说明配置成功！🎉

---

## DataAgent 配置详解

### DGP 协议（Data Governance Protocol）

DGP 是 JoyAgent 定义的数据治理协议，包含三个层级：

#### 1. 表级（Table Level）

**配置：**
```yaml
- name: 订单表              # 表的业务名称
  id: t_orders             # 唯一标识
  content: orders          # 实际表名
  remark: 订单基础信息      # 表的业务描述
  business-prompt: |       # 🔥 关键：业务规则
    - 时间字段说明
    - 状态字段枚举值
    - 特殊计算逻辑
    - 业务约束条件
```

**为什么重要？**
- 帮助 AI 理解表的业务含义
- 生成更准确的 SQL
- 避免错误的数据解读

#### 2. 字段级（Column Level）

**配置：**
```yaml
column-alias-map: |
  {
    "order_date": "订单时间,下单时间,创建时间",
    "total_amount": "订单金额,总金额",
    "status": "订单状态,状态"
  }
```

**同义词映射：**
```
用户说 → 实际字段
"销售额" → "total_amount"
"下单时间" → "order_date"
"订单状态" → "status"
```

**字段分类：**

| 分类 | 配置 | 用途 |
|------|------|------|
| **默认召回** | `default-recall-fields` | 查询时默认返回这些字段 |
| **分析建议** | `analyze-suggest-fields` | 适合用于分组、聚合的字段 |
| **分析禁止** | `analyze-forbid-fields` | 敏感字段，不能用于分析 |
| **忽略** | `ignore-fields` | 完全忽略，AI 看不到 |

#### 3. 值级（Value Level）

**枚举值同步：**
```yaml
sync-value-fields: status,payment_method
```

启用后，DataAgent 会自动提取这些字段的所有值：

```sql
SELECT DISTINCT status FROM orders;
-- 结果：pending, paid, shipped, completed, cancelled

SELECT DISTINCT payment_method FROM orders;
-- 结果：alipay, wechat, credit_card, cash
```

**在 business-prompt 中说明含义：**
```yaml
business-prompt: |
  status 字段的含义：
  - pending: 待支付
  - paid: 已支付
  - shipped: 已发货
  - completed: 已完成
  - cancelled: 已取消
```

这样 AI 就能正确理解和使用这些值！

---

## 示例：电商数据配置

假设你有一个电商系统，包含以下表：

```
- orders (订单表)
- order_items (订单明细表)
- products (产品表)
- customers (客户表)
```

### 完整配置示例

```yaml
autobots:
  data-agent:
    db-config:
      type: mysql
      host: 127.0.0.1
      port: 3306
      schema: ecommerce_db
      username: root
      password: ecommerce2024

    model-list:
      # ========== 订单表 ==========
      - name: 订单数据
        id: t_orders
        type: table
        content: orders
        remark: 存储所有订单的基本信息，包括订单状态、金额、时间等
        business-prompt: |
          **时间字段：**
          - order_date: 订单创建时间（日维度）
          - 按月统计使用：DATE_FORMAT(order_date, '%Y-%m')
          - 按年统计使用：YEAR(order_date)

          **金额字段：**
          - total_amount: 订单总金额（元）
          - shipping_fee: 运费（元）
          - discount_amount: 优惠金额（元）

          **状态字段：**
          - status: 订单状态
            * pending: 待支付
            * paid: 已支付
            * processing: 处理中
            * shipped: 已发货
            * delivered: 已送达
            * completed: 已完成
            * cancelled: 已取消
            * refunded: 已退款

          **查询规则：**
          - 统计销售额时只统计 completed 状态的订单
          - 计算平均订单金额时排除 cancelled 和 refunded

        ignore-fields: created_by,updated_by,deleted_at,internal_notes
        default-recall-fields: order_id,order_no,order_date,total_amount,status,customer_id
        analyze-suggest-fields: status,payment_method,shipping_method
        analyze-forbid-fields:
        sync-value-fields: status,payment_method,shipping_method
        column-alias-map: |
          {
            "order_date": "订单时间,下单时间,订单日期,创建时间",
            "order_no": "订单号,订单编号",
            "total_amount": "订单金额,总金额,销售额,金额",
            "status": "订单状态,状态",
            "customer_id": "客户ID,会员ID,用户ID",
            "payment_method": "支付方式,付款方式",
            "shipping_method": "配送方式,物流方式"
          }

      # ========== 订单明细表 ==========
      - name: 订单明细数据
        id: t_order_items
        type: table
        content: order_items
        remark: 订单中的商品明细，包含产品信息、数量、价格等
        business-prompt: |
          **关联关系：**
          - 通过 order_id 关联到 orders 表
          - 通过 product_id 关联到 products 表

          **金额计算：**
          - unit_price: 单价（元）
          - quantity: 数量
          - subtotal = unit_price * quantity
          - discount: 单品优惠金额
          - final_amount = subtotal - discount

          **注意事项：**
          - 一个订单可以有多个明细
          - 统计商品销量时使用 SUM(quantity)

        default-recall-fields: order_id,product_id,product_name,quantity,unit_price,subtotal
        analyze-suggest-fields: product_id,product_name,product_category
        sync-value-fields: product_category
        column-alias-map: |
          {
            "product_name": "商品名称,产品名称,名称",
            "quantity": "数量,购买数量,销量",
            "unit_price": "单价,价格",
            "subtotal": "小计,金额",
            "product_category": "商品分类,类别,品类"
          }

      # ========== 产品表 ==========
      - name: 产品信息
        id: t_products
        type: table
        content: products
        remark: 产品主数据，包含产品名称、分类、价格、库存等
        business-prompt: |
          **价格字段：**
          - price: 售价（元）
          - cost: 成本价（元）
          - profit = price - cost

          **库存字段：**
          - stock: 当前库存数量
          - stock < 10 视为低库存
          - stock = 0 视为缺货

          **分类字段：**
          - category: 一级分类（如：电子产品、服装、食品）
          - sub_category: 二级分类（如：手机、笔记本电脑）

        ignore-fields: supplier_id,supplier_info
        default-recall-fields: product_id,product_name,category,price,stock
        analyze-suggest-fields: category,sub_category,brand
        sync-value-fields: category,sub_category,brand
        column-alias-map: |
          {
            "product_name": "商品名称,产品名称,名称",
            "category": "分类,类别,品类,大类",
            "sub_category": "子分类,小类",
            "price": "价格,售价,单价",
            "stock": "库存,库存量",
            "brand": "品牌"
          }

      # ========== 客户表 ==========
      - name: 客户信息
        id: t_customers
        type: table
        content: customers
        remark: 客户基本信息，包含注册时间、等级、积分等
        business-prompt: |
          **时间字段：**
          - register_date: 注册时间
          - 按月统计新增客户：DATE_FORMAT(register_date, '%Y-%m')

          **等级字段：**
          - customer_level: 客户等级
            * normal: 普通会员
            * silver: 银卡会员
            * gold: 金卡会员
            * diamond: 钻石会员

          **积分字段：**
          - points: 当前积分
          - total_points: 历史累计积分

        ignore-fields: password,id_card,bank_account
        default-recall-fields: customer_id,customer_name,register_date,customer_level,points
        analyze-suggest-fields: customer_level,gender,age_group,city
        analyze-forbid-fields: phone,email,address,id_card
        sync-value-fields: customer_level,gender,city
        column-alias-map: |
          {
            "customer_name": "客户姓名,会员姓名,姓名",
            "register_date": "注册时间,注册日期",
            "customer_level": "客户等级,会员等级,等级",
            "points": "积分,会员积分",
            "gender": "性别",
            "city": "城市,所在城市"
          }
```

### 使用示例

配置完成后，你可以用自然语言查询：

```
✅ "查询最近三个月的销售额"
✅ "按产品分类统计销量"
✅ "找出销售额最高的 10 个客户"
✅ "分析订单状态分布"
✅ "哪些产品库存不足 10 件？"
✅ "计算金卡会员的平均订单金额"
```

DataAgent 会自动：
1. 理解你的意图
2. 召回相关的表和字段
3. 生成正确的 SQL
4. 执行查询
5. 解释结果

---

## 高级功能

### 1. 使用 SQL 视图

如果你的查询逻辑复杂，可以定义 SQL 类型的模型：

```yaml
- name: 订单汇总视图
  id: v_order_summary
  type: sql               # 注意：类型是 sql
  content: |              # SQL 查询
    SELECT
      o.order_id,
      o.order_date,
      o.total_amount,
      c.customer_name,
      c.customer_level,
      COUNT(oi.product_id) as product_count
    FROM orders o
    LEFT JOIN customers c ON o.customer_id = c.customer_id
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.status = 'completed'
    GROUP BY o.order_id
  remark: 已完成订单的汇总信息，包含客户和商品数量
  business-prompt: |
    这是一个汇总视图，已经过滤了已完成的订单。
    product_count 是该订单的商品种类数量。
```

### 2. 启用向量搜索（提升语义匹配）

**安装 Qdrant：**

```bash
docker run -d -p 6333:6333 qdrant/qdrant
```

**配置向量数据库：**

```yaml
qdrantConfig:
  enable: true
  embeddingUrl: http://localhost:8080/embedding  # 或你的 embedding 服务
  host: 127.0.0.1
  port: 6333
  apiKey: ""  # Qdrant Cloud 需要
```

**效果：**
- 更好的表名和字段名匹配
- 理解同义词和相似概念
- 支持模糊查询

### 3. 多数据库支持

**配置多个数据库：**

```yaml
db-config:
  type: mysql
  host: 127.0.0.1
  port: 3306
  schema: ecommerce_db
  username: root
  password: password

# 第二个数据库配置（需要在代码中扩展）
# 或者使用不同的 schema
```

**切换数据库：**
在查询时指定数据库名：
```
查询 warehouse_db 数据库的库存表
```

---

## 常见问题

### Q1: 配置后查询没有返回数据？

**检查清单：**

1. ✅ 数据库连接是否正确？
```bash
mysql -h127.0.0.1 -P3306 -uroot -p
USE my_database;
SHOW TABLES;
```

2. ✅ 表名是否正确？
```yaml
content: orders  # 必须是实际的表名
```

3. ✅ 是否执行了初始化脚本？
```bash
# 检查系统表是否存在
SELECT * FROM chat_model_info;
```

4. ✅ 查看后端日志：
```bash
sudo docker logs genie-app -f | grep -i error
```

### Q2: AI 理解不了我的查询？

**优化建议：**

1. **丰富同义词映射：**
```yaml
column-alias-map: |
  {
    "total_amount": "订单金额,总金额,销售额,金额,总价,交易额"
  }
```

2. **详细的业务说明：**
```yaml
business-prompt: |
  status 字段：
  - pending: 待支付（用户下单但未付款）
  - paid: 已支付（已付款但未发货）
  - shipped: 已发货（正在配送中）
  ...
```

3. **提供查询示例：**
在 `business-prompt` 中添加常见查询：
```yaml
business-prompt: |
  常见查询示例：
  - 查询本月销售额：WHERE DATE_FORMAT(order_date, '%Y-%m') = '2024-12'
  - 统计每日订单量：GROUP BY DATE(order_date)
```

### Q3: 如何处理敏感数据？

**方法 1：使用 `analyze-forbid-fields`**
```yaml
analyze-forbid-fields: phone,email,id_card,address
```

**方法 2：使用 `ignore-fields`**
```yaml
ignore-fields: password,secret_key,internal_memo
```

**方法 3：创建只读用户**
```sql
-- 创建只读用户
CREATE USER 'readonly'@'%' IDENTIFIED BY 'read_only_pass';
GRANT SELECT ON ecommerce_db.* TO 'readonly'@'%';
FLUSH PRIVILEGES;
```

然后在配置中使用只读用户：
```yaml
username: readonly
password: read_only_pass
```

### Q4: 如何提升查询性能？

**优化建议：**

1. **添加索引：**
```sql
-- 为常用查询字段添加索引
CREATE INDEX idx_order_date ON orders(order_date);
CREATE INDEX idx_status ON orders(status);
CREATE INDEX idx_customer_id ON orders(customer_id);
```

2. **使用 SQL 视图：**
预先定义复杂查询为 SQL 类型的模型。

3. **限制返回数量：**
```yaml
business-prompt: |
  默认查询只返回前 100 条记录。
  如需更多，请明确说明。
```

### Q5: Docker 容器无法连接宿主机 MySQL？

**解决方案：**

**方法 1：使用 host 网络模式**
```bash
sudo docker run -d --network host --name genie-app genie:latest
```

**方法 2：使用 host.docker.internal**
```yaml
# 在 application.yml 中
url: jdbc:mysql://host.docker.internal:3306/my_database
```

**方法 3：使用容器 IP**
```bash
# 查找 MySQL 容器 IP
sudo docker inspect mysql-server | grep IPAddress

# 使用该 IP
url: jdbc:mysql://172.17.0.4:3306/my_database
```

### Q6: 如何验证配置是否生效？

**测试步骤：**

1. **检查数据库连接：**
```bash
curl -X POST http://localhost:8080/data/queryModelInfo \
  -H "Content-Type: application/json" \
  -d '{"query": "获取所有表信息"}'
```

2. **测试简单查询：**
在前端输入：
```
查询订单表的前 5 条数据
```

3. **查看日志：**
```bash
# 后端日志
sudo docker logs genie-app -f

# Python 工具日志
sudo docker exec genie-app tail -f /app/tool/logs/app.log
```

---

## 总结

通过以上配置，你的 MySQL 数据库已经可以被 JoyAgent-JDGenie 智能查询和分析了！

**关键要点：**

✅ **database 连接配置** - 确保数据库可以访问
✅ **model-list 配置** - 定义表和字段的业务含义
✅ **business-prompt** - 提供业务规则和枚举值说明
✅ **column-alias-map** - 丰富同义词，提升理解
✅ **字段分类** - 区分默认/建议/禁止字段

**下一步：**

1. 📊 尝试更复杂的查询
2. 🎨 使用报告生成功能
3. 🔍 启用向量搜索提升精度
4. 🛠️ 根据实际使用优化配置

如有问题，查看日志或参考 [ARCHITECTURE.md](ARCHITECTURE.md) 了解更多细节。

Happy Querying! 🚀
