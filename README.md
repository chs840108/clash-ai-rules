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

## 接入方式（2026-10-16 已实施：内网直供）

```
github.com/chs840108/clash-ai-rules      ← 事实源（改这里）
   │  Cron 每 15 分钟 git pull（SSH）
   ▼
192.168.9.102:8787  nginx 容器 clash-rules（只读挂载本目录，restart=always）
   │  路由器 mihomo 每小时拉取（provider interval: 3600）
   ▼
OpenClash 规则 AI (Domain) → 👽 AI1（美国节点）
```

**nginx 容器**（在 192.168.9.102 上，已部署）：

```bash
docker run -d --name clash-rules --restart=always \
  -p 192.168.9.102:8787:80 \
  -v /home/agentadmin/clash-ai-rules:/usr/share/nginx/html:ro \
  192.168.9.9/dockerhub-cache/library/nginx:1.27-alpine
```

**路由器 provider**（`/etc/openclash/config/merged.yaml`）：

```yaml
  AI (Domain):
    type: http
    behavior: domain
    url: "http://192.168.9.102:8787/AI.list"
    interval: 3600
```

**生效延迟**：改库 → ≤15 分钟（git pull）→ ≤1 小时（路由器重拉）。
想立即生效（无需重启）：

```bash
curl -s -X PUT -H "Authorization: Bearer <路由器root密码>" \
  'http://192.168.9.1:9090/providers/rules/AI%20%28Domain%29'
```

### ❌ 已否决：asailor 中转自建库

实测 asailor 对自建库统一返回 `payload: ['--placeholder--']`（只转它认识的已知 URL）；
更危险的是**若服务降级，会把规则集静默替换成占位符，AI 分流悄无声息全废**。
（注：`AI2 (Domain)` 目前仍走 asailor 的已知上游 URL，暂未动。）

## 维护注意

- **不要删** `+.anthropic.com` / `+.claude.ai` / `+.claude.com` 三条 —— 删了 Claude 全线走香港。
- 新增 AI 服务（新模型网页版等）：查清它加载哪些域名，按 `+.domain` 格式加进 `AI.list`。
- 路由器的本地兜底规则（merged.yaml rules 顶部的 3 条 `DOMAIN-SUFFIX`）保持不动，是本清单之外的第二道保险。
