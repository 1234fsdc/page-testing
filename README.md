<h1 align="center">dev-scanner</h1>
<p align="center"><em>全自动浏览器项目扫描器 — 一句话触发，逐页扫描，结构化报告</em></p>

<p align="center">
  <a href="https://github.com/1234fsdc/dev-scanner/blob/main/LICENSE"><img src="https://img.shields.io/badge/License-MIT-green.svg" alt="License: MIT"/></a>
  <img src="https://img.shields.io/badge/skill-dev--scanner-blue.svg" alt="Skill: dev-scanner"/>
  <img src="https://img.shields.io/badge/browser--use-required-orange.svg" alt="Requires: browser-use"/>
  <img src="https://img.shields.io/badge/scans-8%20dimensions-brightgreen.svg" alt="8 Dimensions"/>
</p>

---

## ✨ What it does

一句话触发，自动打开浏览器逐页扫描前端项目，输出结构化审计报告。

```
审计项目
↓
自动检测项目类型 → 发现路由 → 逐页浏览器扫描 → 生成 .md 报告
```

<table>
  <tr>
    <th>扫描维度</th>
    <th>检查内容</th>
  </tr>
  <tr><td>🎨 UI/视觉设计</td><td>布局溢出、响应式断点、组件状态、对比度、裂图</td></tr>
  <tr><td>⚡ 功能交互</td><td>链接有效性、表单验证、核心流程、交互组件</td></tr>
  <tr><td>🐛 控制台/网络</td><td>JS 异常、API 失败、弃用警告、混合内容</td></tr>
  <tr><td>♿ 可访问性</td><td>键盘导航、焦点指示、ARIA 属性、触控目标</td></tr>
  <tr><td>🚀 性能</td><td>页面加载时间、大图检测、渲染阻塞资源</td></tr>
  <tr><td>📝 内容</td><td>占位文本残留、死链、Meta 标签、404 页面</td></tr>
  <tr><td>🔐 安全</td><td>密码存储、localStorage 数据、CSRF 防护</td></tr>
  <tr><td>📋 报告</td><td>严重度分级、代码定位、影响分析</td></tr>
</table>

---

## 📥 Install

将 `SKILL.md` 放入你的 skills 目录即可：

```bash
# ZCode
cp SKILL.md ~/.agents/skills/dev-scanner/SKILL.md

# 或者 clone 整个仓库
git clone https://github.com/1234fsdc/dev-scanner.git ~/.agents/skills/dev-scanner
```

### 前置依赖

- [browser-use](https://github.com/browser-use/browser-harness) — 浏览器 CDP 控制
- Chrome 浏览器（需开启远程调试）

```bash
# 开启 Chrome 远程调试
# 1. 打开 Chrome
# 2. 地址栏输入 chrome://inspect/#remote-debugging
# 3. 勾选 "Allow remote debugging for this browser instance"
```

---

## 🚀 Usage

说出以下关键词自动触发：

```
审计项目 / 扫描项目 / 全面检查 / 跑一遍检查
检查UI / 检查设计 / 检查页面
review project / audit project
```

### 示例对话

```
你：审计项目
AI：📋 发现 7 个页面路由
    ❌ 排除路由：无
    🔑 是否需要登录？(是/否)
    
你：不需要
    
AI：开始扫描... (逐页检查中)
    
    ✅ 扫描完成！生成报告：audit-reports/audit-report-2026-07-16.md
```

---

## 📊 Report Example

报告保存在 `项目根目录/audit-reports/audit-report-YYYY-MM-DD-HHmm.md`

```
# 📊 campus-second-hand-market - 浏览器审计报告

## 📈 概览统计
| 发现页面 | 总问题数 | 🔴 严重 | 🟠 主要 | 🟡 次要 |
|---------|---------|---------|---------|---------|
| 7       | 14      | 3       | 4       | 5       |

## 📄 逐页详情

### 🔐 /login (登录页)
#### 🟠 表单输入框缺少 name 属性
**现状**：登录表单的两个 input 元素都没有 name 属性...
**影响**：屏幕阅读器无法识别字段，密码管理器无法工作...
**代码位置**：`src/views/LoginView.vue:46`

### 🔒 /publish (发布页)
#### 🔴 路由守卫在无痕模式下可能失效
**现状**：路由守卫依赖 localStorage 判断登录态...
**影响**：新浏览器/无痕模式下，未登录用户可直接访问...
**代码位置**：`src/router/index.js:57-66`
```

**报告风格**：只描述问题（现状 + 影响 + 代码位置），不给修复建议。

---

## ⚠️ Known Limitations

browser-use 在扫描过程中有以下已知问题（已在 SKILL.md 中提供解决方案）：

| 问题 | 现象 | 解决方案 |
|------|------|----------|
| 大视口截图超时 | `capture_screenshot()` 在 2560px+ 视口卡死 | 固定 1440×900 视口 |
| SPA 渲染等待 | `wait_for_load()` 返回时内容未渲染 | `time.sleep(3)` 等待 |
| 循环导航超时 | 单次 heredoc 中多次 `goto_url` 断连 | shell 循环每页单独调用 |
| 多次 JS 调用超时 | 单次调用中 3+ 次 `js()` 累积延迟 | 合并为一次 JS 检查 |

---

## 📁 Structure

```
dev-scanner/
├── SKILL.md          # Skill 定义（核心文件）
└── README.md         # 本文件
```

---

## 🤝 Contributing

1. Fork 本仓库
2. 创建特性分支：`git checkout -b feature/amazing-feature`
3. 提交更改：`git commit -m 'Add amazing feature'`
4. 推送分支：`git push origin feature/amazing-feature`
5. 创建 Pull Request

---

## 📄 License

MIT License - 详见 [LICENSE](LICENSE)

---

<p align="center">
  <em>Built with ❤️ for the AI agent community</em>
</p>
