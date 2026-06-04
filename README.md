# Figma Interaction Prototype

把你的 Figma 交互稿(设计稿 + 箭头 + 标注)变成可点击演示的 HTML demo,丢给客户/同事**双击就能打开**,不需要服务器、不需要 npm install。

受众:**交互设计师**(不一定会写代码)。

## 安装

需要 Claude Code(或其他支持 Skill 的客户端)。

```bash
# 用 git 克隆到 Claude Code 的 skills 目录
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/ricoocuii-source/figma-interaction-prototype.git
```

装完后,你在 Claude Code 里输入 `/figma-interaction-prototype` 就能触发。

## 它能干嘛

- 你给一个 Figma 链接 → 出来一个能交互的 HTML demo
- 自动套 iPhone Air 框(移动端)/ 原尺寸(网页端)
- 自动下载你 Figma 里的 icon、图片、二维码等 asset 到本地(再古老的组件库都行,只要 Figma 渲染出来了就能搬)
- 自动从 Figma 取**真实文案**,不会用占位文字
- 自动识别**箭头和标注**(包括"无论何时退出就弹这个 modal"这种文字说明)
- 一致性巡检:同一个红色用了 5 个不同 hex / 圆角忽 8 忽 10 等手抖问题,自动按多数派合并

## 你需要准备什么

1. **Claude Code** + **Figma MCP** 装好,且能正常连接(你之前已经用过)
2. **Chrome** 装在标准位置(`/Applications/Google Chrome.app/`)— 用来截图自检
3. **Figma 稿**整理一下:
   - 主要交互流程的 frame 都在(无论命名规范不规范都行)
   - 箭头标记好 page-to-page 跳转(connector 工具画的)
   - 拿不准的状态/分支用文字标注(比如"loading 状态" / "无论何时退出都弹这个")

## 怎么用

打开 Claude Code,丢一句:

```
帮我用 figma-interaction-prototype 实现这个 demo:
https://www.figma.com/design/<fileKey>/<fileName>?node-id=<入口 node>
```

Claude 会按 5 个 Step 跑:

1. **摸底**:扫所有 frame + 箭头 + 标注,告诉你"找到 N 个 frame"
2. **抽 token**:发现颜色/字号/圆角偏差,小的(差 ≤ 4px)自动合并,大的弹选项卡问你
3. **整理交互清单**:把页面流和浮层组件列出来,拿不准的问你
4. **实现 + 接交互**:每个 frame 单独处理,asset 自动下载存档
5. **自检对照**:自己派 subagent 跟 Figma 截图对比,严重问题报给你

**全程除了少数关键决策外,不打断你**。

## 产物长啥样

```
我的项目/
├── index.html          ← 双击打开就能交互
└── assets/
    └── icons/
        ├── icon-back.svg
        ├── icon-edit.svg
        └── ...
```

- HTML 是独立文件,可以丢 iCloud / Dropbox / 微信文件给同事/客户
- 不需要服务器,不需要安装东西,**双击就能看**

## 它做不到的

- 复杂动画 / 视频 / 3D 效果
- 真原生交互(Apple Pay、指纹、相机权限弹窗这种)
- 像素级 100% 1:1(目标是 90%+,够给客户看效果)
- 自适应桌面端布局(只做设计稿里那个尺寸)

## 出问题了怎么办

| 问题 | 解决 |
|---|---|
| Figma MCP 连不上 | 桌面 Figma 客户端打开,在 Claude Code 重新连一下 MCP |
| 跑到一半中断 | 直接跟 Claude 说"继续",它从上次断的地方接着跑 |
| icon 看着不对 / 还原度不够 | 跟 Claude 说"重新跑 Step 5 自检",或具体指哪块不对 |
| asset 没下载下来 | Claude 会告诉你哪个 asset 失败,可以让它重试 |

## 设计稿里要注意

为了让 AI 抓得准,**建议(不强求)**:

- 箭头用 Figma 自带的 connector 工具画(start/end 端点能识别)
- 重要标注用文字框,**放在节点旁边**(便于 AI 关联到对应组件)
- 命名规范:frame 名能体现"是什么页",比如"职位详情"、"分享海报"
- 浮层组件(modal、action sheet)单独画在画布外,旁边写清楚"什么时候触发"

但即使你完全不规范也行 —— AI 会扫所有节点,拿不准的会问你。

## 反馈和迭代

这个 Skill 是从真实项目踩坑总结来的,有 13+1 条"铁律"(写在 SKILL.md 里,1 条总纲 + 13 条具体),都是**血泪教训**。每跑一个新项目,踩到新坑就追加。

如果你在用的过程中又踩了新的坑,欢迎补充进去。

---

**版本**:v1.0(2026-06-03)
**作者**:Rico
**许可**:MIT
