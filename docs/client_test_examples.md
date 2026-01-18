# 稳定性与搜索功能：测试示例指南

为了验证近期对 API 400 错误及“搜索文件错误”的修复效果，您可以在 Claude CLI (Claude Code) 中运行以下指令进行实测。

## 1. 验证搜索工具自愈 (Grep/Glob Fix)

针对之前的 "Error searching files" 问题，这些指令将触发 `Grep` 和 `Glob` 工具调用，并验证参数映射是否正确。

### 测试指令示例
*   **指令 A**：`在当前目录中搜索包含 "fn handle_messages" 的 Rust 文件。`
    *   *验证点*：检查代理是否能正确将 `query` 映射为 `pattern`，并注入默认的 `path: "."`。
*   **指令 B**：`列出 src-tauri 目录下所有 .rs 文件。`
    *   *验证点*：验证 `Glob` 工具名是否被正确识别，且路径过滤逻辑正常。

---

## 2. 验证协议顺序与签名稳定性 (Thinking/Signature Fix)

针对之前的 `Found 'text'` 和 `Invalid signature` 400 错误。

### 测试指令示例
*   **指令 A（推理+搜索）**：`分析本项目中处理云端请求的核心逻辑，按调用顺序总结，并给出关键代码行的 Grep 搜索证据。`
    *   *验证点*：这是一个复杂的“思维 -> 工具调用 -> 结果 -> 继续思维”循环。验证流式输出在收到工具结果后，是否能保持块顺序正确（不再非法注入末尾 Thinking 块）。
*   **指令 B（历史记录重试）**：手动触发一个可能导致签名的报错（例如在长对话中频繁切换模型）。
    *   *验证点*：观察 `Cargo` 日志。如果触发了 400 错误，代理应在毫秒内自动捕获关键词并静默重试。

---

## 附录：深度错误对照与修复方案

如果您在日志中看到以下具体的报错特征（通常伴随 API 400 错误），以下是我们的处理逻辑：

| 错误类别 | 具体报错特征码 (Error Detail) | 代理采取的修复/应对逻辑 |
| :--- | :--- | :--- |
| **消息流顺序违规** | `If an assistant message contains any thinking blocks, the first block must be 'thinking'... Found 'text'.` | **已修复 (Core)**：`streaming.rs` 不再允许在文字块之后非法追加思维块。 |
| **思维签名不匹配** | `Invalid signature in thinking block` / `Invalid \`signature\` in \`thinking\` block` | **已修复 (Core)**：优化工具名映射策略，优先保留原始名称以保护 Google 后端签名校验。 |
| **思维签名缺失** | `Function call is missing a thought_signature in functionCall parts.` (如 LS, Bash, TodoWrite) | **已修复 (Adaptive)**：代理自动注入 `skip_thought_signature_validator` 占位符，强制协议通行。 |
| **上下文超限** | `Prompt is too long (server-side context limit reached).` | **智能策略**：建议使用 `/compact`。代理已优化负载转换以减少不必要的冗余数据。 |
| **非法缓存标记** | `thinking.cache_control: Extra inputs are not permitted` | **已修复 (Stripping)**：全局递归清理函数 `clean_cache_control_from_messages` 会剔除这些干扰。 |
| **工具结果缺失** | `tool_use ids were found without tool_result blocks immediately after` | **核心逻辑自愈**：`close_tool_loop_for_thinking` 会通过注入合成消息自动闭合损坏的工具调用链。 |
| **搜索工具失败** | `Error searching files` | **参数对齐**：自动将 `query` 映射至 `path` 并补全执行路径。 |

---

## 3. QuotaData 字段逻辑解析

您提到的 `QuotaData` 字段是系统用于**分账号配额管理**的核心模型。

### 在代码中的主要位置
- **定义**：[models/quota.rs](file:///Users/lbjlaq/Desktop/cew/src-tauri/src/models/quota.rs#L13)
- **获取逻辑**：[modules/quota.rs](file:///Users/lbjlaq/Desktop/cew/src-tauri/src/modules/quota.rs#L116) 中的 `fetch_quota` 函数负责向 Google API 请求最新的配额数据。
- **持久化**：[modules/account.rs](file:///Users/lbjlaq/Desktop/cew/src-tauri/src/modules/account.rs#L628) 中的 `update_account_quota` 将数据保存到本地数据库。

### 目前哪些逻辑在使用它？
1.  **前端 UI 展示**：设置页面中的“账号管理”列表，下方的蓝色/灰色进度条（显示已使用/总配额）数据直接来源于 `QuotaData`。
2.  **配额保护逻辑 (Strict Group Quota)**：
    *   位于：[proxy/upstream/token_manager.rs](file:///Users/lbjlaq/Desktop/cew/src-tauri/src/proxy/upstream/token_manager.rs)
    *   用途：在请求前检查对应的账号是否有剩余配额。如果 `QuotaData` 显示该账号已达到阈值（由您的保护设置决定），系统会自动跳过该账号，轮换到下一个可用账号。
3.  **自动刷新**：系统会定期（或在报错 429 后）调用 `fetch_quota` 来同步最新的 `QuotaData`。

---

## 调试建议
如果您在运行 CLI 时想看实时的参数转换，请保持终端运行：
```bash
RUST_LOG=debug npm run tauri dev
```
在日志中搜索 `[Claude-Request]` 和 `[Streaming]` 关键词，您将看到每一个被修正的参数细节。
