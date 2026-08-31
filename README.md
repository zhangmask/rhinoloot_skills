# rhinoloot_skills

洞犀 Rhinoloot 的 AI 技能集合。

## rhino-publish

一键发布到洞犀：把小红书/知乎/任意链接或文字，提炼后自动投稿。

### 使用方式

直接说你想发什么：

```
我要投稿 https://www.xiaohongshu.com/explore/xxx
```

```
帮我发这条到洞犀 https://openai.com/blog/xxx
```

```
发到洞犀：[粘贴内容]
```

AI 会自动：提取信息 → 询问缺失字段（如官方来源链接）→ 展示确认卡 → 等你确认 → 发布

发布后返回链接：https://rhinoloot.pages.dev/content/<id>

### 文件结构

```
SKILL.md          # Skill 定义（给 AI 看的指令）
auth.example.md   # 凭据模板
README.md         # 本文件
```
