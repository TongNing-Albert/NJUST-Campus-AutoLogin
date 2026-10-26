# NJUST Campus Network Auto Login (macOS)

南京理工大学校园网 macOS 后台自动认证脚本，使用 `launchd` 持续运行。**这是基于已在一台 Mac mini 上验证的 V5 版本整理的公开示例，其他网络环境可能需要调整。**

## 功能

- 每 10 秒检测指定网络接口的公网 HTTPS 连通性（期待 HTTP 204）。
- 检测失败且本机 Clash HTTP 代理 `127.0.0.1:7890` 可连接时，调用校园网认证 API。
- 相邻两次自动登录尝试至少间隔 30 秒。
- 借助用户级 `launchd` 在登录 macOS 账户后自动启动，并在异常退出时重启。
- 使用 `caffeinate` 阻止空闲系统睡眠（不阻止显示器关闭；无法在真正系统睡眠、关机或断电期间持续运行）。
- 独立保存监控日志、curl 输出和错误日志。

## 环境要求

- macOS，Bash、`curl`、`launchctl`、`networksetup`、`caffeinate` 等系统工具。
- 校园网接入设备和接口已经配置。默认网络接口为 `en1`，**这并不是所有 Mac 的 Wi-Fi 接口**。
- Clash 或兼容的 HTTP 代理在 `127.0.0.1:7890` 监听并可访问认证域名。
- 具备有效校园网账号与密码；**不要把带真实凭据的脚本上传公开仓库**。

## 使用前配置

编辑 `njust_autologin.sh`：

1. 在 `exact_login_curl()` 的 `--data-raw` JSON 中把 `YOUR_USERNAME` 和 `YOUR_PASSWORD` 替换为本地账号密码。**只在自己电脑上的副本修改，不要提交这一改动。**
2. 检查 `IFACE='en1'` 是否对应目标网络：`networksetup -listallhardwareports`。
3. 如需在 Wi-Fi 完全断开后重新关联，在 `WIFI_SSID=''` 中配置保存过的 Wi-Fi 名称。留空时脚本只处理网络认证。
4. 如 Clash 端口不是 7890，需要自行调整；这会偏离作者原始验证过的 curl 配置。

> 安全提示：该示例使用 HTTP 登录端点，并在 `curl` 命令中包含凭据；存在明文传输及本地泄露风险。发布前请核查 Git 历史、截图、日志和配置文件是否包含密码。公开仓库应始终保持占位符版本。

## 安装与运行

```bash
# 从仓库目录执行：
bash ./njust_autologin.sh --login-once  # 先验证认证请求
bash ./njust_autologin.sh --install     # 安装并启动后台服务
bash ./njust_autologin.sh --status      # 查看服务与近期日志
bash ./njust_autologin.sh --diagnose    # 诊断可能存在的旧监控进程、接口及代理
```

安装后可关闭 Terminal。脚本安装到 `~/.njust-autologin/njust_autologin.sh`，并创建 `~/Library/LaunchAgents/com.njust.autologin.plist`。

注意：**修改仓库中的脚本不会自动更新已经安装的副本**；修改后需重新执行 `--install`。

## 查看日志

```bash
tail -f ~/Library/Logs/NJUSTAutoLogin.v5.log
tail -f ~/Library/Logs/NJUSTAutoLogin.v5.curl.log
tail -f ~/Library/Logs/NJUSTAutoLogin.v5.error.log
```

## 停用

```bash
bash ./njust_autologin.sh --uninstall
```

卸载命令停止本项目的后台服务并删除其 LaunchAgent 配置，但保留日志和安装脚本；不会删除其他校园网自动登录服务。

## 诊断建议

- `--login-once` 可登录、自动登录不触发：检查公网探测条件、`en1` 是否正确、是否有其他网卡承载公网。
- `--status` 没有运行中 PID：检查 `launchd` 配置和错误日志。
- Wi-Fi 本身未连接：校园网 POST 登录命令无法替代 Wi-Fi 连接；需要先连接网络。
- 代理未监听：由于认证请求明确使用 `--proxy http://127.0.0.1:7890`，无法在 Clash 关闭时正常认证。
- 同时运行多个历史脚本：可能产生矛盾日志、重复请求。通过 `--diagnose` 排查并手动停用旧服务。

## 安全发布检查

```bash
# 提交前检查工作区中有没有实际账号、密码、Token：
git diff --cached
# 检查 Git 历史是否曾纳入凭据（如曾上传，应轮换凭据并清理历史）：
git log --all --oneline
```

项目仅供授权的校园网账户使用，请遵守学校网络管理要求。
