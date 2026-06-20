# 代码审查示例

这些示例用于校准输出粒度，不是固定模板。审查真实变更时，以代码证据和用户指定范围为准。

## 示例一：轻量审查，局部 bugfix

用户请求：

```text
使用 $code-review 审查当前工作区修改，轻量一点。
```

变更摘要：

```diff
- if user == nil || user.ID == "" {
+ if user.ID == "" {
    return ErrUnauthorized
  }
```

合格输出：

```markdown
**问题**
- High: `internal/auth/session.go:42` 这里删掉了 `user == nil` 保护，但上游 `LoadSession` 在 token 过期时会返回 `(nil, nil)`。当前代码会在匿名请求上 panic，影响所有需要会话校验的接口。建议保留 nil 检查，或让 `LoadSession` 明确返回错误并在调用方处理。

**测试**
未运行：这是轻量审查，只定位到相关路径。
缺口/残余风险：建议补一个过期 token 返回 nil user 的回归测试。
```

要点：

- 轻量审查只输出确认到的高信号问题。
- 不展开完整架构形态。
- 不为了凑数量添加泛泛建议。

## 示例二：严格审查，架构边界变化

用户请求：

```text
使用 $code-review 严格审查这个 PR。
```

变更摘要：

```diff
+func (h *OrderHandler) Cancel(w http.ResponseWriter, r *http.Request) {
+  order := h.store.GetOrder(id)
+  if order.Status == "paid" {
+    h.inventory.Release(order.Items)
+    h.store.UpdateStatus(id, "cancelled")
+  }
+}
```

合格输出：

```markdown
**问题**
- High: `internal/http/order_handler.go:88` handler 直接读取订单、判断状态、释放库存并写状态，绕过了已有 `OrderService.Cancel` 的状态机和审计事件。API 入口和 service 会出现两套取消规则，后续 CLI/worker 调用仍走旧逻辑，容易产生库存和审计不一致。建议 handler 只做鉴权和参数解析，然后调用统一的 application service。
- Medium: `internal/http/order_handler.go:91` 释放库存和更新订单状态不在同一事务或补偿流程里。库存释放成功但状态写入失败时，订单仍显示 paid，库存却已释放。建议复用 service 里的事务边界，或显式建模补偿/重试。
- Medium: `internal/http/order_handler_test.go:34` 新增测试只覆盖 paid 订单成功取消，没有覆盖重复取消、库存释放失败和审计事件写入。建议增加至少一个库存失败不更新订单状态的回归测试。

**架构形态**
业务语义: 订单取消需要统一处理状态迁移、库存释放和审计事件。
架构位置: HTTP handler 是入口层，`OrderService` 是应用编排层，repository/inventory client 是持久化和外部适配器。
变更前形态: API/CLI -> OrderService -> domain policy -> repository/inventory adapter
变更后形态: API handler -> repository/inventory adapter，并和 OrderService 形成双脑。
判断: 这是绕过边界，不是必要的新抽象。

**测试**
已定位: `go test ./internal/http ./internal/order`
缺口/残余风险: 需要覆盖库存失败和重复取消，否则这次边界回归不会稳定暴露。
```

要点：

- 严格审查至少给出三条有价值反馈。
- 架构问题要说明业务语义、位置、变更前后形态和风险。
- 测试反馈必须指向会在回归时失败的行为。

## 示例三：严格审查，并发和一致性风险

用户请求：

```text
使用 $code-review 看这个 commit 有没有数据一致性风险。
```

变更摘要：

```diff
+existing := repo.FindByEmail(email)
+if existing == nil {
+  repo.Insert(User{Email: email})
+}
```

合格输出：

```markdown
**问题**
- High: `internal/user/register.go:57` 先查 email 再插入不是原子操作，并发注册同一邮箱时两个请求都可能看到空结果并插入成功。建议用数据库唯一索引作为最终约束，并把 duplicate key 映射成业务错误。
- Medium: `internal/user/register.go:63` duplicate key 没有被分类处理。如果后续补唯一索引，当前错误会变成 500，调用方无法区分已注册和系统失败。建议在 repository 或 service 边界转换为 `ErrEmailAlreadyRegistered`。
- Medium: `internal/user/register_test.go:18` 测试只串行调用注册流程，不能证明并发安全。建议增加并发同邮箱注册用例，断言只有一个成功，其余返回已注册错误。

**测试**
未运行：当前只审查 commit，未执行本地命令。
缺口/残余风险：需要确认 migration 是否已经包含 email 唯一索引。

**开放问题**
- 数据库 schema 里是否已有 `users.email` 唯一约束？如果没有，这是必须补的生产约束。
```

要点：

- 一致性问题优先看原子性、唯一约束、错误分类和测试能否回归。
- 没证据的 schema 状态放到开放问题，不写成确定结论。
