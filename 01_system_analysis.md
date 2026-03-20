# 01 系统分析报告

## 项目一句话摘要
【已确认】这是一个基于FastAPI+SQLAlchemy构建的多角色电商后端API系统，完整实现客户/供应商认证、购物车管理、订单处理、Paystack支付集成和ARQ异步任务队列架构。

---

## 系统分层

| 层级 | 目录 | 职责 | 确认状态 |
|------|------|------|----------|
| API层 | `api/endpoints/` | 路由定义、请求参数校验、依赖注入 | 已确认 |
| 服务层 | `services/` | 业务逻辑编排、跨CRUD操作协调 | 已确认 |
| CRUD层 | `crud/` | 数据访问抽象、数据库操作封装 | 已确认 |
| 模型层 | `models/` | SQLAlchemy ORM模型定义、数据库表结构 | 已确认 |
| Schema层 | `schemas/` | Pydantic数据验证、请求/响应模式定义 | 已确认 |
| 核心层 | `core/` | 数据库连接、认证令牌、错误处理、支付客户端 | 已确认 |
| 依赖注入层 | `api/dependencies/` | 服务和CRUD实例构造、依赖管理 | 已确认 |
| 任务队列 | `task_queue/` | 异步任务处理、定时任务调度 | 已确认 |

---

## 核心业务链路

### 链路1：购物车→结账→支付完整流程
【已确认】
```
客户认证→添加商品到购物车（/cart/add）→结账（/cart/checkout）→创建订单→
异步创建配送详情/订单项→跳转Paystack支付→验证支付（/cart/verify-payment/{ref}）→
创建支付记录→异步更新库存→【缺少：清空购物车】
```

### 链路2：供应商订单管理
【已确认】
```
供应商认证→获取订单列表（/order/vendor/）→更新订单项状态（/order/order-items/{id}/status）→
查看销售统计（/order/vendor/activity、/order/vendor/sales/date）
```

### 链路3：认证与用户注册
【已确认】
```
注册→创建AuthUser→发送OTP→验证OTP→创建Customer/Vendor profile→获取Token→
访问带权限的接口
```

---

## 最关键的端到端调用链
【已确认】

```
POST /cart/checkout (api/endpoints/cart.py:75)
  ↓ Depends(get_current_verified_customer)
  ↓ Depends(get_cart_service)
  → CartService.checkout() (services/cart_service.py:94)
    → CRUDCart.get_cart_summary() (crud/cart.py:29)
    → CRUDCustomer.get() (crud/base.py:25)
    → CRUDOrder.create() (crud/base.py:55) 【创建订单】
    → queue_connection.enqueue_job("add_shipping_details")
    → queue_connection.enqueue_job("add_order_items")
    → Paystack.initialize_payment() (core/paystack.py:18) 【跳转到支付】

GET /cart/verify-payment/{payment_ref} (api/endpoints/cart.py:85)
  → CartService.verify_order_payment() (services/cart_service.py:148)
    → CRUDPayment.get_by_payment_ref() (crud/order.py:80)
    → Paystack.verify_payment() (core/paystack.py:51) 【验证支付】
    → CRUDPayment.create() (crud/base.py:55) 【创建支付记录】
    → queue_connection.enqueue_job("update_stock_after_checkout") 【异步扣减库存】
    → 【未调用：CRUDCart.clear_cart()】
```

---

## 5个架构特征

1. **完整依赖注入体系**【已确认】
   - 两层注入：CRUD层依赖（`get_crud_order`等）+ 服务层依赖（`get_order_service`等）
   - 实现位置：`api/dependencies/services.py`、`crud/*.py`

2. **异步/同步混合执行模型**【已确认】
   - 服务层（services）统一使用 `async/await` 包装
   - CRUD层为同步SQLAlchemy调用，底层未使用异步驱动
   - 支付和队列使用 `httpx.AsyncClient`、`arq` 真正异步库

3. **ARQ任务队列解耦核心业务**【已确认】
   - 配置完整：`task_queue/main.py` 定义 `WorkerSettings`
   - 任务上下文注入CRUD操作
   - 解耦操作：库存更新、订单项创建、邮件发送

4. **角色分离的鉴权体系**【已确认】
   - 双角色：`Roles.CUSTOMER`、`Roles.VENDOR`
   - 路由级保护：`get_current_verified_customer`、`get_current_verified_vendor`
   - Token校验流程：黑名单→解码→用户存在性→角色校验

5. **CRUD泛型基类+业务定制**【已确认】
   - 泛型基类 `CRUDBase` 提供标准CRUD操作
   - 业务CRUD类继承基类并扩展特定查询
   - 示例：`CRUDOrder.get_all_orders()` 带joinedload优化

---

## 3个最可能出问题的环节

1. **数据库会话泄漏风险**【推断，高风险】
   - 代码证据：`core/db.py:11-16` 的 `get_db()` 用 `return` 而非 `yield`
   - 推断影响：推断 `Depends(get_db)` 注入的会话可能无法触发 `finally` 的 `db.close()`
   - 未确认：缺少实际运行测试验证会话泄漏行为

2. **Token黑名单内存存储不可靠**【已确认，高风险】
   - 代码证据：`core/tokens.py:20` 定义 `BLACKLISTED_TOKEN = []`
   - 已确认限制：内存列表无持久化，服务重启即失效；多实例部署时状态不同步

3. **库存更新竞态条件**【已确认，高风险】
   - 代码证据：`task_queue/tasks/cart_tasks.py:67-69` 先读stock再减再update
   - 已确认风险：非原子读-改-写操作，高并发下存在超卖风险

---

## 仍需继续排查的问题

1. **数据库会话实际行为**【未确认】缺少代码证据：缺少实际运行测试验证 `get_db()` 的会话关闭行为

2. **支付回调接口**【未确认】缺少代码证据：未找到Paystack Webhook回调处理接口

3. **订单状态回滚机制**【未确认】缺少代码证据：支付失败场景下的订单补偿逻辑未实现

4. **任务异常重试机制**【未确认】缺少代码证据：`task_queue/main.py` 中未配置ARQ任务重试策略

5. **分页查询实现**【已确认】已确认：列表接口（如 `/order/`、`/order/vendor/`）无分页参数处理

6. **接口幂等性保障**【未确认】缺少代码证据：未找到重复请求防护或幂等性校验机制