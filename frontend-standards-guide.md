# 前端企业级规范入门指南

> 用你的 shrimp-8.html 作为案例，对比讲解"不规范"和"企业级"的区别

---

## 目录

1. [为什么要有规范？](#1-为什么要有规范)
2. [HTML 语义化](#2-html-语义化)
3. [CSS 设计系统](#3-css-设计系统)
4. [JavaScript 健壮性](#4-javascript-健壮性)
5. [无障碍访问 (A11y)](#5-无障碍访问-a11y)
6. [性能优化](#6-性能优化)
7. [常见错误自查表](#7-常见错误自查表)

---

## 1. 为什么要有规范？

你原来的代码"能工作"，但企业级项目关注的是**长期维护**：

| 维度 | 你原来的代码 | 企业级要求 |
|------|-------------|-----------|
| 可读性 | 自己看得懂 | 团队任何人都能快速理解 |
| 可维护性 | 改一处可能影响别处 | 改动局部不影响整体 |
| 可靠性 | 运行正常，但隐藏bug | 边界情况都处理了 |
| 可访问性 | 正常人能用 | 盲人、键盘用户也能用 |
| 性能 | 加载慢不关心 | 优化到极致 |
| 协作 | 一个人写 | 多人并行开发 |

**核心思想：写代码时要假设 6 个月后的你会完全忘记现在的逻辑。**

---

## 2. HTML 语义化

### 2.1 用对标签，别只用 div

你原来的写法，大量 `<div>`：

```html
<!-- ❌ 不规范：全是 div，不知道每个部分是什么 -->
<div class="header">...</div>
<div class="content-area">...</div>
<div class="footer">...</div>
```

改后的写法，标签自带含义：

```html
<!-- ✅ 企业级：标签告诉浏览器和开发者这是什么 -->
<header>...</header>
<main>
  <section aria-label="系统概览">...</section>
  <section aria-label="AI算法与技术">...</section>
</main>
<footer>...</footer>
```

**记住这张对照表：**

| 你的习惯 | 应该用 |
|---------|--------|
| `<div class="header">` | `<header>` |
| `<div class="nav">` | `<nav>` |
| `<div class="main">` | `<main>` |
| `<div class="footer">` | `<footer>` |
| `<div class="section">` | `<section>` |
| `<div class="article">` | `<article>` |
| `<div class="figure">` | `<figure>` |
| `<div class="sidebar">` | `<aside>` |

### 2.2 为什么这很重要？

1. **SEO** — 搜索引擎知道哪里是主要内容，提升排名
2. **无障碍** — 屏幕阅读器能直接跳转到 `<main>`、`<nav>`
3. **可读性** — 看标签就知道结构，不用读 class 名
4. **未来兼容** — 浏览器原生支持，不会因为框架变迁而失效

### 2.3 meta 标签

你原来的：

```html
<!-- ❌ 缺了最重要的 description -->
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

企业级：

```html
<!-- ✅ 描述会显示在搜索引擎结果页 -->
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta name="description" content="基于人工智能的虾类识别与分类系统，支持10种常见虾类自动识别">
```

---

## 3. CSS 设计系统

### 3.1 CSS 变量（Custom Properties）

你原来的：

```css
/* ❌ 到处硬编码，改个主色调要搜索替换几十处 */
background: #3498db;
color: #2c3e50;
border-radius: 10px;
```

企业级：

```css
/* ✅ 定义一次，全局使用 */
:root {
  --color-primary: #3498db;
  --color-primary-dark: #2980b9;
  --color-text-primary: #2c3e50;
  --radius-md: 10px;
  --shadow-sm: 0 3px 10px rgba(0, 0, 0, 0.1);
  --transition-base: 0.3s ease;
}

/* 使用 */
.button {
  background: var(--color-primary);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-sm);
  transition: all var(--transition-base);
}

.button:hover {
  background: var(--color-primary-dark);
}
```

**好处：** 想换主题色？改 `:root` 里的一个值，全局生效。

### 3.2 BEM 命名法

你原来的：

```css
/* ❌ 命名随意，层级关系靠猜 */
.section-title { }
.accuracy-text { }
.species-habitat { }
.chart-container { }
```

企业级 BEM：

```css
/* ✅ Block__Element--Modifier，一看就知道关系 */
.chart-carousel { }                    /* 块（Block） */
.chart-carousel__title { }             /* 元素（Element）：属于 block 的一部分 */
.chart-carousel__slide { }             /* 元素 */
.chart-carousel__slide--active { }     /* 修饰符（Modifier）：特殊状态 */
```

BEM 规则就三条：

| 符号 | 含义 | 例子 |
|------|------|------|
| `.block` | 组件本身 | `.species-card` |
| `.block__element` | 组件的一部分 | `.species-card__name` |
| `.block--modifier` | 组件的变体 | `.species-card--selected` |

**为什么不用你原来的方式？**

你写 `.upload-area i` 这样的选择器，依赖于 HTML 结构。如果你把 `<i>` 改成 `<span>`，样式就失效了。BEM 不依赖标签结构，class 本身就说清楚了一切。

### 3.3 把"样式"和"结构"分开

你原来的 HTML 里混着行内样式：

```html
<!-- ❌ style 散落在 HTML 里，难以维护 -->
<div class="accuracy-fill" style="width: 88%"></div>
<div class="species-habitat" style="background: #ecf0f1; border-radius: 10px;"></div>
```

企业级——所有样式在 CSS 里，HTML 只负责结构：

```html
<!-- ✅ HTML 只保留结构和 class -->
<div class="accuracy-bar__fill" style="width: 88%"></div>
```

> 宽度数据必须行内写，因为它是动态的。但颜色、圆角等静态样式都应该在 CSS 里。

### 3.4 响应式设计

你原来只做了一个断点，而且零散分布。企业级做法是按设备归类：

```css
/* ✅ 断点集中管理，逻辑清晰 */

/* 平板及以下 */
@media (max-width: 768px) {
  .stats-grid { grid-template-columns: 1fr; }
  .detail-page__grid { grid-template-columns: 1fr; }
  .map-layout { flex-direction: column; }
}

/* 手机 */
@media (max-width: 480px) {
  .algorithm-grid { grid-template-columns: 1fr; }
  .chart-carousel__viewport { height: 400px; }
}
```

---

## 4. JavaScript 健壮性

### 4.1 不要信任任何 DOM 元素

你的代码：

```js
// ❌ 假如这个 id 改名字或者被删除了，整个脚本报错崩溃
const mainContent = document.getElementById('mainContent');
mainContent.style.display = 'none';
```

企业级：

```js
// ✅ 获取不到就跳过，不影响其他功能
var el = document.getElementById('mainContent');
if (el) el.style.display = 'none';

// 或者封装成工具函数
function getElement(id) {
  var el = document.getElementById(id);
  if (!el) console.warn('Element not found: #' + id);
  return el;
}
```

**原则：页面上任何元素都可能不存在（被删除、被改id、被条件渲染）。总是检查。**

### 4.2 不要用 alert()

你的代码：

```js
// ❌ alert 无法自定义样式，阻塞用户操作，体验极差
alert('图片大小不能超过5MB');
```

企业级——用 Toast 通知：

```js
// ✅ 不阻塞用户，自动消失，可自定义样式
function showToast(message, type) {
  var toast = document.createElement('div');
  toast.className = 'toast toast--' + type;
  toast.textContent = message;
  document.body.appendChild(toast);
  setTimeout(function () { toast.remove(); }, 4000);
}

// 使用
showToast('图片大小不能超过5MB', 'error');
```

### 4.3 避免重复代码（DRY 原则）

你的代码里文件验证逻辑写了两遍（change 事件和 drop 事件）：

```js
// ❌ 同样的逻辑复制粘贴，改了上面忘了下面
// change 事件里:
if (!file.type.match('image.*')) { alert('...'); return; }
if (file.size > 5 * 1024 * 1024) { alert('...'); return; }
const reader = new FileReader();
...

// drop 事件里:
if (!file.type.match('image.*')) { alert('...'); return; }
if (file.size > 5 * 1024 * 1024) { alert('...'); return; }
const reader = new FileReader();
...
```

企业级——提取成函数：

```js
// ✅ 写一次，到处用
function isValidImageFile(file) {
  if (!file.type.match('image.*')) {
    showToast('请选择图片文件', 'error');
    return false;
  }
  if (file.size > 5 * 1024 * 1024) {
    showToast('图片大小不能超过5MB', 'error');
    return false;
  }
  return true;
}

function handleFileSelect(file) {
  if (!isValidImageFile(file)) return;
  // 只处理业务逻辑
  currentImageFile = file;
  var reader = new FileReader();
  reader.onload = function(e) {
    previewImage.src = e.target.result;
    performRecognition(file);
  };
  reader.readAsDataURL(file);
}
```

### 4.4 一个事件入口

你原来有多个 `DOMContentLoaded`：

```js
// ❌ 多个入口，执行顺序不明确
document.addEventListener('DOMContentLoaded', function() {
  // 图表轮播
});
document.addEventListener('DOMContentLoaded', function() {
  // 地图初始化
});
```

企业级：

```js
// ✅ 一个入口，内部按功能模块拆分成函数
document.addEventListener('DOMContentLoaded', function () {
  initSpeciesCards();
  initBackButton();
  initUploadArea();
  initModelSelector();
  initChartCarousel();
  initDistributionMap();
});
```

### 4.5 使用 'use strict'

```js
// ✅ 捕获常见错误（如未声明变量直接赋值）
'use strict';

// 这会报错，帮你发现bug：
x = 10;  // ReferenceError: x is not defined
```

---

## 5. 无障碍访问 (A11y)

### 5.1 屏幕阅读器支持

你的代码：

```html
<!-- ❌ 屏幕阅读器不知道这是什么 -->
<div class="species-card" data-species="河虾">
  <i class="fas fa-shrimp"></i>
  <div class="species-name">河虾</div>
</div>
```

企业级：

```html
<!-- ✅ 告诉屏幕阅读器：这是个按钮，点击会查看详情 -->
<article class="species-card" data-species="河虾"
         tabindex="0"
         role="button"
         aria-label="查看河虾详情">
  <i class="fas fa-shrimp species-card__icon" aria-hidden="true"></i>
  <p class="species-card__name">河虾</p>
</article>
```

**关键属性解释：**

| 属性 | 作用 |
|------|------|
| `role="button"` | 告诉屏幕阅读器这是个按钮 |
| `tabindex="0"` | 让键盘 Tab 键能聚焦到它 |
| `aria-label="..."` | 提供语音描述 |
| `aria-hidden="true"` | 让图标对屏幕阅读器静音（纯装饰） |
| `aria-live="polite"` | 内容变化时通知屏幕阅读器 |

### 5.2 键盘导航

你的代码里，点击物种卡片是鼠标事件：

```js
speciesCard.addEventListener('click', function() { ... });
```

企业级还要支持键盘：

```js
// ✅ 同时处理鼠标和键盘
speciesCard.addEventListener('click', function() { ... });
speciesCard.addEventListener('keydown', function(e) {
  if (e.key === 'Enter' || e.key === ' ') {
    e.preventDefault();
    // 执行同样的逻辑
  }
});
```

### 5.3 焦点管理

打开详情页时，把焦点移到详情页：

```js
// ✅ 让键盘用户能立即操作详情页
detailPage.focus();
```

---

## 6. 性能优化

### 6.1 非阻塞加载 CSS

你的写法：

```html
<!-- ❌ 阻塞渲染，用户要等所有 CSS 加载完才能看到页面 -->
<link rel="stylesheet" href="https://...font-awesome.css">
<link rel="stylesheet" href="https://...leaflet.css">
```

企业级：

```html
<!-- ✅ 先加载核心样式，非关键样式异步加载 -->
<link rel="stylesheet" href="https://...font-awesome.css"
      media="print" onload="this.media='all'">
```

`media="print"` 让浏览器以为这个样式只在打印时用，不阻塞渲染。`onload="this.media='all'"` 加载完成后把它切换为正常样式。

### 6.2 使用 defer 加载 JS

```html
<!-- ✅ 不阻塞 HTML 解析，等 DOM 就绪后再执行 -->
<script src="https://unpkg.com/leaflet.js" defer></script>
```

对比：

| 方式 | 行为 |
|------|------|
| 普通 `<script>` | 阻塞 HTML 解析，下载完立刻执行 |
| `<script defer>` | 不阻塞，HTML 解析完再按顺序执行 |
| `<script async>` | 不阻塞，下载完立刻执行（顺序不保证） |

> 对于操作 DOM 的脚本，99% 的情况用 `defer`。

### 6.3 预连接 CDN

```html
<!-- ✅ 提前建立连接，减少 DNS 查询时间 -->
<link rel="preconnect" href="https://cdnjs.cloudflare.com">
<link rel="preconnect" href="https://fonts.googleapis.com">
```

### 6.4 图片懒加载与备用

你原来每个 chart-slide 里嵌了一大段相同的 base64 SVG 作为 onerror 备选：

```html
<!-- ❌ 每个图片都嵌了相同的 600 行 base64，冗余 -->
<img src="..." onerror="this.src='data:image/svg+xml;base64,PHN...'">
```

企业级——用 CSS 或 JS 统一处理：

```js
// ✅ 统一的降级策略，不改 HTML
function setFallback(img) {
  img.onerror = function() {
    this.style.display = 'none';
    // 或者显示一个统一的占位
  };
}
```

---

## 7. 常见错误自查表

写代码时对照这张表检查：

| 检查项 | ✅ 规范做法 | ❌ 不规范做法 |
|--------|-----------|-------------|
| HTML 结构 | 用 `<header>` `<main>` `<section>` | 全是 `<div>` |
| CSS 颜色 | CSS 变量 `var(--color-primary)` | 到处写 `#3498db` |
| CSS 命名 | BEM `block__element--modifier` | 随意取名或者 `-` `_` 混用 |
| CSS 选择器 | `.card__title` | `.card div span`（依赖标签层级） |
| JS 变量 | 总是用 `var`/`let`/`const` | 直接 `x = 10`（全局污染） |
| JS DOM 操作 | 先检查元素是否存在 | 直接 `el.style = ...`（可能崩溃） |
| JS 用户反馈 | Toast / Modal | `alert()` / `confirm()` |
| JS 重复代码 | 提取成函数复用 | 复制粘贴 |
| 可访问性 | 加 `role` `aria-label` `tabindex` | 完全不考虑 |
| 键盘支持 | 同时处理 `click` 和 `keydown` | 只支持鼠标点击 |
| 性能 | CSS 异步加载、JS defer | 全部同步加载 |

---

## 总结

企业级前端规范的核心就一句话：

> **不要只顾着"让它跑起来"，要考虑"让下一个人能看懂、能修改、不容易弄坏"。**

你原来的代码并没有"错"——每个初学者都那样写。但每当你学会一条规范，你就从一个"能写代码的人"向"专业的软件工程师"迈进了一步。

下一步建议：

1. 打开你改好的 shrimp-8.html，对照这个文档逐条看区别
2. 自己手写一个简单页面（比如个人介绍页），刻意练习 BEM 和语义 HTML
3. 学习 Chrome DevTools 的 Accessibility 面板，检查自己的页面

有任何不理解的地方，随时问我。
