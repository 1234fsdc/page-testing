---
name: dev-scanner
description: >
  全自动浏览器项目扫描器。当用户说"审计项目"、"扫描项目"、"检查UI"、"测试页面"、"全面检查"、"跑一遍"、
  "review project"、"audit project"、"检查设计"时触发。打开浏览器逐页扫描项目，做设计审查 + 功能测试 + 
  可访问性检查 + 性能扫描 + 内容审查，输出结构化的 .md 审计报告。适用于 React / Vue / 纯前端等各种项目。
  与 browser-use 配合使用（浏览器操作依赖 browser-use），与 ui-ux-pro-max 互补（它出设计稿，本 skill 审稿）。
---

# dev-scanner — 项目浏览器全自动扫描器

全自动浏览器扫描工具。对项目做 8 个维度的全方位检查，输出结构化审计报告。

使用 `browser-use` skill 进行浏览器操作，通过 CDP 控制 Chrome。

---

## 触发方式

用户说出以下任意关键词即自动触发：
- `审计项目` / `扫描项目` / `全面检查` / `跑一遍检查`
- `检查UI` / `检查设计` / `检查页面`
- `测试页面` / `测试功能`
- `review project` / `audit project`

---

## 工作流程概览

```
项目检测 → 路由发现 → 逐页扫描（8个维度） → 报告生成
```

---

## 阶段一：项目检测与准备

### 1.1 检测项目类型
```bash
# 检查项目类型
cat package.json 2>/dev/null | grep -E '"next|"nuxt|"vue|"react|"vite|"angular"'
ls next.config.js nuxt.config.js vite.config.js vue.config.js 2>/dev/null
```

识别后确定启动方式：
- Next.js: `npm run dev` (默认 3000)
- Vue/Vite: `npm run dev` (默认 5173)
- Nuxt: `npm run dev` (默认 3000)
- 纯静态: 直接 `npx serve .` (默认 3000)
- 如果已有 dev server 在运行，直接使用

### 1.2 发现页面路由

按优先级尝试以下方法：

**方法 A：扫描路由配置文件**
```bash
# Next.js App Router
find src/app -name "page.tsx" -o -name "page.jsx" -o -name "page.js" 2>/dev/null
# Next.js Pages Router
find src/pages -name "*.tsx" -o -name "*.jsx" 2>/dev/null
# Vue Router
grep -r "path:" src/router/ 2>/dev/null
# Nuxt
find pages -name "*.vue" 2>/dev/null
```

**方法 B：首页爬取**
用 browser-use 打开首页，执行 JS 提取所有内部链接。

**方法 C：询问用户**
如果自动发现不完整，列出已发现的路由，让用户补充/排除。

### 1.3 前置确认

开始扫描前向用户确认：
```
📋 发现 {N} 个页面路由
❌ 排除路由：/admin/settings, /api/...
🔑 是否需要登录？(是/否)
```

---

## 阶段二：逐页全面扫描（8个维度）

对每个页面使用 browser-use 执行以下操作。

### 2.0 通用浏览器操作（关键：每页单独调用）

```bash
# ✅ 正确：shell 循环，每页单独一次 browser-use
for page in "/" "/login" "/products"; do
  browser-use <<PY
goto_url("http://localhost:PORT${page}")
import time; time.sleep(3)
r = js("JSON.stringify({title: document.title, inputs: document.querySelectorAll('input').length})")
print(r)
PY
  sleep 1
done

# ❌ 错误：单次 heredoc 中循环导航，会超时
browser-use <<'PY'
for p in ["/", "/login"]:
    goto_url(f"http://localhost:3000{p}")  # 超时！
PY
```

### 维度 A：UI/视觉设计审查

对每个页面执行：

1. **截图**（3 种）
   - 桌面端截图 (1440×900) `capture_screenshot()`
   - 移动端截图 (375×812) — 设置 viewport 后截
   - 平板截图 (768×1024)

2. **布局检查** — 用 JS 检查：
   ```javascript
   // 检查水平溢出
   document.documentElement.scrollWidth > document.documentElement.clientWidth
   // 检查重叠元素
   // 检查意外滚动条
   ```

3. **组件状态检查** — 对每个可交互元素：
   - 悬停状态：`js("el.dispatchEvent(new MouseEvent('mouseover'))")` → 截图
   - 聚焦状态：`js("el.focus()")` → 截图
   - 禁用状态检测：`el.disabled`

4. **对比度检查**
   - 提取文本颜色和背景色
   - 计算对比度（参考 WCAG AA: ≥ 4.5:1）

5. **图片检查**
   - 检测裂图：`document.querySelectorAll('img').forEach(i => { if(!i.complete || i.naturalWidth===0) ...})`
   - 检测缺少 alt：`document.querySelectorAll('img:not([alt])')`
   - 检测宽高比异常

### 维度 B：功能交互测试

1. **链接有效性**
   ```javascript
   // 提取所有内部链接
   Array.from(document.querySelectorAll('a[href^="/"]')).map(a => a.href)
   ```
   逐个访问，检查是否 404/报错。

2. **表单测试**
   - 找到表单 → 尝试空提交 → 检查验证提示
   - 填入错误数据 → 检查错误提示
   - 填入合法数据 → 检查提交成功

3. **核心流程**
   - 如果页面有登录/注册/搜索等核心功能，模拟操作
   - 每次操作后截图记录

4. **交互组件**
   - 找到弹窗按钮 → 点击 → 截图 → 关闭
   - 下拉菜单 → 展开 → 截图 → 收起
   - 折叠面板/标签页 → 切换 → 截图

### 维度 C：控制台 & 网络监控

在每个页面打开时启用 CDP 的 Console 和 Network 监听：

```python
# 通过 js() 捕获 console 信息
# 注意：browser-use 的 CDP 支持 Network 域
```

记录：
- ❌ JS 异常（error 级别）
- ⚠️ 警告（warn 级别）
- ❌ 失败的 API 请求（4xx/5xx）
- ⚠️ 弃用警告
- ❌ 混合内容错误

### 维度 D：可访问性基础

```javascript
// 键盘导航检查
document.querySelectorAll('a, button, input, select, textarea, [tabindex]')

// 检查焦点指示（聚焦时 outline 是否为 none）
document.activeElement?.style?.outline

// 检查触控目标
document.querySelectorAll('a, button').forEach(el => {
  const rect = el.getBoundingClientRect()
  if (rect.width < 44 || rect.height < 44) {/* 记录 */}
})

// 检查 ARIA
document.querySelectorAll('[role]:not([aria-label]):not([aria-labelledby])')
```

### 维度 E：性能基础

```javascript
// 页面加载时间（Performance API）
performance.timing.loadEventEnd - performance.timing.navigationStart

// 大图检测
document.querySelectorAll('img').forEach(i => { /* 记录自然宽高 > 视口很多的 */ })
```

### 维度 F：内容合理性

```javascript
// 检测 lorem ipsum 残留
document.body.innerText.match(/lorem\s+ipsum/i)

// Meta 标签检查
document.querySelector('title')?.textContent
document.querySelector('meta[name="description"]')?.content
document.querySelector('link[rel="icon"]')

// 自定义 404 检测（如果页面是 404）
```

---

## 阶段三：报告生成

所有检查完成后，生成结构化 markdown 报告。

### 报告存储
```
项目根目录/audit-reports/audit-report-YYYY-MM-DD-HHmm.md
项目根目录/audit-reports/screenshots/  (截图存放)
```

### 报告风格要求

**只描述问题，不给修复建议。** 每个问题必须包含三部分：

1. **现状**：浏览器实际看到了什么、代码里写的是什么（引用具体文件和行号）
2. **影响**：对用户/SEO/安全/可访问性具体有什么后果
3. **代码位置**：问题在哪个文件哪一行

不要写"修复方案"、"建议"、代码示例。问题描述要足够详细，让读者看完就知道该怎么改。

### 报告模板

```markdown
# 📊 {项目名} - 浏览器审计报告

**日期**: {日期}
**扫描页面**: {已扫}/{发现} 个
**总体评分**: {分数}/100

---

## 📈 概览统计

| 指标 | 数值 |
|------|------|
| 发现页面 | {N} |
| 扫描页面 | {N} |
| 总问题数 | {N} |
| 🔴 严重 | {N} |
| 🟠 主要 | {N} |
| 🟡 次要 | {N} |
| 🔵 建议 | {N} |

---

## 🏆 Top 急需修复

1. **[严重]** {一句话问题描述}
2. **[严重]** {一句话问题描述}
3. **[主要]** {一句话问题描述}

---

## 📄 逐页详情

---

### 🏠 / (首页)

**浏览器测试数据**: 16 链接 / 0 表单 / 0 按钮
**页面标题**: {实际标题}

#### 🔴 {问题标题}

**现状**：{浏览器实际看到了什么、HTML 结构是什么样的}

**影响**：{对用户/SEO/安全/可访问性具体后果，列 2-4 个要点}

**代码位置**：`{文件路径}:{行号}` — {具体代码片段或描述}

#### 🟠 {下一个问题标题}

**现状**：...

**影响**：...

**代码位置**：...

---

### 🔐 /login (登录页)

**浏览器测试数据**: 1 表单 / 2 输入框 / 7 链接

#### 🟠 {问题标题}

**现状**：...

**影响**：...

**代码位置**：...

---
(其他页面同理)

---

## 🔐 安全专项发现

| 级别 | 问题 | 位置 | 详细说明 |
|------|------|------|----------|
| 🔴 严重 | {问题} | `{文件}:{行号}` | {一句话详细说明} |

---

## 📋 修复优先级

### 🔴 必须立即修复
1. ...

### 🟠 尽快修复
4. ...

### 🟡 可选优化
7. ...

---

## 🛠️ 技术备注

(扫描过程中发现的工具限制等技术信息)
```

### 严重度分级标准

| 级别 | 定义 | 示例 |
|------|------|------|
| 🔴 **严重** | 用户无法使用或严重错误 | 页面白屏、表单不可提交、JS 报错、布局严重断裂 |
| 🟠 **主要** | 影响体验但可绕过 | 对比度不足、响应式局部断裂、缺空态、链接 404 |
| 🟡 **次要** | 不符合最佳实践 | 缺 alt 文本、触控目标偏小、缺 meta 标签 |
| 🔵 **建议** | 可优化项 | 文案微调、间距微调、频率不高的可访问性问题 |

---

## ⚠️ browser-use 稳定性问题与解决方案（实测验证）

### 问题 1：capture_screenshot() 大视口超时

**根因**：视口越大，截图 PNG 越大，CDP 传输超时。2560px 视口截图约 10-15MB。

**解决方案**：
```python
browser-use <<'PY'
# 方案 A：截图时自动缩图（推荐，最简单）
capture_screenshot(max_dim=1440)

# 方案 B：先设小视口再截
import time
cdp("Emulation.setDeviceMetricsOverride", width=1440, height=900, deviceScaleFactor=1)
capture_screenshot()
cdp("Emulation.clearDeviceMetricsOverride")  # 恢复
PY
```

### 问题 2：wait_for_load() SPA 渲染超时

**根因**：`wait_for_load()` 检查的是 `document.readyState == "complete"`，但 Vue/React SPA 在 DOM ready 后还要做组件渲染、数据请求。SPA 报告 `complete` 时，内容可能还没出来。

**解决方案**：自定义等待函数，检查 Vue/React 根元素是否挂载
```python
browser-use <<'PY'
import time

def wait_for_spa(timeout=10):
    """等 Vue/React SPA 渲染完成"""
    deadline = time.time() + timeout
    while time.time() < deadline:
        ready = js("""
            (() => {
                // Vue: #app 下有子节点说明渲染了
                if (document.querySelector('#app')?.children.length > 0) return true;
                // React: #root 下有子节点
                if (document.querySelector('#root')?.children.length > 0) return true;
                // 通用：body 下有非 script 的子节点
                const els = document.body.querySelectorAll('h1, h2, section, main, .page');
                return els.length > 0;
            })()
        """)
        if ready:
            return True
        time.sleep(0.5)
    return False

goto_url("http://localhost:5173/products")
wait_for_spa()
print(js("document.querySelectorAll('.product-card').length"))
PY
```

### 问题 3：单次 heredoc 循环导航超时

**根因**：browser-use 的 IPC 连接在 `goto_url` 后，`cdp("Page.navigate")` 返回很快，但浏览器可能还在加载新页面。下一次 `js()` 调用时 CDP 连接已断或 session 过期。

**解决方案**：shell 循环 + 每页独立 browser-use 调用
```bash
source <venv>/Scripts/activate
PAGES=("/" "/login" "/register" "/products")
PORT=5173

for page in "${PAGES[@]}"; do
  echo "=== 扫描: $page ==="
  browser-use <<PY
import time
goto_url("http://localhost:${PORT}${page}")
time.sleep(3)
r = js("JSON.stringify({title: document.title, url: location.href, inputs: document.querySelectorAll('input').length, forms: document.querySelectorAll('form').length, links: document.querySelectorAll('a[href]').length})")
print(r)
PY
  sleep 1  # 页间间隔
done
```

### 问题 4：每次调用只能做 1-2 个检查

**根因**：IPC 连接不稳定，多次 `js()` 调用累积延迟，容易在第 3-4 次调用时超时。

**解决方案**：合并多个检查为一次 JS 调用
```python
browser-use <<'PY'
import time, json
goto_url("http://localhost:5173/products")
time.sleep(3)

# 一次 js() 完成所有检查
r = js("""
(() => {
  const c = {};
  c.title = document.title;
  c.inputs = document.querySelectorAll('input').length;
  c.inputsWithoutName = Array.from(document.querySelectorAll('input')).filter(i=>!i.name).length;
  c.forms = document.querySelectorAll('form').length;
  c.links = document.querySelectorAll('a[href]').length;
  c.overflow = document.documentElement.scrollWidth > document.documentElement.clientWidth;
  c.brokenImages = Array.from(document.querySelectorAll('img')).filter(i=>!i.complete||i.naturalWidth===0).length;
  c.imagesNoAlt = document.querySelectorAll('img:not([alt])').length;
  c.hasDescription = !!document.querySelector('meta[name="description"]');
  return c;
})()
""")
print(json.dumps({"page": "/products", **json.loads(r)}, ensure_ascii=False))
PY
```

### 最佳实践：完整扫描脚本模板

```bash
source <venv>/Scripts/activate
PORT=5173
PAGES=("/" "/login" "/register" "/products" "/publish" "/profile")

for page in "${PAGES[@]}"; do
  browser-use <<PY
import time, json
goto_url("http://localhost:${PORT}${page}")
time.sleep(3)

r = js("""
(() => {
  const c = {};
  c.title = document.title;
  c.inputs = document.querySelectorAll('input').length;
  c.inputsWithoutName = Array.from(document.querySelectorAll('input')).filter(i=>!i.name).length;
  c.forms = document.querySelectorAll('form').length;
  c.links = document.querySelectorAll('a[href]').length;
  c.overflow = document.documentElement.scrollWidth > document.documentElement.clientWidth;
  c.brokenImages = Array.from(document.querySelectorAll('img')).filter(i=>!i.complete||i.naturalWidth===0).length;
  c.imagesNoAlt = document.querySelectorAll('img:not([alt])').length;
  c.smallTouchTargets = Array.from(document.querySelectorAll('a, button')).filter(el => {
    const r = el.getBoundingClientRect();
    return (r.width > 0 && r.height > 0) && (r.width < 44 || r.height < 44);
  }).length;
  c.hasDescription = !!document.querySelector('meta[name="description"]');
  c.hasFavicon = !!document.querySelector('link[rel="icon"]');
  return c;
})()
""")
print(json.dumps({"page": "${page}", **json.loads(r)}, ensure_ascii=False))
PY
  sleep 1
done
```

## 其他注意事项

1. **登录态**：如果项目需要登录，询问用户提供凭据（用户名/密码或 cookie）
2. **耗时**：20+ 页面全面扫描可能需要 5-15 分钟，提前告知用户
3. **排除路由**：用户可指定不需要扫描的路由（如 /api/*、/admin/*）
4. **扫描中断**：如果扫描中途出错，已完成的页面仍然生成报告，在报告顶部注明"部分完成"
5. **截图保存**：截图保存到 `audit-reports/screenshots/`，在报告中引用路径
6. **增量扫描**：如果用户说"再扫一次"，对比前一次报告，只标注新增/修复的问题
7. **不破坏环境**：扫描过程中不要修改项目代码，只读不写
