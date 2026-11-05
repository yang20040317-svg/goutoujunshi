# wuuud MCP 连接与工具清单（wechat-data-analysis-mcp v1.0.0）

server 名：`wechat-data-analysis-mcp`，协议 streamable HTTP（无 `Mcp-Session-Id`，单次请求独立）。
地址与认证见 `../config.local.json`：`mcp_endpoint` 走 JSON-RPC，`api_base` 走消息修改 HTTP API，均带 `Authorization: Bearer <token>`。

## 调用骨架

```text
POST mcp_endpoint
Content-Type: application/json
Authorization: Bearer <token>
Accept: application/json, text/event-stream
```
```json
{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"wb","version":"1.0"}}}
{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"<tool>","arguments":{...}}}
```
- 优先读 `result.structuredContent`；content[].text 是兜底。
- 真实微信动作前先 `wechat.hook.call_tool name=wechat_hook_status`，仅 `ready=true` 继续；文本与图片分离，文本优先；`call_completed` ≠ 送达/渲染；结果未知不自动重发。

## 工具分组（共 74 个，一页返回）

### 读 / 分析（默认开启）
- chat：`list_sessions` `resolve_session` `get_messages` `search_messages` `get_message_around` `get_message_anchor` `get_message_raw` `resolve_chat_history` `get_daily_message_counts` `list_search_senders` `resolve_app_message`
- contacts：`list_contacts` `resolve_contact`
- analytics：`get_wrapped_annual` `get_wrapped_card` `get_wrapped_meta`
- mobile：`get_overview` `get_home_snapshot` `get_chat_context` `get_session_bundle` `get_message_media_bundle` `get_media_links` `get_analytics` `resolve_target` `search_chat` `search_context` `search_moments`
- media（返回 URL，不内联）：`get_chat_image_url` `download_chat_image(s)` `get_chat_voice_url` `get_chat_video_url` `get_chat_emoji_url` `get_avatar_url` `get_decrypted_resource_url` `get_proxy_image_url`
- biz（公众号）：`list_accounts` `get_messages` `get_pay_records` `get_proxy_image_url`
- moments：`list_timeline` `list_users` `search_moments` `get_media_url` `get_local_video_url` `get_remote_video_url` `get_article_thumb_url` `get_self_info`
- core：`get_status` `get_account_info` `list_accounts` `list_tools`

### 真实发送（需确认）
- hook：`wechat.hook.list_tools` `wechat.hook.call_tool`（内部 `wechat_send_text` ✅、`wechat_send_at` ✅；`wechat_send_image/video/emoji/file` 当前 501 未实现）
- sending：`send_text` `forward_messages`

### 修改 / 补录（③ 写操作，逐条确认 + 审计）
- editing：`add_text_message` `add_image_message` `add_file_message` `add_typed_message` `edit_message` `edit_image_message` `delete_message` `flip_message_direction` `repair_message_sender` `get_message_edit_status` `list_edited_messages` `list_edited_sessions` `reset_message_edit` `reset_session_edits`
- business：`list_operations` `call_operation`（OpenAPI 业务路由，含 chat/contacts/moments/favorites/automation/advanced/exports/analytics；`destructiveHint` 部分为真）

## 上下文预算
- `get_messages` / `biz.get_messages` 等 limit≤200；列表类先小 limit 探路，再按需展开明细。
- 说话人映射先 `resolve_session`/`resolve_contact` 锁定，避免按左右或语气猜。
- 媒体与导出只传 URL，绝不在模型上下文内联二进制/大文本。
