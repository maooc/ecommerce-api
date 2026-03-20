# 02 审查发现报告

---

## 审查发现1：数据库会话泄漏风险
**风险等级**：高  
**结论状态**：推断（代码模式已确认，实际行为未验证）  
**涉及文件**：`core/db.py:11-16`、所有`crud/*.py`中的`get_crud_*`函数  
**证据**：
```python
def get_db():
    db = SessionLocal()
    try:
        return db  # 直接return而非yield
    finally:
        db.close()
```
**问题原因**：FastAPI的`Depends`对生成器（yield）和普通函数（return）行为不同；推断`return`方式可能导致`finally`块不执行。  
**缺少的证据**：缺少实际运行测试验证会话是否真正泄漏。  
**实际后果**：可能导致数据库连接池耗尽，系统响应变慢或拒绝服务。  
**修复思路**：将`return db`改为`yield db`，遵循FastAPI依赖注入最佳实践。

---

## 审查发现2：Token黑名单内存存储不可靠
**风险等级**：高  
**结论状态**：已确认  
**涉及文件**：`core/tokens.py:20-36`  
**证据**：
```python
BLACKLISTED_TOKEN = []  # 内存列表存储
lock = threading.Lock()

async def deactivate_token(token, auth_id):
    if token in BLACKLISTED_TOKEN:
        raise InvalidRequest("Already Logged Out")
    with lock:
        BLACKLISTED_TOKEN.append(token)
```
**问题原因**：使用Python内存列表存储已登出Token，无持久化机制。  
**实际后果**：已确认：服务重启后黑名单清空，已登出用户的Token可重新使用；多实例部署时状态不一致。  
**修复思路**：改用Redis集中存储黑名单，设置与Token过期时间一致的键过期。

---

## 审查发现3：库存更新竞态条件导致超卖
**风险等级**：高  
**结论状态**：已确认  
**涉及文件**：`task_queue/tasks/cart_tasks.py:58-69`  
**证据**：
```python
for product_id, cart_item_quantity in product_id_and_quantity:
    product = crud_product.get_active_products(id=product_id)
    quantity = product.stock - cart_item_quantity  # 先读
    await crud_product.update(id=product_id, data_obj={Product.STOCK: quantity})  # 后写
```
**问题原因**：非原子的读-改-写操作序列，无乐观锁/悲观锁控制。  
**实际后果**：已确认：高并发下存在竞态条件，可能导致商品超卖。  
**修复思路**：改用原子UPDATE语句：`UPDATE product SET stock = stock - :qty WHERE id = :id AND stock >= :qty`，通过rowcount判断结果。

---

## 审查发现4：支付验证异常导致未定义变量返回
**风险等级**：高  
**结论状态**：已确认  
**涉及文件**：`core/paystack.py:51-59`  
**证据**：
```python
async def verify_payment(self, payment_ref):
    try:
        rsp = await self.client.get(url=f"transaction/verify/{payment_ref}")
        rsp_data = rsp.json()["data"]
    except Exception as e:
        logger.error(e)  # 仅记录日志不抛出
    return rsp_data  # 异常时rsp_data未定义
```
**问题原因**：异常捕获后未进行变量初始化或重新抛出。  
**实际后果**：已确认：支付网关超时或网络错误时触发`UnboundLocalError`，导致支付状态不一致。  
**修复思路**：异常后抛出`InvalidRequest`异常，或在except块中初始化`rsp_data`为错误结构。

---

## 审查发现5：订单状态流转校验不完整
**风险等级**：中  
**结论状态**：已确认  
**涉及文件**：`services/order_service.py:70-74`、`models/order.py:30`、`models/order.py:61`  
**证据**：
```python
if order_item.status == OrderStatusEnum.PROCESSING:
    return await self.crud_order_item.update(...)
raise InvalidRequest("Item Status has been changed to Shipped or Refunded")
```
**问题原因**：仅允许从`PROCESSING`状态变更，未定义完整状态机；数据库默认值使用硬编码字符串`"processing"`而非枚举值。  
**实际后果**：已确认：订单状态管理混乱，无法进行退款等逆向操作；字符串与枚举值可能不匹配。  
**修复思路**：定义完整的状态转换矩阵；统一使用枚举值存储状态。

---

## 审查发现6：CRUD层async包装无实际异步操作
**风险等级**：中  
**结论状态**：已确认  
**涉及文件**：`crud/` 目录下所有文件  
**证据**：
```python
# crud/order.py:41
async def get_order_items_by_vendor_id(...) -> Optional[List[OrderItem]]:
    query = self._db.query(self.model)...all()  # 同步SQLAlchemy调用
```
**问题原因**：CRUD方法用`async`装饰但底层是同步SQLAlchemy调用，未使用异步驱动。  
**实际后果**：已确认：虚假异步，调用时仍阻塞事件循环，未获得异步性能收益。  
**修复思路**：统一改为同步调用（移除async/await）或改用SQLAlchemy 1.4+异步API + asyncpg驱动。

---

## 审查发现7：支付成功后购物车未清空
**风险等级**：中  
**结论状态**：已确认  
**涉及文件**：`services/cart_service.py:148-187`、`services/cart_service.py:137-146`  
**证据**：`verify_order_payment`方法支付成功分支未调用`self.crud_cart.clear_cart()`；货到付款分支也无清空调用。  
**问题原因**：业务流程遗漏，支付完成后应清理购物车。  
**实际后果**：已确认：用户支付成功后购物车仍显示已购商品，可能导致重复下单。  
**修复思路**：支付成功后增加`await self.crud_cart.clear_cart(customer_id)`调用。

---

## 审查发现8：API权限边界不统一
**风险等级**：高  
**结论状态**：已确认  
**涉及文件**：`api/endpoints/order.py:16-20`  
**证据**：
```python
@router.get("/")
async def get_all_orders(
    order_service: OrderService = Depends(get_order_service),
    # 缺少权限依赖：无任何认证或角色校验
):
    return await order_service.get_all_orders()
```
**问题原因**：获取所有订单的接口无认证或权限校验，匿名用户即可访问；而同文件`/vendor/`前缀接口有严格权限控制。  
**实际后果**：已确认：系统所有订单数据可被任意匿名访问，造成严重数据泄露。  
**修复思路**：添加超级管理员权限校验，或删除该接口。