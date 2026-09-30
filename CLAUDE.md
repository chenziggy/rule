# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 这个仓库是什么

纯配置仓库，没有代码、没有构建、没有测试、没有 lint。所有文件都是 subconverter（ACL4SSR 风格）的订阅转换配置：

- `*.ini` — 转换器配置，只有 `[custom]` 段
- `*.list` — 规则集，转换时由 subconverter 通过 URL 抓取

仓库通过 `raw.githubusercontent.com/chenziggy/rule/main/` 直接对外提供这些文件，**push 到 main 即生效**，没有构建或发布步骤。

## 两层结构与生效时机

`*.ini` 里的 `ruleset=` 引用的是本仓库自己的 `*.list`：

```
ruleset=🤖 AI,https://raw.githubusercontent.com/chenziggy/rule/main/AI.list
```

由此带来两种不同的生效延迟：

| 改动 | 生效方式 |
|---|---|
| 改 `*.list` | 客户端下次更新订阅时自动拉到（URL 未变） |
| 改 `*.ini` | 需要客户端重新导入/更新该 ini 订阅地址 |
| 新增 `*.list` | 必须同时在该 ini 里加 `ruleset=` 行，否则文件没人引用 |

## ini 格式

只有 `[custom]` 一个段，两类指令：

```
ruleset=<策略组名>,<规则集URL 或 []GEOSITE,CN 这类内联规则>
custom_proxy_group=<组名>`<类型>`<筛选正则>`<测速URL>`<间隔>,,<容差>
```

文件末尾的 `enable_rule_generator=true` / `overwrite_original_rules=true` 必须保留，否则自定义规则不生效。

### 改 ini 的两个硬约束

1. **组名必须逐字符一致，emoji 也算。** `ruleset=` 引用的组名必须能在同一个文件里找到对应的 `custom_proxy_group=`，否则转换报错或规则全部落到默认组。改完务必检查：
   ```bash
   grep -o '^ruleset=[^,]*' ziggy.ini | sed 's/ruleset=//' | sort -u > /tmp/r
   grep -o '^custom_proxy_group=[^`]*' ziggy.ini | sed 's/custom_proxy_group=//' | sort -u > /tmp/g
   comm -23 /tmp/r /tmp/g   # 引用了但没定义的组 —— 必须为空
   ```
2. **顺序即优先级。** Clash 首条匹配生效，`ruleset=` 的排列顺序就是规则顺序。AI/直连这类具体规则要排在 `[]GEOSITE,cn` / `[]FINAL` 之前。

注释用 `;`（`.ini`）和 `#`（`.list`）；临时禁用某行就是行首加 `;`，这是本仓库唯一的"开关"方式——不要删行，历史文件里大量靠注释切换。

## list 格式

标准 Clash 规则行，按 `# 服务名` 分节，节内再分"域名后缀/域名关键字/IP段"：

```
DOMAIN-SUFFIX,example.com
DOMAIN-KEYWORD,example
IP-CIDR,1.2.3.4/32,no-resolve
```

域名类尽量用 `DOMAIN-SUFFIX`（占 95%），`IP-CIDR` 一律带 `no-resolve`，避免 DNS 泄漏和解析延迟。

## 各 list 的归属

| 文件 | 挂到的策略组 | 用途 |
|---|---|---|
| `Direct.list` | 🎯 全球直连 | 国内直连补充，按厂商分节，最大（~550 行） |
| `ProxyLite.list` | 🚀 节点选择 | 代理补充：自建站、国外 DNS、开发/求职类站点 |
| `AI.list` | 🤖 AI | OpenAI / Gemini / Anthropic / Cursor |
| `ProxyAppend.list` | 🚀 节点选择 | 临时追加的少量域名 |
| `ProxyMedia.list` | 🌍 国外媒体 | 自建流媒体域名 |

## ini 变体

同一套 list 被多个 ini 变体复用，选哪个取决于客户端订阅的是哪个 URL：

- `ziggy.ini` — 主力版本，策略组最全（Twitter/GitHub/Telegram/YouTube/Apple 各有独立组，Netflix/HBO 走 🌍 国外媒体）
- `ziggylite.ini` — 精简版（19 组）：把上面那些独立组全部并入 `🚀 节点选择`，注释掉 ChinaDomain/GEOSITE,CN 等大块国内规则，同时打开广告拦截组
- `ziggy.back.ini`、`ziggylite copy.ini` — 手工备份，内容与主文件已有分歧，**不要当作当前版本参考**
- `qichiyu.ini` — 上游 qichiyuhub 的原始模板，仅供对照，改规则时不要动它

改策略组/规则结构时，先确认改的是哪个变体，别只改 `ziggy.ini` 就以为生效了。

## 其他

- 本仓库的 list 只写规则，**节点信息不在仓库里**。
- 仓库里混用 `/main/` 和 `/refs/heads/main/` 两种 raw URL 写法，效果相同，新增时跟随该文件已有的写法即可。
