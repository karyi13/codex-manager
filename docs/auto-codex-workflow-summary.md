# auto_codex.py 操作流程总结（映射到 codex-manager）

> 说明：本文档只总结样本脚本的链路结构，并映射到仓库里**已有的、可维护的功能模块**。
> 不新增或增强批量注册能力，不提供绕过风控、规避限制或扩大化自动注册的实现。

## 1. 样本脚本的整体链路

从结构上看，`auto_codex.py` 主要是在做一条自动化账号流程：

1. **准备邮箱来源**
   - 生成/申请临时邮箱或自有域名邮箱
   - 保存邮箱上下文，供后续收验证码使用

2. **准备网络环境**
   - 读取代理配置
   - 为注册/登录请求分配代理
   - 失败时切换代理或重试

3. **发起 OpenAI / ChatGPT 授权链路**
   - 初始化会话
   - 获取 Device ID
   - 处理 Sentinel / authorize 相关校验
   - 提交邮箱进入注册或登录分支

4. **分支判断：新号注册 vs 已有账号登录**
   - 如果是新号：设置密码、发送验证码、创建账户资料
   - 如果检测为老号：跳过注册资料创建，转入登录 OTP / 密码验证流程

5. **收取验证码并完成校验**
   - 轮询邮箱
   - 提取 OTP
   - 校验 OTP
   - 跟进 continue_url，确保授权 Cookie / 会话状态完整

6. **提取 workspace 并完成 OAuth 回调**
   - 从 Cookie / consent 页面 / JSON 响应中提取 workspace_id
   - 选择 workspace
   - 跟随重定向链
   - 处理 callback，换取 access_token / refresh_token / id_token

7. **落地结果**
   - 保存账号信息
   - 生成适配 `Codex CLI` 的 `auth.json`
   - 可选上传到外部服务（如 CPA / 其他管理端）

---

## 2. codex-manager 里已经对应的能力

仓库当前已经覆盖了这条链路的大部分关键步骤：

### A. 注册主链路
- 文件：`src/core/register.py`
- 已包含：
  - IP 检查
  - 邮箱创建
  - OAuth 初始化
  - Device ID 获取
  - Sentinel 校验
  - 注册/登录分支判断
  - OTP 发送与二次验证码等待
  - continue_url 跟进
  - workspace 提取
  - workspace 选择
  - redirect chain 跟随
  - OAuth callback 处理
  - token 入库

### B. Codex CLI 授权链路
- 文件：`src/core/codex_auth.py`
- 已包含：
  - 对已有账号执行 Codex Auth 登录
  - OTP 校验后解析 workspace
  - 生成 Codex CLI 兼容的 `auth.json`

### C. Web API / UI 层能力
- 文件：`src/web/routes/accounts.py`
- 已包含：
  - 单账号 Codex Auth 登录
  - 批量 Codex Auth 登录
  - `auth.json` 导出
  - 成功后持久化回数据库

### D. 注册完成后的上传链路
- 文件：`src/web/routes/registration.py`
- 已包含：
  - 注册成功后自动上传到 CPA
  - 自动上传到 Sub2API
  - 自动上传到 Team Manager / NewAPI（按仓库现有实现）

---

## 3. 推荐的本地工作流（替代“脚本堆功能”）

相比把流程继续塞进单文件脚本，仓库里更适合保持下面这套分层工作流：

### 工作流 1：注册 / 登录引擎层
目标：只负责和 OpenAI 授权链路交互。

放在：
- `src/core/register.py`
- `src/core/codex_auth.py`

职责：
- 会话初始化
- OTP / continue_url / workspace / callback
- 只返回结构化结果，不耦合上传与展示

### 工作流 2：任务编排层
目标：控制批量任务、并发、代理、失败切换、日志。

放在：
- `src/web/routes/registration.py`
- `src/web/task_manager.py`

职责：
- 任务启动
- 并发数限制
- 服务候选切换
- 代理切换
- 日志流式输出
- 成功后调用上传模块

### 工作流 3：导出 / 上传层
目标：只负责将结果转换成目标格式并发送到外部系统。

放在：
- `src/core/upload/*.py`
- `src/web/routes/accounts.py`

职责：
- 导出 `auth.json`
- 导出 JSON / CSV / ZIP
- 上传 CPA / Sub2API / TM

### 工作流 4：配置层
目标：统一从数据库读配置，而不是把大量参数散落在脚本或 `.env` 里。

放在：
- `src/config/settings.py`

职责：
- OpenAI OAuth 参数
- Web UI 参数
- 代理配置
- 邮箱服务配置
- 超时 / 重试 / 默认值

---

## 4. 我建议保留/强化的流程原则

### 原则 A：把“注册引擎”和“外部上传”解耦
原因：
- 出错更容易定位
- 调试不需要真的上传
- 更适合单元测试

### 原则 B：continue_url 必须显式跟进
这是样本脚本和仓库现有实现里都非常关键的一环。
如果 OTP 成功后不继续访问 `continue_url`，常见后果是：
- Cookie 不完整
- workspace 提取失败
- 后续 callback 断链

### 原则 C：workspace 提取要多路径兜底
当前仓库这点做得已经比较完整：
- Cookie 解码
- HTML hidden input
- URL query/fragment
- JSON payload 递归扫描

### 原则 D：Codex Auth 独立成单独引擎是对的
不要把 `auth.json` 生成混进主注册逻辑里。
这样可以：
- 复用已有账号
- 单独刷新 / 重登
- 更适合账号管理页面调用

---

## 5. 建议使用姿势

### 场景 1：已有账号，需要生成 Codex CLI 用的 auth.json
推荐：
1. 在账号管理里选中账号
2. 执行 `Codex Auth 登录`
3. 成功后导出 `auth.json`

对应仓库模块：
- `src/core/codex_auth.py`
- `src/web/routes/accounts.py`

### 场景 2：注册成功后，需要接入下游系统
推荐：
1. 注册任务只产出标准账号数据
2. 在任务编排层勾选自动上传
3. 上传动作由 `upload/*` 模块负责

### 场景 3：需要排查 OTP / workspace 问题
推荐优先检查：
1. 邮箱是否确实收到 OTP
2. OTP 校验后是否访问 `continue_url`
3. consent 页面里是否拿到 workspace_id
4. callback 链路是否完整

---

## 6. 当前结论

结论很简单：

- `auto_codex.py` 的核心价值，不在于“把一切塞进一个脚本”，而在于它验证了一条完整链路：
  **邮箱 → OTP → continue_url → workspace → callback → token/auth.json**
- `codex-manager` 当前已经具备这条链路的大部分正式实现。
- 更合理的“本地工作流程修改”不是继续增强脚本化批量注册，而是：
  - 用文档把流程讲清楚
  - 继续沿用仓库现有模块化结构
  - 把注册、Codex Auth、导出、上传分层维护

如果后续你要，我可以继续做两类安全改动之一：

1. **补架构图 / 时序图**（文档增强）
2. **补测试**（针对 continue_url / workspace 提取 / auth.json 导出）
