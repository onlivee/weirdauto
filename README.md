# Weirdhost 自动续期脚本

Weirdhost 自动续期脚本（基于 Playwright + GitHub Actions）用于在 Weirdhost 仪表盘中为指定服务器自动点击“时间增加/续期”按钮并将运行结果通过 Telegram Bot 发送通知。本仓库包含一个可在 GitHub Actions 中执行的 Python 脚本 `main.py`，也可在本地或 CI 环境运行进行调试。

本 README 根据 `main.py`（同步 Playwright 版本）编写，详细说明配置、运行方式、环境变量、调试与常见问题排查。

---

## 目录

- 功能概览
- 工作流程
- 依赖与环境
- 环境变量（详尽说明）
- 在本地运行（调试）
- 在 GitHub Actions 中运行（示例 workflow）
- Telegram 通知格式
- 运行结果和退出码
- 常见问题与排查建议
- 安全与注意事项
- 贡献与许可

---

## 功能概览

- 支持两种登录方式：
  - Cookie 登录（使用 `REMEMBER_WEB_COOKIE`）
  - 邮箱/密码登录（`WEIRDHOST_EMAIL` + `WEIRDHOST_PASSWORD`）
- 遍历指定的服务器页面，尝试查找并点击“时间增加 / 续期”按钮（脚本对按钮文本使用多种选择器、XPath、JS 查找等策略）。
- 自动判断点击结果（成功 / 已续期 / 无按钮 / 按钮不可点击 / 页面无变化 / 未知情况等）。
- 将运行结果按可读格式通过 Telegram Bot 发送（可选）。
- 可在 headless 模式或可视化模式下运行，便于调试。

---

## 工作流程（简要）

1. 脚本读取环境变量并解析待处理的服务器 URL 列表。
2. 启动 Playwright（Chromium），创建浏览上下文与页面。
3. 尝试按优先顺序使用 Cookie 登录 -> 邮箱登录。
4. 登录成功后逐个访问服务器页面：
   - 等待页面完全加载（多个等待策略）。
   - 尝试通过多种方式定位续期按钮（文本、类名、所有按钮遍历、JS 查找）。
   - 点击按钮并通过页面内容变化 / 文本匹配判断结果。
5. 将结果通过 Telegram Bot 发送（如果已配置）。
6. 根据结果返回退出码：全部成功时返回 0，否则返回非 0。

---

## 依赖与运行环境

- Python 3.8+
- Python 包：
  - playwright
  - requests
- Playwright 浏览器二进制（需额外安装）

建议在虚拟环境中安装并运行：

示例安装命令：
```bash
python -m venv .venv
source .venv/bin/activate
pip install playwright requests
# 安装浏览器二进制
python -m playwright install --with-deps
```

在 GitHub Actions 中可使用官方 playwright action 或在 job 中运行 `python -m playwright install`。

---

## 环境变量（详尽说明）

必需/可选变量以及说明：

- WEIRDHOST_URL
  - 说明：Weirdhost 仪表盘主站地址（默认 `https://hub.weirdhost.xyz`）。
  - 可选，除非站点地址非默认。

- WEIRDHOST_LOGIN_URL
  - 说明：登录页面 URL（默认 `https://hub.weirdhost.xyz/auth/login`）。
  - 可选。

- WEIRDHOST_SERVER_URLS
  - 说明：需要续期的服务器页面列表，逗号分隔。例如：
    ```
    https://hub.weirdhost.xyz/server/user00001,https://hub.weirdhost.xyz/server/user00002
    ```
  - 必需：脚本不会在没有服务器列表时运行。

- REMEMBER_WEB_COOKIE
  - 说明：优先用于 Cookie 登录的 remember_web cookie 值（建议优先使用）。
  - 推荐：使用 Cookie 更稳定，避免验证码或二次认证。
  - 注意：仅填写 cookie 的 value 部分（脚本会构建完整 cookie 对象）。

- WEIRDHOST_EMAIL / WEIRDHOST_PASSWORD
  - 说明：备用登录方式，使用邮箱/密码在登录页提交表单。
  - 当没有 cookie 时尝试此方式。

- HEADLESS
  - 说明：是否以 headless 模式运行浏览器。默认 `true`。
  - 可选值：`true` / `false`。本地调试时可设置为 `false`。

- TELEGRAM_BOT_TOKEN
  - 说明：可选，配置后脚本会将运行结果通过该 Bot 发送到 chat。
  - 格式：Bot token，如 `123456:ABC-DEF...`

- TELEGRAM_CHAT_ID
  - 说明：接收消息的 chat id（群组或用户 ID）。

其他可用的环境变量（脚本内有默认值）：
- REMEMBER_WEB_COOKIE（上文）
- PLAYWRIGHT 环境变量（如 PWDEBUG）可以在本地用于调试

---

## 在本地运行（调试）

1. 设置环境变量（示例）：
```bash
export WEIRDHOST_SERVER_URLS="https://hub.weirdhost.xyz/server/user00001"
export REMEMBER_WEB_COOKIE="这里填remember_web cookie的value"
export TELEGRAM_BOT_TOKEN="123456:ABC-DEF"
export TELEGRAM_CHAT_ID="987654321"
export HEADLESS="false"  # 调试时设为 false
```

2. 安装依赖并启动（参考上文的安装步骤）：
```bash
python main.py
```

调试技巧：
- 如果需要可视化调试，设置 `HEADLESS=false`，并运行脚本。Playwright 会以可视窗口打开 Chromium，方便观察元素和操作流程。
- 可启用 Playwright 调试器：设置环境变量 `PWDEBUG=1`（或使用 Playwright 提供的 inspector），以便交互式调试。
- 若登录失败或找不到按钮，可保存页面内容或使用 `page.screenshot()`（在代码中临时加入）来辅助定位问题。

---

## 在 GitHub Actions 中运行（示例 workflow）

以下为示例 `.github/workflows/renew.yml`（示意）：

```yaml
name: weirdhost-renew

on:
  schedule:
    - cron: '0 2 * * *' # 每天 02:00 UTC（根据需要调整）
  workflow_dispatch:

jobs:
  renew:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install playwright requests
          python -m playwright install --with-deps

      - name: Run renew script
        env:
          WEIRDHOST_SERVER_URLS: ${{ secrets.WEIRDHOST_SERVER_URLS }}
          REMEMBER_WEB_COOKIE: ${{ secrets.REMEMBER_WEB_COOKIE }}
          WEIRDHOST_EMAIL: ${{ secrets.WEIRDHOST_EMAIL }}
          WEIRDHOST_PASSWORD: ${{ secrets.WEIRDHOST_PASSWORD }}
          TELEGRAM_BOT_TOKEN: ${{ secrets.TELEGRAM_BOT_TOKEN }}
          TELEGRAM_CHAT_ID: ${{ secrets.TELEGRAM_CHAT_ID }}
          HEADLESS: "true"
        run: |
          python main.py
```

注意：
- 将敏感信息（cookie、邮箱、密码、telegram token、chat id）存为 GitHub Secrets，避免泄露。
- 根据需要调整 cron 执行频率与时区。

---

## Telegram 通知格式与状态映射

脚本会将执行时间（北京时间）以及每台服务器的运行状态整合后通过 Telegram 发送。主要状态及映射如下：

- success -> ✅ 续期成功
- already_renewed -> ⚠️ 已经续期过了
- no_button_found -> ❌ 未找到续期按钮
- button_disabled -> ❌ 续期按钮不可点击
- login_failed -> ❌ 登录失败
- click_error -> 💥 点击按钮出错
- unknown_changed -> ⚠️ 页面变化但结果未知
- no_change -> ⚠️ 页面无变化
- error: no_auth -> ❌ 无认证信息
- error: no_servers -> ❌ 无服务器配置
- error: timeout -> ⏰ 操作超时
- error: runtime -> 💥 运行时错误

示例消息（Telegram）：
```
Weirdhost 自动续期脚本 - 最后运行时间: 2025-12-18 10:53:40（北京时间）

运行结果：
- 服务器 `user00001`: ✅ 续期成功
- 服务器 `user00002`: ✅ 续期成功
```

---

## 运行结果与退出码

- 脚本在控制台打印详细日志并最终返回：
  - 退出码 0：所有续期操作成功（无 login_failed 或 error:*）。
  - 退出码 非 0：存在登录失败或其他错误（脚本会在 main 中调用 sys.exit(1)）。

如果你在 CI 中希望避免因为部分失败导致 workflow 失败，可以在 workflow 中捕获非零退出码或在脚本外层做容错处理（例如将 python main.py 的执行包装在 try/catch 或使用 `continue-on-error: true`，但要注意这会掩盖真实错误）。

---

## 常见问题与排查建议

1. 登录失败（login_failed）
   - 检查 `REMEMBER_WEB_COOKIE` 是否正确（仅填 cookie 的 value）。
   - Cookie 可能已过期，尝试重新获取或使用邮箱/密码登录。
   - 使用 `HEADLESS=false` 本地运行观察登录流程并捕获页面错误信息。

2. 未找到续期按钮（no_button_found）
   - 网站可能已更改按钮文本或 DOM 结构，脚本当前查找关键词包括 "시간", "시간추가", "시간 추가" 等（韩文示例）。修改 `main.py` 中 `find_renew_button` 的选择器以匹配当前页面。
   - 使用本地可视化运行保存页面快照或截图来查看按钮实际文本与 class。

3. 按钮不可点击（button_disabled）
   - 按钮可能因条件（例如每日限制）被禁用，或需要额外确认对话框。观察页面的 disabled 属性或 JS 行为。
   - 检查是否存在需要先展开的 UI（如折叠面板）。

4. 页面无变化 / 未知（no_change / unknown_changed）
   - 有时候点击后页面只是局部更新或显示临时通知，脚本未能识别成功或已续期提示。可将 `success_patterns`、`error_patterns` 在脚本中扩展以匹配更多提示文本。

5. Playwright 相关错误
   - 确保在运行环境中已安装 Playwright 浏览器二进制：`python -m playwright install --with-deps`。
   - 在容器或无头 CI 中运行时若出现依赖问题，参考 Playwright 官方文档安装必要的系统依赖。

---

## 安全与注意事项

- Cookie 与账号密码属于敏感信息，请务必保存在受保护的环境变量或 GitHub Secrets 中。
- 请遵守 Weirdhost 的使用条款及自动化策略；频繁自动续期可能违反服务条款或触发限制，请合理设定运行频率。
- 在向 Telegram 发送消息时，请确保 Bot Token 与 Chat ID 的安全，避免将其硬编码到仓库中。

---

## 可扩展与改进建议

- 支持更多语言的按钮文本匹配（例如英文 "Add time" / "Renew"）。
- 更鲁棒的成功/失败识别逻辑（解析页面内具体通知或监听网络请求）。
- 并发处理多个服务器（当前为串行），但需考虑速率限制与并发登录库问题。
- 添加更完善的日志记录与持久化（例如上传到 artifact 或推送到日志服务）。

---

## 贡献与许可

欢迎提交 issues 或 pull requests 来改进脚本。请在修改时注意保密敏感信息（不要提交 cookie、密码或 tokens）。

---

感谢使用！如需根据你的 Weirdhost 页面定制选择器或调试帮助，请准备目标服务器页面的截图或 HTML（不包含敏感数据），我可以帮你调整查找与判断逻辑以提高成功率。
