# EpicHost 保活脚本

基于 GitHub Actions 的定时续期脚本,用于自动续期 EpicHost 免费服务器,支持同时管理多个服务器,并通过 Telegram 推送续期结果。

## 功能

- 每小时自动触发一次,执行前随机等待 1~5 分钟,避免请求时间过于规律
- 支持多个服务器,URL 与 API Key 按 JSON 数组下标一一对应
- 单个服务器续期失败自动重试 3 次
- 某个服务器失败不影响其他服务器继续续期
- 自动提取接口返回的 `detail`/`message` 字段,通过 Telegram Bot 推送每台服务器的成功/失败详情
- 每次运行结束后把最后运行时间和总体状态写入仓库的 `time.txt` 并自动提交
- 支持手动触发(workflow_dispatch),方便测试

## 使用方法

### 1. 放置 workflow 文件

将 `keepalive.yml` 放到仓库的 `.github/workflows/keepalive.yml`

### 2. 配置 Secrets

进入仓库 `Settings → Secrets and variables → Actions → New repository secret`,新增以下 Secrets:

| Secret 名称 | 说明 |
|---|---|
| `EPICHOST_BASE_URLS` | JSON 数组,每个服务器的续期接口地址 |
| `EPICHOST_API_KEYS` | JSON 数组,与 `EPICHOST_BASE_URLS` 按下标一一对应的 API Key |
| `TELEGRAM_BOT_TOKEN` | Telegram Bot Token,找 [@BotFather](https://t.me/BotFather) 创建机器人获取 |
| `TELEGRAM_CHAT_ID` | 接收通知的 chat id,可发消息给 [@userinfobot](https://t.me/userinfobot) 获取自己的 id |

`EPICHOST_BASE_URLS` 示例:

```json
[
  "https://panel.epichost.pl/api/client/freeservers/7f8e936f-9b70-4041-ad83-5911155d642e/renew",
  "https://panel.epichost.pl/api/client/freeservers/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/renew"
]
```

`EPICHOST_API_KEYS` 示例:

```json
[
  "key_for_server_1",
  "key_for_server_2"
]
```

> 两个数组长度必须一致,下标 `i` 的 URL 对应下标 `i` 的 Key。脚本启动时会自动校验数量是否匹配,不匹配会直接报错退出。

### 3. 调整执行频率(可选)

默认 cron 为:

```yaml
schedule:
  - cron: '0 * * * *'   # 每小时一次,UTC 时间
```

实际执行时会在触发后随机等待 60~300 秒(1~5 分钟)再开始续期,代码位置:

```bash
DELAY=$(( (RANDOM % 300) + 60 ))
sleep "$DELAY"
```

如需调整频率或随机延迟范围,直接修改 cron 表达式或 `RANDOM % 300` 的取值范围即可。

### 4. 手动测试

仓库页面 → `Actions` → 选择 `EpicHost Keepalive` → `Run workflow`,可立即触发一次执行,查看日志确认是否成功。

## Telegram 通知

每次运行结束(无论成功失败)都会发送一条汇总消息,格式示例:

```
EpicHost 保活通知
时间: 2026-06-22 14:03:27 UTC
总体状态: ⚠️ 部分/全部续期失败

✅ [1] https://panel.epichost.pl/.../7f8e936f.../renew
状态: 成功
详情: 续期成功

❌ [2] https://panel.epichost.pl/.../xxxxxxxx.../renew
状态: 失败
详情: You can't renew your server currently, because you can only once at one time period.
```

`详情` 字段提取逻辑:

- 接口返回 `errors` 字段(失败响应,如官方报错示例)→ 取 `errors[0].detail`
- 接口返回 `message` 字段 → 取 `message`
- 都没有(正常成功响应)→ 显示“续期成功”

## time.txt 记录

每次运行结束后,会在仓库根目录写入/更新 `time.txt`:

```
Last run: 2026-06-22 14:03:27 UTC
Status: SUCCESS
```

并由 `github-actions[bot]` 自动 commit & push 回仓库,提交信息带 `[skip ci]` 避免触发额外的 CI。

> 注意:workflow 中已声明 `permissions: contents: write`,这是 push 成功所必需的权限。如果仓库分支设置了保护规则(branch protection)禁止 Actions 直接 push,需要额外配置 PAT 或调整保护规则例外。

## 新增/删除服务器

无需修改 workflow 文件,只需要同步编辑 `EPICHOST_BASE_URLS` 和 `EPICHOST_API_KEYS` 两个 Secrets,新增/删除对应下标的条目即可。

## 常见问题

**Q: API Key 过期了怎么办?**
A: 重新在面板生成 Key,更新对应下标的 `EPICHOST_API_KEYS` 即可,无需改动代码。

**Q: 想给不同服务器设置不同的重试次数/间隔?**
A: 当前版本所有服务器共用同一套重试策略(3 次,间隔 10 秒)。如有差异化需求需要扩展脚本逻辑,可以再提出来。

**Q: cron 时间是哪个时区?**
A: GitHub Actions 的 cron 固定为 UTC 时间,需要换算成你所在时区对应的时间。

**Q: 没收到 Telegram 消息怎么排查?**
A: 检查 `TELEGRAM_BOT_TOKEN` 和 `TELEGRAM_CHAT_ID` 是否正确;确认你已经在 Telegram 里先给机器人发过一条消息(否则机器人无法主动推送给你);查看 Actions 日志中 `Send Telegram notification` 步骤的输出,看 Telegram API 是否返回了错误信息。

**Q: time.txt 没有更新/push 失败?**
A: 检查 workflow 文件里是否有 `permissions: contents: write`;如果仓库开启了分支保护且不允许 GitHub Actions bot 推送,需要在保护规则里为 Actions 添加例外,或改用具备写权限的 PAT。
