# clash-ai-rules

自维护的 OpenClash / mihomo AI 域名分流清单（`behavior: domain` 规则集）。

## 为什么自维护

上游清单（`liandu2024/clash` 的 `AI.list`，经 `api.asailor.org` 转换）**漏了 `claude.com`**，
导致 `platform.claude.com`、`claude.com` 落到默认组走了香港节点 —— Anthropic 不支持香港，
表现为 Claude Code 全新安装首启预检 403、浏览器出 app-unavailable-in-region 页。
（完整排查见 `~/Documents/Claude域名分流修复记录-2026-10-16.md`）

本仓库 = 上游快照 + 自维护增补，作为唯一事实源。

## 文件

| 文件 | 说明 |
|---|---|
| `AI.list` | 规则集本体（mihomo domain behavior，`+.domain` = 后缀匹配） |

## 接入路由器（两种，二选一）

### 路线 A：经 asailor 中转（改动最小，路由器直接"从本库取"）

把本库 raw URL 做 base64，替换 provider 的 url 参数：

```bash
# 在电脑上生成（OpenWrt 没有 base64 命令）
echo -n 'https://raw.githubusercontent.com/<你的用户名>/clash-ai-rules/main/AI.list' | base64 -w0
```

然后改 `/etc/openclash/config/merged.yaml` 中 `AI (Domain)` provider 的：

```yaml
url: "https://api.asailor.org/getruleset?type=3&url=<上面的base64>"
```

> 注意：仍需依赖 asailor 服务存活（现状本来就在依赖它）。

### 路线 B：内网同步（零外网依赖，最稳）

路由器改用 `type: file` provider，由 192.168.9.102 定期把本库同步到路由器：

```yaml
rule-providers:
  AI (Domain):
    type: file
    behavior: domain
    path: "./rule_provider/ai-custom.yaml"
    format: yaml
```

同步脚本（192.168.9.102）：

```bash
cd ~/clash-ai-rules && git pull -q
scp AI.list root@192.168.9.1:/etc/openclash/rule_provider/ai-custom.yaml
# 通过 mihomo API 热刷新（无需重启 OpenClash）
curl -s -X PUT -H "Authorization: Bearer <路由器root密码>" \
  http://192.168.9.1:9090/providers/rules/AI%20%28Domain%29
```

## 维护注意

- **不要删** `+.anthropic.com` / `+.claude.ai` / `+.claude.com` 三条 —— 删了 Claude 全线走香港。
- 新增 AI 服务（新模型网页版等）：查清它加载哪些域名，按 `+.domain` 格式加进 `AI.list`。
- 路由器的本地兜底规则（merged.yaml rules 顶部的 3 条 `DOMAIN-SUFFIX`）保持不动，是本清单之外的第二道保险。
