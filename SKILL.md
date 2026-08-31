---
name: "rhino-publish"
description: "Distill raw material (text/URL/file) into a structured post and auto-publish it to 洞犀 Rhinoloot site via admin API. Invoke when user asks to 发布/发帖/发到洞犀/发条资讯/发个活动/提炼发布, or wants content published to rhinoloot.pages.dev."
---

# 洞犀一键发布（rhino-publish）

把任意素材（一段文字 / 文章链接 / 本地文件）提炼成结构化内容，通过管理员 API 自动发布到洞犀 https://rhinoloot.pages.dev ，并验证发布结果。

## 工作流（5 步）

### 第 1 步：获取素材
- 用户直接贴文字 → 原文使用
- 给 URL → 先抓取正文再提炼
- 给本地文件路径 → 读取
- 素材缺关键事实（如活动没写截止日期）→ 标注"素材未提供"，**禁止编造**

### 第 2 步：提炼为发布 JSON 并确认
按下方《字段规范》生成完整 JSON，向用户展示确认卡：标题 / 分类 / 标签 / 简介 / 正文预览（前 200 字）。
用户本次或历史消息中明确说过"直接发 / 不用确认" → 跳过确认直接进入第 3 步。

### 第 3 步：鉴权
1. 读取同目录 `auth.local.md`（格式见文末）取 email / password。文件缺失或为空 → 向用户询问一次管理员账号，写入该文件再继续（该文件名匹配 `.gitignore` 的 `*.local.md` 规则，不会被提交）。询问时如果同目录存在 `auth.example.md`，提示用户可自行复制填写。
2. API 登录：
   ```powershell
   [IO.File]::WriteAllText("$env:TEMP\rh_login.json", '{"email":"<email>","password":"<password>"}')
   curl.exe -s -X POST https://rhinoloot.pages.dev/api/auth/login -H "Content-Type: application/json" --data-binary "@$env:TEMP\rh_login.json"
   ```
   成功 → 记下 `.token`（JWT，约 30 天有效），并确认响应 `user.is_admin` 为 1，否则无发布权限。登录后无需缓存 token，每次发布前重新登录即可（一次请求的开销，换来永不过期）。
3. 登录失败（改密等）→ Plan B 浏览器辅助鉴权：调用 TRAE-browseruse 打开 https://rhinoloot.pages.dev/admin ，用 browser_waiting_for_user_interaction 让用户手动登录，完成后在页面执行 `localStorage.getItem('apifree_token')` 取 token，继续第 4 步。

### 多管理员说明
- 本技能随仓库分发，但**凭据不入库**：每位管理员在本机首次使用时，AI 会询问一次账号并写入本机 `auth.local.md`（或手动复制 `auth.example.md` 改名填写），之后免问。
- 一台机器一份凭据；多人共用一台机器时不要共用凭据文件，各自走 Plan B 浏览器辅助鉴权（登录的是谁，就以谁的身份发布）。
- 发布归属：后端会记录 `submitted_by`（发布者 id）与审计日志，谁登录就以谁的账号发布、谁担责。
- 权限差异：admin_level 1 普通管理员可发布/删除；admin_level 2 超级管理员额外可管理管理员列表。发布技能对两者等价。

### 第 4 步：发布
**PowerShell 下 JSON 一律先写入临时文件再 `--data-binary`，禁止用 -d 内联（转义必坏）。推荐 hashtable + ConvertTo-Json：**
```powershell
$body = @{ platform=".."; title=".."; main_cat=".."; sub_cat=".."; description="..";
  content_md=".."; tags=@("标签1"); benefit_type="free_tier"; source_url=".."
  # extra_data = @{ ... }  # api 板块必看下方规则
} | ConvertTo-Json -Depth 5
[IO.File]::WriteAllText("$env:TEMP\rh_pub.json", $body)
$tok = "<token>"
curl.exe -s -X POST https://rhinoloot.pages.dev/api/admin/add -H "Authorization: Bearer $tok" -H "Content-Type: application/json" --data-binary "@$env:TEMP\rh_pub.json"
```
成功响应：`{"success":true,"id":N,"message":"已发布"}`

### 第 5 步：验证与汇报
- `curl.exe -s https://rhinoloot.pages.dev/api/content/<id>` 应返回 `success:true`
- 汇报用户：标题 + 链接 https://rhinoloot.pages.dev/content/<id> + 分类 + 标签
- 发错可撤：`POST /api/admin/delete`，body `{"activity_id":N}`（同一 Bearer token）

## 字段规范（/api/admin/add body）

| 字段 | 必填 | 规范 |
|---|---|---|
| platform | ✅ | 来源平台/厂商名（如 OpenAI、GitHub、量子位） |
| title | ✅ | 中文为主、信息量足，≤50 字 |
| main_cat / sub_cat | ✅ | 见分类映射；后端严格校验，拼错即 400 |
| description | ✅ | 100~160 字中文简介（列表卡片与 ai_summary 兜底都用它） |
| content_md | | Markdown 正文，≤50000 字。支持：标题/粗斜体/列表/引用/行内代码/代码块/表格/删除线/链接/图片。站内链接用 `/content/<id>`，本站图片 `/media/...` 均可渲染 |
| tags | | 字符串数组，**只能从白名单选**——后端会静默丢弃白名单外的标签。选 1~4 个最贴切的 |
| benefit_type | | api 板块：短期→token、长期→credit；其余板块→free_tier |
| benefit_amount | | 如 "100万 Token"、"$5" |
| ai_summary | | 不填则自动取 description 前 160 字；按下方推荐格式生成更佳 |
| end_date | | YYYY-MM-DD，无则省略 |
| source_url | | 原文链接 |
| is_llm_api | | api 板块且为 LLM API 时传 true |
| extra_data | | 见下方规则 |

### 分类映射
| main_cat | 板块 | sub_cat |
|---|---|---|
| events | 活动概览 | hackathon 黑客松 / contest 竞赛 / algorithm 算法赛 / aigc AIGC 赛 / other 其他 |
| products | 产品推荐 | hot 热门 / featured 精选 |
| api | API 福利 | long 长期 / short 短期 |
| news | AI 资讯 | release 模型发布 / industry 行业动态 |
| cases | 实践案例 | practice 实战 / tutorial 教程 |

判定原则：有赛程/截止→events；产品或工具→products；送额度/免费 API→api；新闻/发布/融资→news；教程/经验→cases。两可时在确认卡给用户二选一。

### 标签白名单（按板块，白名单外一律无效）
- events：有奖金、免费参赛、学生可参赛、团队参赛、线上举办、线下举办、国际赛事、国内赛事、提供算力、新手友好、就业机会
- products：免费、开源、图像生成、视频生成、语音合成、多模态、Agent、RAG、微调、代码助手、浏览器插件、桌面应用、移动端、国产
- api：送 Token、免费额度、无需绑卡、新用户专享、学生认证、开源模型、国产模型、国际模型
- news：开源发布、技术突破、融资动态、政策监管、模型评测、行业分析、AI 安全
- cases：入门级、进阶、完整项目、提示词、工作流、本地部署、避坑指南、效率提升

### ai_summary 推荐格式（供 AI 爬虫 GEO 摘要）
`标题：<title>；分类：<板块>/<子类中文名>；平台：<platform>；来源：<source_url>；标签：官方、<tags>；要点：<2~3 个核心事实>`

### extra_data 规则（api 板块必看）
- api/short **必填**：token_amount、model、concurrency、rpm（素材没给就问用户，禁止编造）
- api/long **必填**：benefit_note
- events 可选：reg_start、reg_end、event_start、event_end、organizer、prize_pool、prize_detail、eligibility、custom_tags

## 图片（可选）
本地图片先上传换 URL 再插入 MD：
```powershell
curl.exe -s -X POST https://rhinoloot.pages.dev/api/upload -H "Authorization: Bearer $tok" -H "Content-Type: image/png" --data-binary "@C:\path\img.png"
# → {"success":true,"url":"/media/img/<uid>/<key>.png"}
```
限制：≤1MB（超限 400），支持 png/jpg/webp/gif。在线图片直接贴原 URL。

## 常见错误
| 响应 | 原因 → 处理 |
|---|---|
| 401 未登录 | token 失效 → 回第 3 步重新登录 |
| 403 无权限 | 账号非管理员 → 换管理员账号 |
| 400 请选择正确的分类 | main_cat/sub_cat 拼错 → 对照映射表 |
| 400 平台和标题必填 | 字段遗漏 |
| 发布后标签消失 | 用了白名单外的词 → 只从白名单选 |

## auth.local.md 格式
```markdown
email: someone@example.com
password: xxxxxx
```
首次使用可复制同目录 `auth.example.md` 为 `auth.local.md` 后填写。
