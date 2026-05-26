---
name: script-highlighter
description: 上传剧本(.doc/.docx/.txt)，指定高亮角色及颜色，生成带目录、角色备注、一键回顶、手机适配的台词高亮HTML
---

# 剧本台词高亮工具

将剧本（.doc/.docx/.txt）转换为美观的台词高亮 HTML，支持指定角色颜色、护眼模式、目录章节跳转、场景角色备注、手机阅读适配。

---

## 一、确认剧本文件

检查当前工作区间是否已有剧本文件（.doc / .docx / .txt）。

- **若已有** → 确认文件名，询问用户是否使用该文件
- **若没有** → 指示用户上传剧本文件到工作区间，然后继续

---

## 二、询问配置

用 `ask_choice` 逐项确认：

### 2.1 背景模式

| 选项 | 色值 |
|------|------|
| 护眼模式（豆沙绿） | body `#C7EDCC` / container `#DAECD6` / 正文 `#222222` |
| 默认（茶白色） | body `#F5EDE0` / container `#FFFBF5` / 正文 `#2c2c2c` |
| 自定义 | 用户指定色值 |

### 2.2 高亮角色及颜色

自动扫描剧本中所有有台词的角色，按台词频率排序，以**复选框列表**展示。
用户勾选需要高亮的角色，每个角色可单独调整颜色。默认勾选前 5 个高频角色。

未勾选的角色统一黑色、不加粗。

> 角色名在剧本中可能带空格（如 `若  莲`、`燕  儿`），匹配时需归一化后再比对。

### 2.3 其他角色台词颜色

默认黑色。允许自定义。

### 2.4 场景目录确认（关键步骤）

自动检测场景标题后，进入**场景目录审核页**，用户可以：
- **编辑**：直接修改场景名称输入框
- **删除**：点击场景行尾 `✕` 移除不需要的场景
- **插入**：点击行间 `＋` 按钮，在任意两个场景之间插入新场景
- **新增**：底部输入框添加额外场景

确认无误后点击「确认无误，继续生成」进入最终生成。

---

## 三、生成 HTML

### 3.0 用户交互流程

```
上传剧本 → 配置参数（背景/角色/颜色）→ 点击生成
          ↓
   📋 场景目录审核页
   ├── 编辑/删除/插入场景
   └── 确认后进入生成
          ↓
   ⏳ 生成中 → 📥 下载 / 👁️ 预览（Blob URL 锚点可跳转）
```

使用前端 HTML/JS 完成（非 Python 脚本），依赖 JSZip 解析 docx。

### 3.1 文本提取

| 格式 | 方法 |
|------|------|
| `.doc` | OLE `WordDocument` 流，UTF-16LE 解码 |
| `.docx` | `python-docx` |
| `.txt` | UTF-8 直接读取 |

统一 `\r\n` → `\n`。找到最后一个 `谢幕` 并在此截断，丢弃尾部字体表乱码。

**字符保留策略：**
- CJK 汉字（`\u4e00-\u9fff`）
- CJK 符号（`\u3000-\u303f`，含 `《》、【】` 等）
- 全角符号（`\uff00-\uffef`）
- 阿拉伯数字 `0-9`
- 常用标点 `，。！？、：（）《》—…·"''「」『』`
- 换行符与空格

### 3.2 场景识别

匹配：`^第.+场|^序幕|^[春夏秋冬]$|^谢幕`，自动去重。

### 3.3 台词角色分类

按用户指定的角色名匹配，追加对应的 CSS class。未指定的归入 `other`。

**角色名过滤（禁止混入非角色内容）**：以下脚本标记不作为角色名出现在目录中——`幕间`、`幕间过场`、`春 尾声`、`冬 第四幕` 等非角色 `<span class="char-name">` 内容，必须跳过。

### 3.4 目录角色提取

每两个相邻场景标题之间的正文段落，提取所有 `<span class="char-name">` 内容，过滤上述非角色标记，去重后按固定顺序排列：

```
若莲、老爷、大太太、二太太、三太太、燕儿、管家、曹二婶、高医生、飞浦、丫鬟、五太太、梅芯
```

不在上述顺序中的角色按字母序排列在末尾。无台词的场景标注 `（无台词）`。

### 3.5 HTML 结构

```
┌──────────────────────────────┐
│  🏮 {剧本名}                  │
│  台词高亮版 · 角色颜色对照     │
├──────────────────────────────┤
│  ■ 燕儿  ■ 老爷  ■ 高医生 …  │  ← 图例
├──────────────────────────────┤
│  📖 目 录                      │
│  序幕：离家            — 若莲  │  ← 竖排，每行一个
│  第一场：初入陈府  — 若莲、燕儿│
│  ……                          │
├──────────────────────────────┤
│  序幕：离家  ↑ 顶部            │  ← 每个场景标题带回顶
│  【舞台指示】                  │
│  角色名：台词内容              │
│  ……                          │
├──────────────────────────────┤
│        ⬆ 回到顶部              │  ← 底部按钮
│  🎭 根据《…》生成              │
└──────────────────────────────┘
```

### 3.6 各元素格式

**场景标题（带 ID 和回顶链接）**
```html
<div class="scene-title" id="scene-N">场景名 <a class="to-top" href="#">↑ 顶部</a></div>
```

**对话行**
```html
<div class="dialogue {class}">
  <span class="char-name">角色名</span>
  <span class="action-parens">（动作说明）</span>
  <span class="char-sep">：</span>
  <span class="char-text">台词内容</span>
</div>
```

**目录条目**
```html
<li>
  <a href="#scene-N">
    <span class="toc-scene">场景名</span>
    <span class="toc-chars">— 角色1、角色2</span>
  </a>
</li>
```

### 3.7 间接称谓高亮（Inferred Highlight）

当高亮角色的名字在**舞台指示（`【】`）**中被提及时（无论是否带动作说明），该名字应以该角色高亮颜色的**浅色阶**显示，且**加粗**。

例如，高亮角色为「燕儿（紫色 #7C3AED）」：

```html
<!-- 处理前 -->
<div class="stage-dir">【燕儿暗戳戳偷窥着两人</div>

<!-- 处理后 -->
<div class="stage-dir">【<span class="ref-yaner">燕儿</span>暗戳戳偷窥着两人</div>
```

```css
.ref-yaner { color: #A78BFA; font-weight: 700; }   /* 浅紫色，比 #7C3AED 淡一阶 */
```

**色阶对照表**（供参考，可根据视觉效果微调）：

| 角色 | 台词高亮色 | 提及浅色 |
|------|-----------|---------|
| 燕儿 | `#7C3AED`（紫） | `#A78BFA`（浅紫） |
| 老爷 | `#B8860B`（黄） | `#D4A843`（浅黄） |
| 高医生 | `#E67E22`（橙） | `#EAA85C`（浅橙） |
| 三太太 | `#DC2626`（红） | `#F87171`（浅红） |

**实现要点：**
1. 仅对 `<div class="stage-dir">` 内的文本进行替换
2. 角色名可能带空格（如 `燕  儿`），正则需归一化 `燕\s*儿`
3. 不替换 `<span class="char-name">` 内的名字（已由对话高亮处理）
4. 不替换 `<span class="char-text">` 中其他角色台词里提及的名字（属于他人台词，保持原色）
5. 如有多个高亮角色，需轮询匹配

### 3.8 完整 CSS 规范

#### 基础色调

| 用途 | 护眼模式 | 默认模式 |
|------|----------|----------|
| 页面底色 `body` | `#C7EDCC`（豆沙绿） | `#F5EDE0`（茶白） |
| 卡片底色 `.container` | `#DAECD6` | `#FFFBF5` |
| 正文色 | `#222222` | `#2c2c2c` |
| 标题色 `h1` | `#6B3410` | `#8B4513` |
| 场景标题 `.scene-title` | `#6B3410` / 边框 `#B8D0B4` | `#8B4513` / 边框 `#E8DDD0` |
| 图例/目录底色 | `#C0DFB8` | `#F5EDE0` |

> 自定义模式使用用户指定的色值。

#### 字体排版

```css
font-family: 'Noto Serif SC', 'SimSun', 'STSong', 'Songti SC', serif;
line-height: 2.0;
font-size: 17px;        /* 正文 */
.scene-title { font-size: 18px; font-weight: 700; }
.stage-dir   { font-size: 14px; color: #6a7a6a; }
.toc-list li { font-size: 14px; }
.toc-chars   { font-size: 13px; color: #4a7a4a; }
.back-to-top { font-size: 15px; font-weight: 700; }
```

#### 高亮角色

```css
.yaner .char-name,
.yaner .char-text { color: #7C3AED; font-weight: 700; }

.laoye .char-name,
.laoye .char-text { color: #B8860B; font-weight: 700; }

/* 以此类推…… */
```

#### 间接称谓高亮（Inferred Highlight）

```css
.ref-yaner { color: #A78BFA; font-weight: 700; }       /* 燕儿被提及时 */
.ref-laoye { color: #D4A843; font-weight: 700; }       /* 老爷被提及时 */
.ref-gao { color: #EAA85C; font-weight: 700; }         /* 高医生被提及时 */
.ref-santaitai { color: #F87171; font-weight: 700; }   /* 三太太被提及时 */
```

#### 高亮角色以外的角色

```css
.other .char-name,
.other .char-text { color: #000000; font-weight: 400; }
```

#### 目录（TOC）

```css
.toc-list { list-style: none; padding: 0; margin: 0; }
.toc-list li {
  padding: 5px 0;
  border-bottom: 1px dotted #B8D0B4;
  line-height: 1.6;
}
.toc-list li:last-child { border-bottom: none; }
.toc-list a {
  color: #2a5a2a; text-decoration: none;
  padding: 3px 8px; border-radius: 4px;
  display: block; transition: background 0.2s;
}
.toc-list a:hover { background: #DAECD6; }
.toc-scene { font-weight: 700; }
.toc-chars { font-weight: 400; color: #4a7a4a; font-size: 13px; }
```

#### 回顶按钮

```css
.to-top {                    /* 场景标题右侧 */
  float: right; font-size: 13px; color: #C4A882;
  text-decoration: none; font-weight: 400;
  padding: 0 6px; border-radius: 4px;
  transition: color 0.2s, background 0.2s; line-height: 1.4;
}
.to-top:hover { color: #fff; background: #C4A882; }

.back-to-top {               /* 底部大按钮 */
  display: block; width: fit-content;
  margin: 28px auto 0; padding: 10px 28px;
  background: #C4A882; color: #fff;
  font-size: 15px; font-weight: 700; text-align: center;
  text-decoration: none; border-radius: 24px;
  letter-spacing: 2px; transition: background 0.2s;
}
.back-to-top:hover { background: #A88B6A; }
```

#### 舞台指示

```css
.stage-dir { color: #4a5a4a; font-size: 14px; padding: 3px 0; }
.action-parens { color: #5a6a5a; font-size: 14px; }
```

#### 平滑滚动

```css
html { scroll-behavior: smooth; }
```

### 3.9 多设备适配

```css
/* 小桌面 / 平板横屏 */
@media (max-width: 1024px) {
  .container { padding: 28px 32px; }
}

/* 平板竖屏 */
@media (max-width: 768px) {
  .container { padding: 24px 20px; }
}

/* 手机 */
@media (max-width: 640px) {
  body { padding: 16px 10px; font-size: 16px; }
  .container { padding: 20px 16px; border-radius: 8px; }
  h1 { font-size: 22px; letter-spacing: 2px; }
  .scene-title { font-size: 16px; margin-top: 16px; }
  .stage-dir { font-size: 13px; }
  .legend { gap: 10px; padding: 8px 10px; }
  .legend-item { font-size: 13px; }
  .action-parens { font-size: 13px; }
  .dialogue { padding: 3px 0; }
  .toc { padding: 12px 14px; }
  .toc-list li { font-size: 13px; padding: 4px 0; }
  .toc-chars { font-size: 12px; }
  .to-top { font-size: 12px; }
  .back-to-top { padding: 8px 22px; font-size: 14px; }
}

/* 小屏手机 */
@media (max-width: 400px) {
  body { padding: 10px 6px; font-size: 15px; }
  .container { padding: 14px 10px; border-radius: 6px; }
  h1 { font-size: 19px; letter-spacing: 1px; }
  .legend { flex-direction: column; align-items: center; gap: 6px; }
  .toc { padding: 10px 10px; }
  .toc-title { font-size: 15px; }
  .toc-list li { font-size: 12px; padding: 3px 0; }
  .toc-chars { font-size: 11px; }
  .to-top { font-size: 11px; padding: 0 4px; }
  .back-to-top { padding: 6px 18px; font-size: 13px; }
}
```

---

## 四、输出

生成 HTML 文件名：`{剧本名}-台词高亮版.html`

输出完成后告知用户：
- ✅ 文件已生成
- ✅ 可用浏览器直接打开
- ✅ 目录可点击跳转
- ✅ 场景标题右侧 `↑ 顶部` 可回顶
- ✅ 底部 `⬆ 回到顶部` 按钮
- ✅ 手机端自适应
