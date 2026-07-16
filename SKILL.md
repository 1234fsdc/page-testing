---
name: dev-scanner
description: >
  全自动浏览器项目扫描器。当用户说"审计项目"、"扫描项目"、"检查UI"、"测试页面"、"全面检查"、"跑一遍"、
  "review project"、"audit project"、"检查设计"时触发。打开浏览器逐页扫描项目，做设计审查 + 功能测试 + 
  可访问性检查 + 性能扫描 + 内容审查，输出结构化的 .md 审计报告。适用于 React / Vue / 纯前端等各种项目。
  与 browser-use 配合使用（浏览器操作依赖 browser-use），与 ui-ux-pro-max 互补（它出设计稿，本 skill 审稿）。
---

# dev-scanner — 项目浏览器全自动扫描器

全自动浏览器扫描工具。对项目做 9 个维度的全方位检查，输出结构化审计报告。

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
项目检测 → 路由发现 → 数据模型分析 → 写入测试数据 → 模拟真实用户操作 → 逐页扫描（9个维度） → 报告生成
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
🧪 是否写入测试数据模拟真实用户？(是/否)
```

### 1.4 分析项目数据模型

扫描前先理解项目的数据结构，为后续写入测试数据做准备。

**读取状态管理文件**：
```bash
# Vue (Pinia/Vuex)
cat src/stores/*.js src/stores/*.ts 2>/dev/null
# React (Redux/Zustand/Context)
cat src/store/*.js src/store/*.ts src/context/*.js src/context/*.ts 2>/dev/null
# 查找数据模型/类型定义
find src -name "*.d.ts" -o -name "types.ts" -o -name "models.ts" 2>/dev/null
```

**分析要点**：
- 用户模型：有哪些字段？密码怎么存的？
- 业务模型：商品/帖子/订单等核心数据结构
- 关联关系：用户和商品之间是什么关系？有收藏/点赞吗？
- 验证规则：必填字段？格式要求？长度限制？

**记录到临时变量**，后续写入测试数据时使用。

### 1.5 写入测试数据 + 模拟真实用户操作

在扫描前，通过浏览器模拟真实用户行为，让项目进入"有数据"的状态。这样扫描时能看到真实的页面表现，而不是空态。

#### 测试数据策略

根据项目类型生成**大量、多样化**的测试数据，覆盖各种真实场景。

**原则：宁多勿少。** 数据量要足以暴露分页、滚动、布局、性能等问题。

##### 电商/二手市场类

| 数据类型 | 数量 | 多样性要求 |
|---------|------|-----------|
| 测试用户 | 3 个 | 不同用户名长度（短/中/长）、不同密码强度 |
| 商品 | 8-12 个 | 不同分类、不同价格区间（0/低价/中价/高价/超长数字）、不同标题长度（2字/20字/50字/100字）、有图/无图/多图 |
| 商品描述 | 每商品 1 段 | 空/10字/200字/500字、含换行/含特殊字符/含 emoji |
| 分类 | 覆盖所有分类 | 每个分类至少 2 个商品 |
| 收藏 | 3-5 条 | 不同用户收藏不同商品 |

```
示例商品数据：
1. 标题："iPhone" (2字) / 价格：1 → 测试极短标题
2. 标题："九成新 MacBook Pro 2024 M3 Max 16寸 深空灰 官方在保" (25字) / 价格：12999 → 正常
3. 标题："出一台用了半年的iPad，成色很好，配件齐全，有意私聊，不刀" (28字) / 价格：3500 → 含口语化表达
4. 标题：100字重复文本 → 测试超长标题截断
5. 价格：0 → 测试免费商品
6. 价格：99999999 → 测试超长数字显示
7. 描述：空 → 测试空态
8. 描述：500字 Lorem ipsum + emoji 🎉📦 → 测试长文本 + 特殊字符
9. 无图片商品 → 测试裂图/占位图
10. 分类：每个可用分类各 1 个 → 测试分类筛选
```

##### 社交/论坛类

| 数据类型 | 数量 | 多样性要求 |
|---------|------|-----------|
| 测试用户 | 3-5 个 | 不同昵称长度、有/无头像 |
| 帖子 | 10-15 条 | 不同分类、不同字数、含图/不含图、置顶/非置顶 |
| 评论 | 每帖 2-5 条 | 空评论/长评论/含 @ /含链接 |
| 点赞 | 随机分布 | 不同帖子不同点赞数 |
| 私信 | 2-3 条 | 空对话/长对话 |

```
示例帖子数据：
1. 标题："求助" (2字) / 内容："怎么办" (3字) → 极短
2. 标题：正常标题 / 内容：500字长文 → 正常
3. 标题：含 emoji 🎉🔥💡 / 内容：含换行+列表 → 特殊字符
4. 标题：含 HTML <b>粗体</b> → XSS 测试
5. 标题：含 " OR 1=1 -- → SQL 注入测试
6. 内容：空 → 测试空态
7. 内容：2000字 → 测试超长内容滚动
8. 每个分类至少 2 条 → 测试分类筛选
9. 有图片的帖子 3 条 → 测试图片加载
10. 无图片的帖子 7 条 → 测试无图布局
```

##### SaaS/后台管理类

| 数据类型 | 数量 | 多样性要求 |
|---------|------|-----------|
| 管理员账号 | 2 个 | 超级管理员/普通管理员 |
| 普通用户 | 5-8 个 | 不同状态（活跃/禁用/未验证） |
| 业务数据 | 10-20 条 | 不同状态、不同时间、不同负责人 |
| 仪表盘 | 各指标有数据 | 确保图表有内容可渲染 |

```
示例后台数据：
1. 用户列表：8 个用户（不同注册时间、不同状态）
2. 订单/记录：15 条（待处理/进行中/已完成/已取消 各几条）
3. 通知/消息：5 条（已读/未读）
4. 设置项：各类型都改一下（文本/数字/开关/下拉）
5. 权限测试：普通用户访问管理员页面 → 检查权限控制
```

##### 内容/CMS 类

| 数据类型 | 数量 | 多样性要求 |
|---------|------|-----------|
| 文章 | 8-12 篇 | 不同分类、不同长度、有/无封面图 |
| 分类/标签 | 覆盖所有 | 每个分类至少 2 篇 |
| 评论 | 每篇 2-3 条 | 正常/垃圾/含链接 |
| 草稿 | 2-3 篇 | 测试草稿列表 |

##### 工具类

| 数据类型 | 数量 | 多样性要求 |
|---------|------|-----------|
| 各功能模块 | 每模块 3-5 条记录 | 正常/边界/空 |

#### 写入方式：通过浏览器操作

**不要直接写 localStorage/数据库**——通过浏览器表单操作来写入，这样同时测试了表单功能。

```bash
# 步骤 1：注册测试用户
browser-use <<PY
import time
goto_url("http://localhost:PORT/register")
time.sleep(3)

# 填写注册表单
js("""
(() => {
  // 找到所有 input 并填入值
  const inputs = document.querySelectorAll('input');
  inputs.forEach(i => {
    if (i.type === 'text' && !i.value) {
      i.value = 'testuser';
      i.dispatchEvent(new Event('input', {bubbles: true}));
    }
    if (i.type === 'password' && !i.value) {
      i.value = 'Test1234!';
      i.dispatchEvent(new Event('input', {bubbles: true}));
    }
  });
  // 触发提交
  const btn = document.querySelector('button[type=submit]');
  if (btn) btn.click();
})()
""")
time.sleep(2)
PY

# 步骤 2：登录
browser-use <<PY
import time
goto_url("http://localhost:PORT/login")
time.sleep(3)

js("""
(() => {
  const inputs = document.querySelectorAll('input');
  inputs.forEach(i => {
    if (i.type === 'text') {
      i.value = 'testuser';
      i.dispatchEvent(new Event('input', {bubbles: true}));
    }
    if (i.type === 'password') {
      i.value = 'Test1234!';
      i.dispatchEvent(new Event('input', {bubbles: true}));
    }
  });
  const btn = document.querySelector('button[type=submit]');
  if (btn) btn.click();
})()
""")
time.sleep(2)
PY

# 步骤 3：发布商品/帖子（根据项目类型）
browser-use <<PY
import time
goto_url("http://localhost:PORT/publish")
time.sleep(3)

js("""
(() => {
  // 通用：找到所有 textarea 和 input，填入测试数据
  document.querySelectorAll('textarea').forEach(t => {
    t.value = '这是一条测试数据，用于验证页面在有内容时的表现。';
    t.dispatchEvent(new Event('input', {bubbles: true}));
  });
  document.querySelectorAll('input[type=text]').forEach(i => {
    if (!i.value) {
      i.value = '测试标题';
      i.dispatchEvent(new Event('input', {bubbles: true}));
    }
  });
  document.querySelectorAll('input[type=number]').forEach(i => {
    i.value = '99';
    i.dispatchEvent(new Event('input', {bubbles: true}));
  });
  // 提交
  const btn = document.querySelector('button[type=submit]');
  if (btn) btn.click();
})()
""")
time.sleep(2)
PY
```

#### 测试用户清单

扫描过程中使用以下测试账号（如果项目需要登录）：

| 用户名 | 密码 | 用途 |
|--------|------|------|
| testuser | Test1234! | 普通用户，测试核心功能 |
| admin | Admin1234! | 管理员（如果项目有后台） |

#### 边界条件测试

在写入正常数据后，**必须**测试以下边界情况。每种情况单独测试，记录结果。

##### 表单验证边界

| 测试场景 | 输入 | 预期行为 | 检查方式 |
|---------|------|---------|---------|
| 空提交 | 所有字段留空，点提交 | 显示验证错误 | 检查 `.error` / `[role=alert]` 是否出现 |
| 仅空格 | 输入全空格 `"   "` | 视为空 / trim 后验证 | 检查是否 trim |
| 最小长度 | 输入 1 个字符 | 如果有最小长度限制应报错 | 对比规则 |
| 最大长度 | 输入 500+ 字符 | 截断 / 报错 / 正常 | 检查是否溢出 |
| 重复字段值 | 两次输入不同值 | 如"确认密码"应报不一致 | 检查错误提示 |
| 数字边界 | 价格输入 0 / -1 / 99999999 / 小数 | 根据业务规则验证 | 检查结果 |

##### 安全边界

| 测试场景 | 输入 | 预期行为 | 检查方式 |
|---------|------|---------|---------|
| XSS | `<script>alert(1)</script>` | 转义或拒绝 | 检查是否弹窗 / DOM 中是否保留标签 |
| HTML 注入 | `<img src=x onerror=alert(1)>` | 转义或拒绝 | 检查 DOM |
| SQL 注入 | `' OR 1=1 --` | 不影响查询 | 检查是否报错 |
| 超长 URL | `/products/` + 2000 字符 | 404 / 截断 / 正常 | 检查响应 |
| 特殊路径 | `/products/../../../etc/passwd` | 404 / 拒绝 | 检查响应 |

##### 交互边界

| 测试场景 | 操作 | 预期行为 | 检查方式 |
|---------|------|---------|---------|
| 快速重复点击 | 连续点击提交按钮 5 次 | 不重复创建 / 防抖 | 检查数据条数 |
| 浏览器前进/后退 | 操作后点后退按钮 | 页面状态正确 | 检查页面内容 |
| 刷新页面 | 操作后刷新 | 数据保持 / 状态正确 | 检查数据 |
| 网络断开模拟 | 提交时断网 | 有错误提示 / 不静默失败 | 检查错误处理 |
| 未登录操作 | 退出后访问受保护页 | 跳转登录页 | 检查 URL |

##### 内容边界

| 测试场景 | 输入 | 预期行为 | 检查方式 |
|---------|------|---------|---------|
| 纯 emoji | 🎉🔥💡📦 | 正常显示 | 检查渲染 |
| 混合语言 | "Hello 你好 ハロorld" | 正常显示 | 检查渲染 |
| RTL 文字 | "مرحبا" | 正常显示 / 不破坏布局 | 检查布局 |
| 换行/制表符 | 含 `\n` `\t` 的文本 | 正确渲染 | 检查 DOM |
| 零宽字符 | `\u200B\u200C\u200D` | 不影响显示 | 检查布局 |
| HTML 实体 | `&amp; &lt; &gt;` | 正确转义显示 | 检查渲染 |

```bash
# 边界条件测试完整示例
browser-use <<PY
import time, json

goto_url("http://localhost:PORT/publish")
time.sleep(3)

results = []

# 1. 空提交
js("document.querySelector('button[type=submit]')?.click()")
time.sleep(1)
error = js("document.querySelector('.error, .form-message, [role=alert]')?.textContent || '无'")
results.append({"test": "空提交", "error_hint": error})

# 2. 超长文本
js("""
(() => {
  const ta = document.querySelector('textarea');
  if (ta) { ta.value = 'A'.repeat(500); ta.dispatchEvent(new Event('input', {bubbles:true})); }
  const ti = document.querySelector('input[type=text]');
  if (ti) { ti.value = 'B'.repeat(100); ti.dispatchEvent(new Event('input', {bubbles:true})); }
})()
""")
time.sleep(1)
overflow = js("document.documentElement.scrollWidth > document.documentElement.clientWidth")
results.append({"test": "超长文本", "overflow": overflow})

# 3. XSS
js("""
(() => {
  const ta = document.querySelector('textarea');
  if (ta) { ta.value = '<script>alert(1)</script>'; ta.dispatchEvent(new Event('input', {bubbles:true})); }
})()
""")
time.sleep(1)
has_script = js("document.body.innerHTML.includes('<script>alert(1)</script>')")
results.append({"test": "XSS", "raw_html": has_script})

# 4. 特殊字符
js("""
(() => {
  const ta = document.querySelector('textarea');
  if (ta) { ta.value = '🎉🔥 "quotes" \\backslash\\ <html>'; ta.dispatchEvent(new Event('input', {bubbles:true})); }
})()
""")
time.sleep(1)
rendered = js("document.querySelector('textarea')?.value || '无'")
results.append({"test": "特殊字符", "rendered": rendered})

print(json.dumps(results, ensure_ascii=False, indent=2))
PY
```

#### 清理测试数据（可选）

扫描完成后，询问用户是否清理测试数据：

```
🧹 扫描完成。是否清理刚才写入的测试数据？
   - 清理：删除测试用户和测试商品
   - 保留：保持现状
```

---

## 阶段二：逐页全面扫描（9个维度）

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

### 维度 G：真实用户操作验证

**在写入测试数据后，验证数据是否正确渲染。** 这是前面"写入测试数据"的闭环检查。

```bash
# 验证测试数据是否出现在页面上
browser-use <<PY
import time
goto_url("http://localhost:PORT/products")
time.sleep(3)

# 检查是否有商品卡片
cards = js("document.querySelectorAll('[class*=card], [class*=product], [class*=item]').length")
print(f"商品卡片数: {cards}")

# 检查是否有测试数据的标题
has_test = js("document.body.innerText.includes('测试标题')")
print(f"包含测试数据: {has_test}")

# 检查空态是否隐藏（有数据时不应显示空态）
empty = js("document.querySelector('[class*=empty], [class*=no-data]')?.offsetHeight === 0")
print(f"空态隐藏: {empty}")
PY
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

## 🧪 边界条件测试结果

| 测试场景 | 结果 | 详情 |
|---------|------|------|
| 空提交验证 | ✅/❌ | {具体表现} |
| 超长文本 | ✅/❌ | {是否溢出} |
| XSS 防护 | ✅/❌ | {是否转义} |
| 特殊字符 | ✅/❌ | {是否正常渲染} |

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
7. **不破坏环境**：扫描过程中不要修改项目代码，只读不写（测试数据通过浏览器写入，可选清理）
