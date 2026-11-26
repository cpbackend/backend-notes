# SECURITY-LAB

本地安全实验笔记：靶场复现步骤、工具用途与合法边界。

> **适用法律与授权声明（先读这一条）**
>
> 本文所有内容**只在本地靶场（localhost / 自建隔离环境）或你有明确书面授权的项目**中使用。
> 对未授权的系统进行扫描、注入、漏洞验证，在大多数司法辖区属于违法行为。
> 本文只记录"怎么在受控环境里复现与验证"，不提供针对第三方的攻击指引。
> 不确定是否在授权范围内时，默认视为**不在**范围内。

## 1. 本地靶场：OWASP Juice Shop 的 DOM XSS

Juice Shop 是 OWASP 的旗舰级**故意留漏洞**的 Web 应用，用于安全培训、CTF 和安全工具测试；官方明确说明演示实例"不供你做 hacking 练习"，所以自己起本地实例。

### 1.1 起一个只在本地监听的实例

用 Docker 时把端口绑定到回环地址，避免暴露到局域网：

```bash
docker run --rm -p 127.0.0.1:3000:3000 bkimminich/juice-shop
# 浏览器打开 http://localhost:3000
```

源码方式：`git clone https://github.com/juice-shop/juice-shop.git --depth 1 && cd juice-shop && npm install && npm start`。

### 1.2 DOM XSS 挑战的官方 payload

Juice Shop 官方配套指南（Pwning OWASP Juice Shop）里，"DOM XSS"挑战（难度 ⭐）的验收 payload 是：

```
<iframe src="javascript:alert(`xss`)">
```

复现路径（对应官方提示）：

1. 找一个"提交后其内容会出现在 HTML 里"的输入框（应用内的搜索框）。
2. 把上面的 payload 填进去并提交。
3. 出现 `alert('xss')` 即挑战通过；关键是看浏览器端 JS 如何把输入写回 DOM，而不是服务器返回。
4. "Bonus Payload" 挑战用同一注入点，换成官方给出的 SoundCloud iframe 载荷，效果更明显但原理相同。

**为什么这是 DOM 型而不是反射/存储型**：判定看"用户输入在哪一步进入可执行上下文"——DOM XSS 是浏览器端脚本取到输入后直接（不安全地）写入 DOM；payload 不会存进数据库，也不出现在服务端响应里。反射/存储型则经过服务器往返。

**验证口径**（写进笔记的结论要有证据）：

```
目标        : http://localhost:3000（本机 127.0.0.1，未对外）
注入点      : 站内搜索框
payload     : <iframe src="javascript:alert(`xss`)">
预期        : 浏览器弹窗 xss；挑战面板 DOM XSS 置为 solved
边界        : 未触及任何非本地系统
```

### 1.3 复现时的安全边界

- 目标地址必须是 `127.0.0.1` / `localhost` 或自己控制的隔离网段；Docker 用 `-p 127.0.0.1:...` 而非 `-p 3000:3000`。
- 用干净浏览器配置 / 无真实 Cookie 的会话；不要带着生产环境登录态访问靶场。
- payload 只用于触发 `alert`/无害标记，不做数据外带、不探测内网、不复用真实凭据。
- 靶场故意有漏洞，**不要**把本地实例端口转发到公网。

## 2. 工具用途与合法边界

三样东西定位不同：XSStrike 是专用 XSS 检测套件，Strix 是通用 AI 渗透测试 agent，Claude-BugHunter 是给 Claude Code 用的漏洞挖掘技能包。共同点：**都只能对准自己的资产或授权目标**。

| 工具 | 用途 | 主要边界 / 注意 |
| --- | --- | --- |
| **XSStrike**（s0md3v/XSStrike，GPLv3） | XSS 检测套件：反射与 DOM XSS 扫描、上下文分析、智能 payload 生成、fuzz、爬虫、WAF 探测与绕过、盲打支持。特点是基于响应解析构造 payload，而不是盲插字典。 | 会主动发请求并尝试绕过 WAF；只对自己拥有或获授权的目标使用。生成的 payload 可能带事件处理器/混淆，仅在靶场验证。 |
| **Strix**（usestrix/strix，Apache-2.0） | 开源 AI 渗透测试 agent：动态运行代码、多 agent 编排、把漏洞以可工作的 PoC 验证、生成修复与报告；可接 GitHub Actions 做 CI 扫描。覆盖 OWASP Top 10（含 stored/reflected/DOM XSS）。 | 官方警告：会**实际测试**指向的目标；仅用于你拥有或"明确书面授权"的系统，并严格限定在约定 scope 内。未授权测试在多数司法辖区违法。`strix view` 默认只绑定 127.0.0.1 并带 token 链接，分享要谨慎。 |
| **Claude-BugHunter**（elementalsouls/Claude-BugHunter） | Claude Code 的技能包（85 个 skill / 16 个命令，含 `hunt-xss` 等），按漏洞类别整理检测模式、payload、绕过表和链式利用模板；含 scope 校验与报告/证据处理流程。也在 DVWA、OWASP Juice Shop 等公开训练平台上使用。 | 面向"你拥有或已获书面授权的资产"（bug bounty 在范围内的目标、pentest 授权书、CTF、自有设施）。内置 7-Question Gate 等范围校验；排除 0day 武器化、post-exploit、恶意软件开发、无差别目标。注意模型侧可能有实时安全策略拦截，授权作业应走正规验证渠道，不要靠改写措辞绕过分类器。 |

### 2.1 工具配置里不要写进仓库的东西

- LLM API key、`LLM_API_KEY`、HackerOne token 等凭据：一律用环境变量，绝不写进本仓库（遵守 README 规则）。
- 真实目标域名 / IP、客户信息、内部地址：不要写进本仓库。
- 需要凭据的步骤只记录"用了哪个环境变量名 + 命令形态"，值留在本地环境。

## 3. 合法使用清单（每次开工前逐条确认）

1. 目标是本地靶场，或我有**书面授权**且目标在授权 scope 内。
2. 测试窗口、允许的技术和禁止的动作都清楚（例如是否允许 DoS、是否允许社会工程）。
3. 只在约定范围内发请求/注入；不顺手扫相邻资产、不做数据外带。
4. PoC 用无害标记（`alert` / 固定字符串），拿到结果即停。
5. 证据做脱敏（Cookie、token、PII 一律打码）后再保存或提交。
6. 发现真实漏洞走正规披露渠道；不公开细节直到对方修复或同意。
7. 有任何不确定，停下来问，而不是"先打了再说"。

## 4. 参考

- Juice Shop 源码与运行方式：<https://github.com/juice-shop/juice-shop>
- 官方配套指南（XSS 章节与 DOM XSS 解法）：<https://pwning.owasp-juice.shop>
- XSStrike：<https://github.com/s0md3v/XSStrike>
- Strix：<https://github.com/usestrix/strix>
- Claude-BugHunter：<https://github.com/elementalsouls/Claude-BugHunter>
