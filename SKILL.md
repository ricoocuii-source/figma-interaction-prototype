---
name: figma-interaction-prototype
description: 把 Figma 交互稿(含设计稿 + 箭头 + 标注)实现成可演示的 HTML demo。用户是交互设计师,目的是让客户/团队动态体验静态交互稿,而不是产出工程代码。
---

# Figma Interaction Prototype

把 Figma 链接 → 可交互 HTML demo。受众是**交互设计师**,产物是给客户/同事演示用的、双击就能打开的本地 HTML 文件。

---

## 何时触发

用户给一个 Figma 设计稿链接,并且:
- 让你"实现"、"做成 demo"、"做个原型"、"做成网页/页面"
- 或链接里明显包含交互流程(多 frame + 箭头 + 标注)

不适用:工程级别的 production 代码、组件库交付、视觉走查。

---

## 核心铁律(本次踩坑提炼,违反即翻车)

按重要度从高到低:

### 0️⃣ 总纲:**遇到任何视觉元素,做之前先查 Figma 数据,不凭印象/经验/默认值**

**这是治根的元规则**,后面所有铁律都是这条的具体落地。

LLM 有个根深蒂固的坏习惯:**用训练知识填空**。看到状态栏想 44px、看到 iOS 按钮想圆角 8、看到 Action Sheet 想"上面圆角 16 下面 0"。这些"经验"在 Figma 设计稿面前**全是噪音**。

**强制流程**:
- 看到任何元素前问自己:"我是凭印象做的吗?"
- 如果是 → 立刻停。`get_design_context` 拿真实数据再做
- 不知道某个细节是从哪取的 → 停。不要"觉得它差不多这样"

适用范围:所有视觉元素 —— icon、文字、布局、间距、颜色、动效、状态机、组件结构。**没有例外**。

### 1️⃣ 设计稿就是真理,不预设标准库

Figma 里的 action sheet 可能是抄 Ant Design 的、Modal 可能是设计师手画的、icon 风格可能 mix-match。

**不要套 iOS 标准库 / Ant Design / 任何外部预设**。一切按 Figma 视觉复刻。

动效有 3 个来源,按优先级:
- ① Figma Prototype 里设置的 Smart Animate / Transition
- ② 交互说明文字里写的(如"上滑出现")
- ③ 按组件类型套通用默认值:bottom sheet=上滑、modal=渐入、page=push、alert=scale+fade

**通用默认值不带"iOS"标签**,只是合理的物理 fallback。

### 2️⃣ 文案/数据/asset 100% 取 Figma,严禁脑补

**反例(这次踩过的坑)**:
- 海报主标题应该是 Figma 里的招聘 slogan,我从职位详情页搬了某个部门名上去 → 全错
- 加了个红色"高新急招" badge,Figma 根本没有 → 多余元素
- 海报插画用了 Unsplash 团队合影,Figma 是定制 TEAM 卡通插画 → 视觉完全不同

**铁律**:用户没编辑过的所有文字 / 颜色 / 图片,必须来自 `get_design_context` 返回的 text 节点和 image asset。**禁止从职位详情页搬到海报、禁止用 Unsplash 占位、禁止脑补 badge / icon 形状**。

### 3️⃣ icon 必须用 Figma asset,禁止手画 SVG

**反例**:我手画的 SVG 编辑图标是"铅笔+斜线",Figma 真实是"铅笔+纸张+横线";手画的微信图标是"一个气泡",Figma 是"双气泡+笑脸圆点+大气泡"。

**铁律**:`get_design_context` 返回的每个 `imgVector* / imgImage* / img` URL 都要 `curl` 下载存档到本地 `assets/icons/`,**7 天 URL 失效前必须存好**。

### 4️⃣ icon SVG 必须按 viewBox 比例渲染

**反例**:icon-back.svg 的 viewBox 是 `0 0 8.86 15.71`(瘦长 chevron 比例 1:1.77),我用 `<img width=22 height=22>` → 因为 SVG 有 `preserveAspectRatio="none"` 被强制拉成方形 → chevron 变胖,用户一眼指出"变形了"。

**铁律**:
- 先 `cat icon.svg` 或 head 拿 viewBox
- `<img>` 的 width/height **必须跟 viewBox 比例一致**,否则会被 `preserveAspectRatio="none"` 拉伸变形

### 5️⃣ icon 容器尺寸 ≠ vector 实际可见尺寸

**反例**:Figma 编辑按钮 icon 数据是 `size-[16px]` 容器,但内层 vector `inset-[2.68px]`(留 2.68 边距),所以**可见 vector 只占 10.64×10.64**。我直接 16×16 渲染 SVG → icon 看着大一圈,跟文字间距视觉错位。

**铁律**:看 design_context 时,`size-[N]` 是"占位容器",真实 vector 大小要看内层 `inset-[X]`。如需精确还原,用 16×16 的 span 包一层,内 img 设为 10.64×10.64 居中。

### 6️⃣ 自检 prompt 必须给精确测量任务,不能问"对不对"

**反例**:第一轮 subagent 验收 prompt 写"逐项检查并报告差异",subagent 回"对得上",但用户一眼看出 3 处不对(返回箭头变形、编辑按钮 padding 错、底部图标尺寸错)。

**铁律**:自检 prompt 必须给**具体数值/比例**:
- ❌ "返回箭头形状对不对"
- ✅ "返回箭头是不是窄长 chevron,高:宽 ≈ 16:9?"

- ❌ "图标大小对不对"
- ✅ "下载图标是不是 32×32,微信图标是不是 31.5×25.4(扁的)?"

### 7️⃣ 布局位置/间距按 Figma 绝对数值,不用 flex 自动分布偷懒

**反例**:Figma 底部三个分享按钮的 x 坐标分别是 77 / 163 / 249,**相邻 gap 38px,整组居中**。我用 `justify-content: space-around` 偷懒,被 flex 算法等分容器宽度 → 三个按钮间距比 Figma 大,看着散。

**铁律**:Figma `data-node-id` 节点的 `left/top/width/height` **都是设计师精确调过的数值**。不要用 `space-around / space-between / space-evenly` 替代。

正确做法:
- **绝对定位**:用 `position:absolute` + `left/top` 按 Figma 数值
- **flex 容器**:用 `gap` 固定数值,搭配 `justify-content:center / flex-start`,不用 space-* 系列

只有当 Figma 数据明确显示"等距分布"(连续 N 个元素 gap 相等)且**等距值本身就是设计意图**时,space-* 才可考虑。否则一律按数值。

### 8️⃣ subagent 报告完,主智能体自己也要复核

**反例**:第二轮 subagent 在 460×920 小尺寸截图下,说"微信图标比例没扁"、"返回箭头还像 '<' 字符" — 但我自己跑 900×1900 高分辨率截图,看到两个都改对了。Subagent 在低分辨率下判断失准。

**铁律**:subagent 报告完,主智能体用 `--force-device-scale-factor=2 --window-size=900,1900` 自己跑一次高 DPI 截图,Read 图自己看一遍。Subagent 不能 rubber-stamp。

### 9️⃣ `<img>` 加载 SVG 时**必须显式 width + height**,不能用 `width:auto`

**反例**:状态栏 symbols svg viewBox 是 `0 0 68 13`(横长比例 ~5.2:1)。我用 `<img height="13" width="auto">` → 因为 SVG 是 `preserveAspectRatio="none"`,浏览器**无法从 viewBox 推算 intrinsic 比例**,`width:auto` fallback 到容器 100% 宽 → symbols 被横拉到整个状态栏宽度,严重变形。

**铁律**:对 Figma 导出的 SVG(全部带 `preserveAspectRatio="none"`),`<img>` 必须设**双向显式尺寸**:`width:Npx; height:Mpx`,且 N:M = viewBox 比例。**绝对禁止 `width:auto`** — 即使父容器宽度看似合理。

铁律 4 讲"width/height 按 viewBox 比例",这条是它的具体落地:**不只是比例对,数值也必须**。

### 1️⃣1️⃣ Figma node 坐标超 frame 边界 → 实际不渲染,不要做

**反例**:Figma 5758:1214(职位详情,frame 高 812)里有个 node "职位分享面板"(5758:1308): `top:805 height:209` — 但 805+209=1014 已超 frame 812 边界。我看到 design_context 里有这个 node,自作主张做成了底部 bottom sheet 弹出动效。**实际 Figma 里这个 node 完全不渲染**(被 frame clip 掉了),可能是作者临时画的备用素材。用户问"我设计稿里哪有这个啊?"

**铁律**:抓 `get_design_context` 后,**对每个直接 child 检查 `top + height ≤ frame.height` 且 `left + width ≤ frame.width`**。超出边界的 node **一律跳过,不要实现**(它们在 Figma 里实际不可见)。

如果不确定,**用 `get_screenshot` 拉 frame 的真实截图自检** — 看不到的东西就别做。

### 1️⃣2️⃣ `position:absolute` 元素的坐标参考系必须是 `.page`(代表 frame 的容器)

**反例**:`.picker-rail` 放在 `.poster-canvas-wrap` 里,wrap 是 `position:relative`(为了让 wrap 内别的元素能 absolute 定位)。我以为 picker 的 `top:638px` 是相对 `.page`,但 absolute 的参考系是**最近的 positioned ancestor** → 实际是相对 wrap。wrap 顶部从 nav 下方起(y≈98),所以 picker 实际位置 = 98+638 = **736px**,直接砸在分享栏(y=716+)上。用户两次截图指出"picker 跟分享栏重叠了"。

**铁律**:Figma 数据里的 `top/left` 是相对 frame 顶部的全局坐标 → HTML 里 `position:absolute` 元素的 **positioned ancestor 必须就是代表 frame 的容器**(通常是 `.page` 或 `.screen`)。

**写之前检查 DOM 路径**:
- 找该元素到 `.page` 之间的所有祖先
- 任何中间层有 `position:relative/absolute/sticky/fixed` 都会重置参考系
- 不对就**把元素挪出去**,或把那个 ancestor 改成 `position:static`

### 1️⃣3️⃣ asset UUID 严格按 design_context 里的 const 名映射,禁止"看起来差不多"复用

**反例**:Figma design_context 里有两个不同的 UUID:
- `imgFrame = c4f67419-...` → "预览效果" 图标(职位详情底部)
- `imgVector = 360bc0e2-...` → 微信好友 logo

我下载 asset 时偷懒,把 `c4f67419` 同时命名为 `icon-preview.svg` 和 `icon-share-wechat.svg`(因为两个都"看起来像图标"),结果微信好友圆里渲染的是预览图标 → 用户指出"微信图标不对"。

**铁律**:`get_design_context` 返回的每个 `const xxx = "<UUID>"` **就是一个独立 asset**。下载时:
- **一个 UUID → 一个文件**,名字按 const 语义命名
- 禁止"两个 const 看起来差不多,我用同一个 UUID 覆盖"
- 写一个映射表(注释)放在代码里,UUID 一一对应文件名

### 🔟 iOS 屏幕的全局 y 坐标按 Figma 实际数据,不凭"iPhone 真实物理"印象

**反例**:Figma 状态栏 height 44(设计基线值),但我凭"iPhone Air 实际状态栏 ~54px"印象设了 54,连带把 nav-bar 整体下移 10px → nav-back 中心 y=76 vs 分段器中心 y=64,**视觉错位 12px**。

**铁律**:Figma 设计稿是真理(铁律 0 的延伸)。常见容易"凭印象"的 iOS 数值:
- 状态栏:Figma 普遍 44,但实际 iPhone 14/15/Air 是 54。**按 Figma**
- 安全区:Figma 通常按 375×812(iPhone X 基线),实际 iPhone Air 是 393×852。**按 Figma**
- Home Indicator:Figma 通常 34 高,实际 iOS 34。**碰巧一致**
- Dynamic Island:Figma 多数稿不画,**画了就按 Figma**

**反向也适用**:不要"我知道 iOS 是这样的,所以这里 Figma 错了" — 即使 Figma 跟物理设备不符,**Figma 是设计意图,你按它实现就对**。

---

## 平台识别(影响输出形态)

按 Figma frame 尺寸自动判断:

| 宽度 | 平台 | 渲染策略 |
|---|---|---|
| 375 / 390 / 393 / 414 | 移动端 | 套 iPhone Air mockup(参考下面"iPhone Air 框结构") |
| 768 / 1024 | 平板 | 套 iPad mockup |
| 1280 / 1440 / 1920 | 网页端 | **按设计稿原尺寸,不做自适应** |

网页端**不做响应式**。设计稿是 1440 就是 1440,缩放查看交给浏览器。

### iPhone Air 框结构

移动端时用这个尺寸:
```css
.phone {
  --w: 393px;
  --h: 852px;
  --bezel: 9px;
  border-radius: 62px;
  flex-shrink: 0;  /* 关键:防止 flex 容器把手机压扁 */
}
.screen {
  border-radius: 55px;
}
.island {  /* Dynamic Island */
  width: 114px; height: 32px;
  top: 11px;
  border-radius: 20px;
  background: #000;
}
```

状态栏 9:41 + 信号 wifi 电池,Home Indicator 底部白条 135×5。

---

## SOP 主流程

### Step 0: 摸底 + 平台识别

```
mcp__plugin_figma_figma__get_metadata(fileKey, nodeId)
```

输出报告(给用户看):
```
[Step 0/5] 扫描中...
  ✓ 找到 N 个 frame
  ✓ 找到 M 条 connector(箭头)
  ✓ 找到 K 个游离 text 节点(交互说明)
  ✓ 识别平台:移动端/网页端(根据 frame 宽度)
  ✓ 入口节点:<node-name>
```

如果范围有歧义(比如多个看起来都像入口),**用 AskUserQuestion 打断确认**。否则直接进 Step 1。

### Step 1: 抽 token + 一致性巡检

```
mcp__plugin_figma_figma__get_variable_defs(fileKey, nodeId)  # 关键 frame 都拉一遍
```

**自动合并规则**(违反不问,自己判断):
- 数值类(圆角/padding/字号/icon size/边框宽度/行高)差 ≤ 4px → 用频次最高的多数派
- 颜色 hex 同色系略差(如 #989898 vs #9A9A9A)→ 多数派
- 字重 Regular/Medium/Semibold 在同类元素混用 → 多数派

**必须问**(用 AskUserQuestion 弹选项卡):
- 数值差 > 4px(圆角 8 vs 16,差太多明显不是手抖)
- 颜色完全不同色系(蓝 vs 绿)
- 主按钮、主标题等关键元素的样式冲突
- 同类元素 3 个值各占 1/3,没法多数派

**例外**:辅助节点(箭头标注、流程说明文字、备注)的不一致 → **直接跳过,不算冲突**。

### Step 2: 交互规则清单 + 动效推断

扫四类"游离"信息:

| 形态 | 怎么识别 | 例子 |
|---|---|---|
| 游离 text 节点 | 在 frame 外、独立的文字 | "无论何时,尝试退出分享页,则出现此对话框" |
| connector 箭头 | 类型为 `connector`,有 start/end 端点 | 职位详情 →(箭头)→ 海报页 |
| 挂在外的浮层 | 独立 instance,旁边可能有说明 | Modal/Caption 单独画在画布旁 |
| 状态/分支标签 | 名字含"状态/分支/成功/失败/loading" | "Loading 状态" "失败态" |

整理成"**交互规则清单**":
```
[页面流]
  1. A 页 → 点 X → B 页(push)
  2. B 页 → 点 Y → C 浮层(bottom sheet,上滑)
  3. C 浮层 → 点 Z → D 浮层(modal,scale+fade)
  ★规则:无论何时尝试关闭 B 都触发 D

[挂在外的浮层组件]
  - Modal/Caption(节点 X:Y):退出确认
  - Action Sheet(节点 ...):3 个按钮

[拿不准的标注]
  - 节点 N 旁写着"延时 800ms",哪个场景的延迟?
```

**拿不准的用 AskUserQuestion 打断问**。其他直接进 Step 3。

### Step 3: 逐 frame 实现

每个 frame:

```
mcp__plugin_figma_figma__get_design_context(fileKey, frameNodeId)
```

子任务清单(每个 frame 都跑一遍):

**3.1 抓 asset(铁律 3)**
- 遍历返回代码里所有 `imgImage* / imgVector* / img / imgFrame` URL
- `mkdir -p <project>/assets/icons/`
- 用 curl 下载,自动检测 Content-Type,重命名 .svg/.png/.jpg

```bash
ct=$(curl -sS -o "name.tmp" -w "%{content_type}" "$url")
case "$ct" in
  *svg*) mv name.tmp name.svg ;;
  *png*) mv name.tmp name.png ;;
  *jpeg*) mv name.tmp name.jpg ;;
esac
```

**3.2 量 viewBox(铁律 4 / 5)**
- 对每个 SVG `head -c 200 icon.svg`,记下 viewBox
- 看 design_context 里的容器 size 和 vector inset
- HTML 里 `<img width="N" height="M">` 必须按 viewBox 比例
- 必要时用 `<span>` 包一层做容器/vector 分离

**3.3 实现 HTML/CSS**
- 文案逐句从 design_context 的 text 节点取(铁律 2)
- 结构按 design_context 还原层级
- token 按 Step 1 确认的统一基线
- 动效按 Step 2 清单的指定/推断

### Step 4: 接交互流程

按 Step 2 清单,一条一条把页面跳转、弹层显隐接起来。

**ESC 键兼容**:demo 跑在浏览器里,加 ESC = 返回 的处理(便于演示)。

### Step 5: subagent 内部自检对照(铁律 6 / 8)

**5.1 派 subagent 做精确测量对照**

prompt 必须包含**具体测量指令**,不能问"对不对":

```
渲染 HTML 到 PNG:
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" \
    --headless=new --disable-gpu --hide-scrollbars \
    --virtual-time-budget=3000 --window-size=900,1900 \
    --force-device-scale-factor=2 \
    --screenshot=/tmp/render.png \
    "file:///<HTML>?page=<targetFrame>"
    
拉 Figma 截图:
  mcp__plugin_figma_figma__get_screenshot(fileKey, nodeId, maxDimension=1200)

逐项检查(每项给具体数值):
  □ 返回箭头是否窄长 chevron,高:宽 ≈ 16:9?
  □ 编辑按钮 padding 左 8 / 右 10,gap icon 到 text 5px?
  □ 三个底部图标尺寸:32×32 / 31.5×25.4 / 32×32?
  □ 三按钮 gap 38px,整组居中?
  □ ... (列 8-15 个具体测量项)

★ 相邻元素间距强制检查(必加,治"重叠盲区"):
  □ **量出每对相邻元素之间的间距数值(px)**,跟 Figma 数据比对。
     特别强制检查:
     - 海报底/picker 顶 之间多少 px?
     - picker 底/分享栏顶 之间多少 px?
     - 编辑按钮和 picker 之间多少 px?
     - 信息卡底 / 分享栏 / Home Indicator 之间多少 px?
     **任意两个元素如果距离 < 5px 或重叠,立刻报警**。

★ 开放发现题(必加,治"未知盲区"):
  □ **除了上面列的,你看到任何 prompt 没列出但跟 Figma 不一致的元素吗?**
     具体描述差异,哪怕是小的(颜色深浅、字号 1-2px 偏差、阴影、行距、对齐)。
     如果什么都没发现,也明确说"无新增差异"。
```

**为什么加相邻元素间距检查**:本会话连续 2 次踩了"picker 跟分享栏重叠"的坑,我 chrome 截图自检都没看出来——因为我看的是"大局对不对",**没量相邻元素的具体间距**。具体数字比"看起来对不对"靠谱 10 倍。

**为什么加开放发现题**:已知 case 用清单兜底,未知 case 靠 subagent 主动发现。
否则主智能体只能拦"自己想到的问题",**没想到的盲区永远拦不住**。

**5.2 主智能体自己用高分辨率截图复核**(铁律 8)

subagent 报告后,主智能体跑一次:
```bash
--force-device-scale-factor=2 --window-size=900,1900
```

用 Read 工具看 PNG,跟 Figma 原图对照。**Subagent 在低分辨率下可能误判,不能 rubber-stamp**。

**5.3 处理差异**
- 自己能修的自己修(数值类、CSS 调整)
- 修不动的报给用户(标"这处和 Figma 不一致,你看要不要管"),不静默忽略

### Step 6: 加调试 debug hook

`<HTML>?page=<frameName>` 直接跳到任意 frame,跳过加载流程。方便自检截图和后续测试。

```js
if (location.search.indexOf('page=poster') !== -1) {
  posterCard.classList.remove('loading');
  $('#page-job').classList.remove('active');
  $('#page-poster').classList.add('active');
  State.currentPage = 'poster';
  State.history = ['job','poster'];
}
```

---

## 打断节奏总览

只在以下时机打断用户(用 AskUserQuestion):

| 时机 | 问什么 |
|---|---|
| Step 0 末 | 入口/范围/平台有歧义时 |
| Step 1 中 | 大冲突(数值差 > 4px、不同色系、关键元素) |
| Step 2 末 | 拿不准的交互标注 |
| Step 5 末 | 自检发现严重无法修复的差异 |

其他场景全跑通,**不打断**。

---

## 输出物结构

```
<project-name>/
├── index.html          # 主 demo 文件,双击打开
└── assets/
    └── icons/
        ├── icon-back.svg
        ├── icon-edit.svg
        ├── poster-illustration.png
        └── ...
```

**不放 README**(产物是给客户/同事双击看的,他们不会读 README)。

---

## 进度输出格式

每步给用户一句话:

```
[Step 0/5] 扫描 Figma 节点... 找到 12 个 frame, 4 个浮层, 8 条箭头
[Step 1/5] 抽 token... 发现 3 处颜色偏差(差 ≤ 2px,已合多数派)
[Step 2/5] 整理交互清单... 1 处不确定,问你
[Step 3/5] 实现 frame 1/8: 职位详情页...
[Step 3/5] 实现 frame 2/8: 团队主题海报... 下载 9/10 asset
[Step 4/5] 接交互流程...
[Step 5/5] 自检对照...
✓ 完成,产物在 <path>
```

---

## 错误兜底

| 错 | 怎么办 |
|---|---|
| Figma MCP 断了 | 主动提示用户重连,不要静默用记忆继续 |
| asset URL curl 失败 | 立即重试 1 次,仍失败 → fallback 占位 + 在 console 标 TODO,**不静默用错的** |
| design_context 抓不到文案 | 留空 + 警告,**禁止脑补** |
| Chrome 路径不对 | 试 `/Applications/Google Chrome.app/...` → `/Applications/Chromium.app/...` → `which google-chrome` |

---

## 边界(管理预期)

这个 Skill 做不到:
- 复杂动画 / 视频 / 3D 效果
- 原生交互(Apple Pay、生物认证、相机权限弹窗)
- 像素级 1:1(目标 90%+ 还原,但不保证完美)
- 自适应桌面端布局(只做设计稿里那个尺寸)
- Code Connect 到真实代码组件库(产物是独立 HTML,不是工程代码)

---

## 示例

参考 `examples/job-share-flow/`(以本次"职位分享流程"为案例的完整产物)。

---

## 如何让 Skill 持续演化(很重要)

这份 Skill 的 8+1 条铁律是从**有限案例**抽象的,只能拦"原案例的近邻"问题。**远端盲区永远存在**。

提升拦截率靠两个机制:

### 1. 每次踩新坑,把新铁律追加进来

工作流:
```
A. 用户指出问题(比如"这里看着不对")
B. 复盘根因:这是哪类问题?之前的铁律覆盖了吗?
C. 如果没覆盖 → 抽象一条新铁律,追加到本文档"核心铁律"部分
D. 如果是已有铁律的延伸 → 在该铁律下补一个反例
E. 编号继续往后排,不重排已有铁律
```

铁律编号 = 历史层积,体现 Skill 的演化轨迹。

### 2. 每次完成 Step 5,问 subagent 这条:

> "如果重做一遍,Step 5 自检清单要补哪些测量项才能更早抓到这次的问题?"

把 subagent 答案当下次的清单补丁,**自检模板会越用越细**。

---

## 边界声明:这个 Skill 永远不能 100% 替代你

哪怕 8 条铁律变 80 条,Skill 也只能拦"过往踩过的坑的近邻"。**未知盲区永远存在**。

设计师本人**最终把关**永远是必要的 —— Skill 帮你拦 90% 的低级错,**最后 10% 的判断只有你能做**。

把 Skill 当**实习生**用,不要当**外包合同**用。
