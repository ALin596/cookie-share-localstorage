# localStorage 导入导出支持总结

## 需求背景

部分网站没有把登录态完整放在 Cookie 中，而是将 JWT、访问令牌、用户偏好或运行时状态保存在 `localStorage`。原有 Cookie Share 只支持 Cookie 的导出、云端保存、本地保存和后续导入，因此在这类网站上可能无法完整迁移登录态。

本次需求是在保留现有 Cookie 能力的基础上，把每条共享记录扩展为同时携带 Cookie 和 `localStorage`，并让本地模式、Node.js 后端、Cloudflare Worker 后端、管理页导入导出都能保留该数据。

## 数据模型与行为

共享记录由原来的：

```json
{
  "id": "abc123",
  "url": "https://example.com/login",
  "cookies": []
}
```

扩展为：

```json
{
  "id": "abc123",
  "url": "https://example.com/login",
  "cookies": [],
  "localStorage": [
    { "key": "token", "value": "jwt..." },
    { "key": "theme", "value": "dark" }
  ]
}
```

关键行为：

- 导出/发送时同时采集当前页面 Cookie 和 `window.localStorage`。
- Cookie 和 `localStorage` 都为空时才拒绝保存。
- 仅有 Cookie 或仅有 `localStorage` 时都允许保存和发送。
- 导入时沿用 Cookie 的覆盖式语义：先清空当前页面 Cookie，再写入记录内 Cookie；同时先清空当前页面 `localStorage`，再写入记录内 `localStorage`。
- 旧记录缺少 `localStorage` 字段时按空数组处理，保证旧数据仍可导入。
- 接口路径继续沿用 `/send-cookies`、`/receive-cookies/:id`，避免破坏现有脚本配置和部署路径。

## 主要改动点

### 油猴脚本

文件：`tampermonkey/cookie-share.user.js`

- 新增 `localStorageManager`，封装 `getAll()`、`clearAll()`、`setAll(entries)`。
- 新增 `shareRecordManager`，统一构建、校验、本地保存和导入共享记录，减少发送、本地保存、账号切换、列表导入之间的重复逻辑。
- `sendCookies()` 请求体新增 `localStorage` 字段。
- 本地保存 `cookie_share_local_<id>` 时写入完整共享记录。
- 云端接收和本地列表接收都会导入 Cookie 与 `localStorage`。
- “新增账号”和“清除本页 Cookie”流程同步清理 `localStorage`。
- 提示文案从 Cookie-only 调整为 Cookie + `localStorage`。

### Node.js 本地后端

文件：

- `server/src/types.ts`
- `server/src/validation.ts`
- `server/src/schema.ts`
- `server/src/store.ts`
- `server/src/app.ts`
- `server/src/admin-page.ts`
- `server/test/server.test.ts`

改动：

- 新增 `LocalStorageEntry { key: string; value: string }` 类型。
- `SendCookiesRequestBody`、`UpdateCookiesRequestBody`、`CookieRecord`、`ImportRecord` 等类型携带 `localStorage`。
- 新增 `normalizeLocalStorage()` 校验逻辑：
  - 缺失或 `null` 时默认 `[]`。
  - 非数组、空 key、非字符串 value 均返回 400。
- SQLite schema 增加 `local_storage_json TEXT NOT NULL DEFAULT '[]'`。
- `ensureSchema()` 会对旧库自动执行 `ALTER TABLE` 补列。
- `send-cookies`、`receive-cookies`、`admin/create`、`admin/update`、`admin/export-all`、`admin/import-all` 全链路保留 `localStorage`。
- 管理页创建/更新表单增加 `localStorage JSON` 输入框，原文查看显示 Cookie 与 `localStorage` 的完整 JSON。
- 增加后端测试覆盖：
  - Cookie + `localStorage` 保存和接收。
  - 仅 `localStorage` 保存和接收。
  - 旧 payload 缺失 `localStorage` 时默认返回 `[]`。
  - 管理导出/导入保留 `localStorage`。
  - 非法 `localStorage` payload 返回 400。

### Cloudflare Worker

文件：`_worker.js`

改动：

- D1 schema 增加 `local_storage_json`。
- `ensureSchema()` 对旧 D1 表自动补列。
- Worker 的校验、upsert、receive、admin list、admin export/import 同步支持 `localStorage`。
- Worker 内嵌管理页同步增加 `localStorage JSON` 输入框和完整原文查看。

### D1 初始迁移

文件：`migrations/0001_init.sql`

- 初始表结构增加 `local_storage_json TEXT NOT NULL DEFAULT '[]'`。

### 文档

文件：

- `README.md`
- `README_CN.md`

改动：

- 将功能描述从 Cookie-only 更新为 Cookie + `localStorage`。
- API 描述说明 `/send-cookies` 和 `/receive-cookies/:id` 支持 Cookie 与 `localStorage`。
- 安全注意事项中补充 `localStorage` 可能包含 JWT/token，长期保存需要谨慎清理。

## 验证结果

已完成本地验证：

- `npm test`，目录：`server/`
  - 结果：通过，`8 tests` 全部通过。
- `npm run build`，目录：`server/`
  - 结果：通过。
- `node --check _worker.js`
  - 结果：通过。
- `node --check tampermonkey/cookie-share.user.js`
  - 结果：通过。
- 本地 Express 后端真实 HTTP 联调：
  - 启动本地服务。
  - 通过加密请求 POST `/send-cookies` 发送 Cookie + `localStorage`。
  - GET `/receive-cookies/:id` 成功返回 Cookie 与 `localStorage`。
- 浏览器自动化验证：
  - 在本地 HTTP 页面写入 `localStorage`。
  - 导出为 `[{ key, value }]`。
  - 清空后重新导入。
  - 确认测试键值恢复成功。

Cloudflare 远端未在本地验证，部署后需要在远端 Worker + D1 环境确认补列、发送、接收、管理页导入导出均正常。

## 注意事项

- `localStorage` 是页面脚本可访问的数据，可能包含 JWT、访问令牌、用户偏好或其他敏感状态。本功能会把这些数据纳入保存与传输范围，使用时应定期清理无用记录。
- 导入策略是覆盖式：会清空当前页面 `localStorage` 后再写入记录。这样更适合账号切换和登录态恢复，但会删除目标页面已有的 UI 偏好或临时状态。
- 新客户端发送到未升级的旧后端时，旧后端可能丢弃 `localStorage` 字段；需要同步升级后端才能完整支持。
