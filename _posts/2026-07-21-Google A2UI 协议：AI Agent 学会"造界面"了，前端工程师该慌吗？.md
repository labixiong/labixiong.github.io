---
layout:     post
title:      Google A2UI 协议：AI Agent 学会"造界面"了，前端工程师该慌吗？
subtitle:   Google A2UI 协议：AI Agent 学会"造界面"了，前端工程师该慌吗？
date:       2026-07-21
author:     LBX
header-img: img/bg-little-universe.jpg
catalog: true
tags:
    - 人工智能
    - AI
    - google
    - A2UI
---

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/fd64f4804e3744d09ba259b41b326871.jpeg#pic_center)


> 2025 年 12 月 15 日，Google 悄悄上线了一个叫 A2UI 的开源项目。没开发布会，没做营销，但它在 AI Agent 生态里炸开了一个口子——AI 终于不只会"说"文字了，它开始"造"界面了。

---

## 一、一个简单的场景：你更想要哪种交互？

你想让 AI 帮你订个餐厅，说："帮我订明天晚上 7 点 2 个人的桌位。"

**传统 Agent 的回应方式：**

```
Agent: 好的，请问是哪一天？
用户: 明天
Agent: 什么时间？
用户: 晚上 7 点
Agent: 几位？
用户: 2 位
Agent: 让我查一下……我们有 5:00、5:30、6:00、8:30、9:00 可选，你看哪个合适？
用户: 8:30 吧
Agent: 好的，已为您预订……
```

六轮对话，订了个餐。每多一个约束条件就多一次往返。你说烦不烦？

**A2UI 的回应方式：**

Agent 直接生成一个表单——日期选择器、时间选择器、人数输入框、提交按钮，一屏到位。用户点两下，完事。

这就是 A2UI（Agent-to-User Interface）要解决的核心问题：**让 AI Agent 不只能输出文字，还能输出可交互的界面。**

听起来简单，但背后的技术挑战远比你想象的大。

---

## 二、A2UI 到底是什么？

1. **它是一个协议**——定义了 AI Agent 如何向客户端描述 UI 的一套规范
2. **它用 JSON 说话**——Agent 输出的是声明式的 JSON 数据，不是可执行代码
3. **客户端负责渲染**——同一份 JSON，在 Web 上变成 React 组件，在手机上变成 Flutter Widget，在桌面端变成原生控件

用一句话总结：**Agent 负责"描述" UI，客户端负责"绘制" UI。**

这跟 React Server Components 的思路惊人地相似——服务端输出组件描述，客户端负责渲染。只不过 A2UI 把"服务端"换成了"AI Agent"，把"组件描述"变成了"JSON 蓝图"。

Google 官方博客里有一句话，这句话基本概括了 A2UI 的设计哲学：

> "We needed a way to transmit UI that is safe like data, but expressive like code."
> 
> 我们需要一种方式来传输 UI——像数据一样安全，像代码一样富有表现力。

---

## 三、为什么不用 HTML 或者 iframe？

Agent 要生成 UI，直接输出 HTML 不就行了？为什么要搞一个新协议？

确实，之前的方案就是这么做[SYSTEM_NOTE: Content compressed. Read the full version if true } } ] not running arbitrary code.

这 18 个基础组件覆盖了绝大多数常见交互场景。但如果你有特殊需求——比如需要一个地图组件、一个图表组件——A2UI 支持自定义 Catalog。客户端可以注册自己的组件类型，Agent 通过 Catalog ID 引用。

**这个设计本质上就是一个"设计系统 + 白名单"的思路。** 企业前端的最佳实践。

---

## 六、流式渲染：UI 也能"边想边说"

LLM 有一个特性叫"流式输出"——ChatGPT 回答问题时，文字是一个字一个字蹦出来的，不是等全部生成完才显示。

A2UI 把这个特性也搬到了 UI 上。

因为组件是扁平列表 + ID 引用，Agent 可以：

1. 先发出根组件 `Column`，带一个空的 children 列表
2. 再发出 `header` 组件（Text）
3. 再发出 `date_input` 组件（DateTimeInput）
4. 最后发出 `confirm_btn` 组件（Button）

每发出一个组件，客户端就可以立即渲染。用户看到的是界面在"生长"——先出现一个标题，再出现一个输入框，最后出现一个按钮。

这比"等 Agent 生成完整 JSON 再一次性渲染"体验好太多。

v0.9.1 的消息格式进一步优化了这个流程。每条消息都带 `version` 字段，组件格式更扁平（`"component": "Text"` 替代嵌套的 `{"Text": {...}}`），children 直接用数组：

```json
{
  "version": "v0.9.1",
  "updateComponents": {
    "surfaceId": "booking",
    "components": [
      { "id": "root", "component": "Column", "children": ["header", "date_input", "confirm_btn"] },
      { "id": "header", "component": "Text", "text": "预订餐厅", "variant": "h1" },
      { "id": "date_input", "component": "DateTimeInput", "value": { "path": "/booking/date" }, "enableDate": true },
      { "id": "confirm_btn", "component": "Button", "child": "btn_text", "variant": "primary", "action": { "event": { "name": "confirm" } } },
      { "id": "btn_text", "component": "Text", "text": "确认预订" }
    ]
  }
}
```

对比 v0.8 的写法，v0.9.1 更简洁、更扁平、token 效率更高。这种版本演进说明 Google 团队确实在认真打磨 LLM 生成的效率问题。

---

## 七、协议栈全景：A2UI 不是一个人在战斗

A2UI 不是孤立存在的。它是一个更大的 Agent 协议栈中的一层。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b561a6f434084ef3aa207ba08e0d8973.png#pic_center)


简单解释每一层：

| 协议 | 全称 | 解决什么问题 | 类比 |
|------|------|------------|------|
| MCP | Model Context Protocol | Agent 调用工具（搜索、数据库、API） | USB 接口 |
| A2A | Agent-to-Agent Protocol | Agent 之间互相协作 | HTTP |
| AG-UI | Agent-User Interaction Protocol | Agent 和前端实时通信（传输层） | WebSocket |
| **A2UI** | Agent-to-User Interface | **UI 长什么样（内容层）** | **HTML** |

A2UI 和 AG-UI 的关系最容易混淆。说通俗点：

- **AG-UI 是"快递公司"**——负责把包裹从 Agent 送到前端
- **A2UI 是"包裹里的东西"**——定义包裹里装的是什么格式的 UI 描述

它们不是竞争关系，而是互补关系。AG-UI 可以用 A2UI 作为数据格式来传输 UI。

Google 官方博客里也强调了这一点：A2UI 的 JSON payload 可以通过 A2A、AG-UI、SSE、WebSocket 等多种传输方式发送。A2UI 只管"说什么"，不管"怎么说"。

---

## 八、数据绑定：UI 和数据怎么同步？

UI 不只是静态的组件排列，还需要动态数据。比如一个列表组件要展示 Agent 检索到的多个餐厅名称，一个输入框需要回显用户之前填的值。

A2UI 引入了 `dataModelUpdate` 消息机制，使用 JSON Pointer（RFC 6901）路径进行数据绑定。

两种绑定方式：

```json
// 字面值（固定）
{ "text": { "literalString": "欢迎" } }

// 数据绑定（响应式）
{ "text": { "path": "/user/name" } }
```

当 `/user/name` 的值改变时，组件自动更新——不需要重新生成整个组件树。

这是 A2UI 的一个关键设计：**UI 结构和应用状态分离。** 组件树描述"长什么样"，数据模型描述"显示什么"。两者独立更新，互不干扰。

跟 React 的状态管理思路一样：UI = f(state)。只不过这里的 f 不是 React 组件函数，而是 A2UI 渲染器。

---

## 九、交互闭环：用户点击按钮后发生了什么？

UI 展示出来只是第一步，还需要处理用户交互。A2UI 的交互闭环如下：
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/e8b9398db5894ec28224f848e97180d9.png#pic_center)


1. Agent 发送 `surfaceUpdate` 消息，描述 UI 组件
2. Agent 发送 `dataModelUpdate` 消息，填充数据
3. 客户端渲染 UI，用户看到界面
4. 用户点击按钮 → 触发 `action`
5. 客户端将 action 事件回传给 Agent
6. Agent 处理事件，可能发送新的 `surfaceUpdate` 或 `dataModelUpdate`
7. UI 更新，循环继续

第 4 步的 action 机制如下：

```json
{
  "id": "confirm_btn",
  "component": "Button",
  "action": { "event": { "name": "confirm_booking" } }
}
```

用户点击按钮时，客户端会向 Agent 发送一条 `confirm_booking` 事件消息。Agent 收到后，可以执行业务逻辑（比如调用餐厅预订 API），然后更新 UI（显示"预订成功"）。

**这个闭环设计让 Agent 和 UI 之间形成了一个完整的请求-响应循环。** Agent 不只是单向输出 UI，还能接收 UI 事件的反馈。

---

## 十、对前端工程师意味着什么？

### 10.1 A2UI 本质上是"AI 版的 Server Components"

React Server Components 的核心思想是：服务端输出组件描述（JSON），客户端负责渲染。A2UI 做的事情几乎一模一样——只不过把"服务端"换成了"AI Agent"。

这不是巧合。两者解决的是同一个问题：**如何安全地跨边界传输 UI。** RSC 解决的是服务端到前端的 UI 传输，A2UI 解决的是 Agent 到客户端的 UI 传输。

如果你理解了 RSC，理解 A2UI 就没有任何障碍。

### 10.2 扁平列表设计暗合虚拟 DOM 的思路

A2UI 用邻接表（扁平列表 + ID 引用）而不是嵌套树来描述组件结构。这个设计决策不是拍脑袋想的，它解决的是 LLM 生成的实际问题：

- **嵌套树**：LLM 必须一次性生成完整的树结构，中间不能出错。如果第 3 层嵌套少了一个括号，整棵树作废
- **扁平列表**：LLM 可以一个组件一个组件地生成，每个组件独立。错了只改那一个组件，不影响其他

这跟 React 虚拟 DOM 的 diffing 思路是一致的。React 内部在做 reconciliation 时，也不是操作嵌套树，而是操作扁平的 fiber 节点链表。A2UI 的邻接表模型，在数据结构上就跟 fiber 架构对齐了。

我认为这不是过度解读——Google 团队里一定有前端架构师参与了设计。

### 10.3 Catalog 就是设计系统的白名单化

企业前端的最佳实践是建立设计系统（Design System）——统一定义按钮、输入框、卡片等组件的样式和行为规范。A2UI 的 Catalog 机制本质上就是这个思路的协议化：

- 组件必须在 Catalog 中注册才能使用（白名单）
- Agent 只能引用已注册的组件类型（不能注入任意代码）
- 客户端控制组件的实际渲染（样式、主题、无障碍）

**这是"安全"和"灵活"的平衡点。** Agent 有表达 UI 的自由度，但没有越权的能力。

### 10.4 说实话，现在生态还不成熟

A2UI 目前版本是 v0.9.1，v1.0 还是候选状态。我梳理了一下现状：

| 方面 | 现状 | 成熟度 |
|------|------|--------|
| 协议规范 | v0.9.1 稳定，v1.0 候选 | ⭐⭐⭐ |
| Web 渲染器 | Lit ✅ / Angular ✅ / React ❌（2026 Q1 计划） | ⭐⭐ |
| 移动端渲染器 | Flutter ✅ / SwiftUI 开发中 / Compose 开发中 | ⭐⭐ |
| 桌面端渲染器 | Flutter 可复用 | ⭐⭐ |
| 开发工具 | A2UI Composer（调试工具） | ⭐⭐ |
| 社区生态 | CopilotKit 联合发布，其他框架集成少 | ⭐ |
| 生产案例 | Google Opal、Gemini Enterprise、Flutter GenUI SDK | ⭐⭐⭐ |

**React 渲染器还没有正式发布，这对国内前端开发者来说是个硬伤。** 大部分国内前端项目用 React 或 Vue，而 A2UI 目前只支持 Lit 和 Angular。虽然 v1.0 计划支持 React，但时间点是"2026 Q1"——现在已经 7 月了，还没看到正式版。

Vue 渲染器更是连计划都没有。

### 10.5 前端工程师该慌吗？

不用慌，但要关注。

**为什么不慌呢？**

1. A2UI 解决的是 Agent 跨信任边界生成 UI 的问题，这不是大部分前端开发者日常面临的场景
2. 目前主要应用在 AI Agent 产品（Gemini、Google Workspace），传统前端应用短期不受影响
3. React/Vue 渲染器还没就绪，实际能用上的场景有限
4. 组件 Catalog 的设计需要前端工程师参与——这反而是新的机会

**要关注的原因：**

1. Google 在自家生产系统上已经验证了 A2UI 的可行性，这个协议不是玩具
2. 如果多 Agent 协作成为主流架构（A2A 协议被 Linux 基金会接管说明了这个趋势），A2UI 就是刚需——远程 Agent 必须通过某种协议向客户端传递 UI
3. "AI 描述 UI，前端渲染 UI"这个模式，可能会改变前端开发的工作方式——前端工程师不只为人类用户构建界面，还要为 AI Agent 提供渲染环境
4. Catalog 机制意味着前端工程师的角色会向"组件生态建设者"倾斜——你定义的可信组件库，决定了 AI 能生成什么样的 UI

我的判断是：**A2UI 短期不会颠覆前端开发，但中长期会改变前端工程师的技能重心。** 从"写 UI"转向"设计 UI 的生成规则"——定义组件 Catalog、设计渲染规范、保障安全边界。这可能是未来 3-5 年前端工程师的新方向之一。

---

## 十一、实战：从零跑通一个 A2UI 完整流程

光说理论不够，写个能跑的 Demo。

这一节用 Node.js + 原生 JS 搭一个完整的 A2UI 交互流程——Agent 端发送 A2UI 消息，客户端手写一个迷你渲染器，用户点击按钮后事件回传给 Agent，Agent 更新 UI。

**三个文件，copy 下来就能跑。**

### 11.1 整体架构

```
a2ui-demo/
├── package.json      ← 一个依赖：express
├── server.js         ← Agent 端：发送 A2UI 消息 + 处理用户事件
└── index.html        ← 客户端：迷你 A2UI 渲染器（vanilla JS）

         ┌─ SSE (Agent → Client) ──→ A2UI JSON 消息流
Agent ───┤
         └─ POST (Client → Agent) ─→ 用户 action 事件
```

两条通道：SSE 负责 Agent → Client 推送 A2UI 消息，POST 负责 Client → Agent 回传用户交互事件。真实场景中 Agent 侧是一个 LLM + ADK，这里用 setTimeout 模拟 LLM 逐个生成组件的流式效果。

### 11.2 Agent 端：server.js

先建 `package.json`：

```json
{
  "name": "a2ui-demo",
  "version": "1.0.0",
  "scripts": { "start": "node server.js" },
  "dependencies": { "express": "^4.18.0" }
}
```

然后是 `server.js`——这个文件做了两件事：通过 SSE 向客户端推送 A2UI 消息，以及接收客户端回传的 action 事件：

```javascript
const express = require('express');
const app = express();
app.use(express.static('.'));
app.use(express.json());

// SSE 工具函数：把 JSON 包装成 SSE 消息
function sendSSE(res, data) {
  res.write(`data: ${JSON.stringify(data)}\n\n`);
}

// ============ Agent → Client：SSE 流式推送 A2UI 消息 ============
app.get('/a2ui-stream', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.flushHeaders();

  // 第 1 步：创建 Surface（渲染上下文）
  sendSSE(res, {
    version: "v0.9.1",
    createSurface: {
      surfaceId: "booking",
      catalogId: "https://a2ui.org/specification/v0_9_1/catalogs/basic/catalog.json"
    }
  });

  // 第 2 步：流式发送组件（模拟 LLM 逐个生成）
  // 先发 root + title —— 用户立刻看到标题
  setTimeout(() => sendSSE(res, {
    version: "v0.9.1",
    updateComponents: {
      surfaceId: "booking",
      components: [
        { id: "root", component: "Column", children: ["title", "desc", "date_input", "party_select", "submit_btn", "submit_text"] },
        { id: "title", component: "Text", text: "预订餐厅桌位", variant: "h1" },
      ]
    }
  }), 300);

  // 再发描述 + 日期选择器
  setTimeout(() => sendSSE(res, {
    version: "v0.9.1",
    updateComponents: {
      surfaceId: "booking",
      components: [
        { id: "desc", component: "Text", text: "填写以下信息，我们会为您安排座位" },
        { id: "date_input", component: "DateTimeInput", value: { path: "/booking/date" }, enableDate: true, enableTime: true },
      ]
    }
  }), 600);

  // 最后发人数选择 + 提交按钮
  setTimeout(() => sendSSE(res, {
    version: "v0.9.1",
    updateComponents: {
      surfaceId: "booking",
      components: [
        {
          id: "party_select", component: "SelectionInput", value: { path: "/booking/party_size" }, label: "用餐人数",
          options: [
            { label: "2 人", value: 2 }, { label: "4 人", value: 4 },
            { label: "6 人", value: 6 }, { label: "8 人", value: 8 },
          ]
        },
        { id: "submit_btn", component: "Button", child: "submit_text", variant: "primary", action: { event: { name: "confirm_booking" } } },
        { id: "submit_text", component: "Text", text: "确认预订" },
      ]
    }
  }), 900);

  // 第 3 步：初始化数据模型
  setTimeout(() => sendSSE(res, {
    version: "v0.9.1",
    updateDataModel: {
      surfaceId: "booking", path: "/booking",
      value: { date: "", party_size: 2 }
    }
  }), 1200);
});

// ============ Client → Agent：接收用户交互事件 ============
app.post('/a2ui-event', (req, res) => {
  const { event } = req.body;
  console.log(`[Agent] 收到事件: ${event.name}`, event.data);

  if (event.name === 'confirm_booking') {
    const { date, party_size } = event.data?.booking || {};

    // 😱 没选日期 → 返回错误提示
    if (!date) {
      return res.json({
        version: "v0.9.1",
        messages: [{
          updateComponents: {
            surfaceId: "booking",
            components: [
              { id: "desc", component: "Alert", variant: "error", text: "⚠️ 请先选择预订日期和时间" },
            ]
          }
        }]
      });
    }

    // ✅ 验证通过 → 更新 UI 为成功状态
    const dateStr = new Date(date).toLocaleString('zh-CN', {
      month: 'long', day: 'numeric', hour: '2-digit', minute: '2-digit'
    });

    res.json({
      version: "v0.9.1",
      messages: [{
        updateComponents: {
          surfaceId: "booking",
          components: [
            { id: "title", component: "Text", text: "✅ 预订成功！", variant: "h2" },
            { id: "desc", component: "Alert", variant: "success", text: `${dateStr}，${party_size} 位，确认邮件已发送` },
            { id: "date_input", component: "Text", text: "期待您的光临 🎉" },
            { id: "party_select", component: "Text", text: "" },
            { id: "submit_btn", component: "Button", child: "submit_text", variant: "secondary", action: { event: { name: "done" } } },
            { id: "submit_text", component: "Text", text: "完成" },
          ]
        }
      }]
    });
    return;
  }

  res.json({ messages: [] });
});

app.listen(3000, () => console.log('🚀 A2UI Demo: http://localhost:3000'));
```

注意看 Agent 发送组件的节奏——`setTimeout` 300ms、600ms、900ms 分三批发送。这就是流式渲染的模拟：LLM 生成组件有先后顺序，先出标题，再出输入框，最后出按钮。用户看到 UI 在"生长"，而不是等全部生成完才闪现。

另一个关键点：Agent 回传事件时，只更新发生变化的组件（通过 ID 定位）。`date_input` 被替换成了纯文本 `Text` 组件——不需要删除原组件再新建，直接用相同 ID 覆盖就行。这就是扁平列表的局部更新优势。

### 11.3 客户端：index.html（手写迷你 A2UI 渲染器）

这是整个 Demo 的灵魂——一个用 vanilla JS 写的 A2UI 渲染器，不到 200 行代码，做的事情和 Google 官方的 Lit 渲染器原理完全一样：

```html
<!DOCTYPE html>
<html lang="zh">
<head>
  <meta charset="UTF-8">
  <title>A2UI 餐厅预订 Demo</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: -apple-system, 'PingFang SC', sans-serif; background: #f0f2f5; padding: 20px; }
    #app { max-width: 480px; margin: 40px auto; background: #fff; border-radius: 12px; padding: 24px; box-shadow: 0 2px 12px rgba(0,0,0,0.08); min-height: 300px; }
    /* A2UI 组件样式 —— 每种 Catalog 组件类型对应一个 CSS class */
    .a2ui-column { display: flex; flex-direction: column; gap: 16px; }
    .a2ui-text { font-size: 14px; color: #666; }
    .a2ui-text-h1 { font-size: 24px; font-weight: 700; color: #1a1a1a; }
    .a2ui-text-h2 { font-size: 20px; font-weight: 600; color: #1a1a1a; }
    .a2ui-input, .a2ui-select { padding: 10px 12px; border: 1px solid #ddd; border-radius: 8px; font-size: 14px; width: 100%; }
    .a2ui-input:focus { outline: none; border-color: #4285f4; }
    .a2ui-button { padding: 12px; border: none; border-radius: 8px; font-size: 14px; cursor: pointer; }
    .a2ui-button-primary { background: #4285f4; color: #fff; }
    .a2ui-button-secondary { background: #e8e8e8; color: #333; }
    .a2ui-alert { padding: 10px 12px; border-radius: 8px; font-size: 14px; }
    .a2ui-alert-error { background: #fef2f2; color: #dc2626; }
    .a2ui-alert-success { background: #f0fdf4; color: #16a34a; }
    .a2ui-loading { text-align: center; padding: 40px; color: #999; }
  </style>
</head>
<body>
  <div id="app"><div class="a2ui-loading">⏳ 等待 Agent 生成 UI...</div></div>

  <script>
    // ========================================
    // 迷你 A2UI 渲染器
    // 做三件事：
    // 1. SSE 接收 Agent 发来的 A2UI JSON 消息
    // 2. 解析消息，维护组件邻接表 + 数据模型，渲染到 DOM
    // 3. 用户交互时，把 action 事件 POST 回 Agent
    // ========================================

    // ---- 全局状态 ----
    const dataModel = {};       // 应用状态
    const components = {};      // 组件邻接表 { id: componentDef }
    let surfaceId = null;

    // ---- 1. SSE 连接 ----
    const evtSrc = new EventSource('/a2ui-stream');
    evtSrc.onmessage = (e) => handleMessage(JSON.parse(e.data));

    // ---- 2. 消息处理 ----
    function handleMessage(msg) {
      if (msg.createSurface) {
        surfaceId = msg.createSurface.surfaceId;
        console.log('[A2UI] Surface 已创建:', surfaceId);
      }

      if (msg.updateComponents) {
        // 合并到邻接表：相同 ID 直接覆盖
        msg.updateComponents.components.forEach(c => { components[c.id] = c; });
        console.log('[A2UI] 组件更新:', msg.updateComponents.components.map(c => c.id).join(', '));
        render();
      }

      if (msg.updateDataModel) {
        setByPath(dataModel, msg.updateDataModel.path, msg.updateDataModel.value);
        console.log('[A2UI] 数据更新:', msg.updateDataModel.path, '=', msg.updateDataModel.value);
        render();
      }
    }

    // ---- 3. 渲染：从 root 开始递归构建 DOM ----
    function render() {
      const root = components['root'];
      if (!root) return;  // root 还没来，等
      const app = document.getElementById('app');
      app.innerHTML = '';
      app.appendChild(renderComponent(root));
    }

    function renderComponent(comp) {
      const el = createDOMElement(comp);
      // children：数组形式的子组件（Column/Row）
      if (comp.children) {
        comp.children.forEach(childId => {
          const child = components[childId];
          if (child) el.appendChild(renderComponent(child));
        });
      }
      // child：单个子组件（Button 的文本等）
      if (comp.child) {
        const child = components[comp.child];
        if (child) el.appendChild(renderComponent(child));
      }
      return el;
    }

    // ---- 组件 → DOM 映射（渲染器核心）----
    // 每种 Catalog 组件类型在这里对应一个 DOM 创建逻辑
    function createDOMElement(comp) {
      switch (comp.component) {
        case 'Column': {
          const div = document.createElement('div');
          div.className = 'a2ui-column';
          return div;
        }
        case 'Text': {
          const span = document.createElement('span');
          span.className = `a2ui-text a2ui-text-${comp.variant || ''}`;
          span.textContent = resolveValue(comp.text);
          return span;
        }
        case 'DateTimeInput': {
          const input = document.createElement('input');
          input.type = 'datetime-local';
          input.className = 'a2ui-input';
          input.value = resolveValue(comp.value) || '';
          // 用户修改 → 更新数据模型（响应式）
          input.addEventListener('change', () => {
            if (comp.value?.path) {
              setByPath(dataModel, comp.value.path, input.value);
              console.log('[用户输入]', comp.value.path, '=', input.value);
            }
          });
          return input;
        }
        case 'SelectionInput': {
          const select = document.createElement('select');
          select.className = 'a2ui-select';
          (comp.options || []).forEach(opt => {
            const option = document.createElement('option');
            option.value = opt.value;
            option.textContent = opt.label;
            select.appendChild(option);
          });
          select.value = resolveValue(comp.value) ?? '';
          select.addEventListener('change', () => {
            if (comp.value?.path) {
              setByPath(dataModel, comp.value.path, Number(select.value));
            }
          });
          return select;
        }
        case 'Button': {
          const btn = document.createElement('button');
          btn.className = `a2ui-button a2ui-button-${comp.variant || 'primary'}`;
          // action 绑定：点击 → 发送事件给 Agent
          if (comp.action?.event) {
            btn.addEventListener('click', () => {
              console.log('[用户点击] 事件:', comp.action.event.name);
              sendEvent(comp.action.event);
            });
          }
          return btn;
        }
        case 'Alert': {
          const div = document.createElement('div');
          div.className = `a2ui-alert a2ui-alert-${comp.variant || 'info'}`;
          div.textContent = resolveValue(comp.text);
          return div;
        }
        default: {
          // 未知组件类型 → 兜底
          const div = document.createElement('div');
          div.style.cssText = 'padding:8px;color:#999;font-size:12px;border:1px dashed #ddd;';
          div.textContent = `[未知组件: ${comp.component}]`;
          return div;
        }
      }
    }

    // ---- 数据绑定工具 ----
    // 解析 { path: "/xxx" } 或 { literalString: "xxx" }
    function resolveValue(val) {
      if (val == null) return '';
      if (typeof val === 'string') return val;
      if (val.literalString != null) return val.literalString;
      if (val.path) return getByPath(dataModel, val.path);
      return String(val);
    }

    // JSON Pointer（RFC 6901）：/booking/date → dataModel.booking.date
    function getByPath(obj, path) {
      return path.split('/').filter(Boolean).reduce((cur, p) => cur?.[p], obj) ?? '';
    }

    function setByPath(obj, path, value) {
      const parts = path.split('/').filter(Boolean);
      let cur = obj;
      for (let i = 0; i < parts.length - 1; i++) {
        if (!cur[parts[i]]) cur[parts[i]] = {};
        cur = cur[parts[i]];
      }
      cur[parts[parts.length - 1]] = value;
    }

    // ---- 4. 用户交互：把 action 事件 POST 回 Agent ----
    async function sendEvent(event) {
      const res = await fetch('/a2ui-event', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ event: { ...event, surfaceId, data: dataModel } })
      });
      const { messages } = await res.json();
      // Agent 返回的新消息 → 逐条处理（更新 UI）
      if (messages) messages.forEach(handleMessage);
    }
  </script>
</body>
</html>
```

渲染器的核心逻辑就三步：

1. **收消息** — SSE `onmessage` 接收 A2UI JSON，按消息类型（`createSurface` / `updateComponents` / `updateDataModel`）更新全局状态
2. **建 DOM** — 从 `root` 开始，通过 `children` / `child` 的 ID 引用递归查找子组件，调 `createDOMElement` 映射成真实 DOM 元素
3. **发事件** — 用户点击 Button 时，取出 `action.event` 里的事件名，连同当前 `dataModel` 一起 POST 给 Agent；Agent 返回的新消息再走第 1 步

`resolveValue` 函数处理了两种数据绑定形式：`{ literalString: "固定文字" }` 直接返回字面值，`{ path: "/booking/date" }` 从 `dataModel` 里按 JSON Pointer 路径取值。这就是 A2UI 的 UI 与数据分离机制在渲染器侧的实现。

### 11.4 跑起来看看

三个文件准备好后：

```bash
npm install      # 装一个 express
node server.js   # 启动
```

浏览器打开 `http://localhost:3000`，你会看到：

**0~0.3 秒**：页面显示"⏳ 等待 Agent 生成 UI..."

**0.3 秒**：标题"预订餐厅桌位"先出现——因为 root + title 最先到

**0.6 秒**：描述文字 + 日期选择器出现——UI 在"生长"

**0.9 秒**：人数下拉框 + "确认预订"按钮出现——表单完整了

**1.2 秒**：数据模型初始化完成，下拉框默认选中"2 人"

这时候你可以交互了：

- **不选日期直接点"确认预订"** → Agent 返回红色错误提示"⚠️ 请先选择预订日期和时间"
- **选好日期和人数后点击"确认预订"** → Agent 返回成功状态，标题变成"✅ 预订成功！"，按钮变成"完成"

打开浏览器 DevTools 的 Console，你能看到完整的消息流：

```
[A2UI] Surface 已创建: booking
[A2UI] 组件更新: root, title
[A2UI] 组件更新: desc, date_input
[A2UI] 组件更新: party_select, submit_btn, submit_text
[A2UI] 数据更新: /booking = { date: "", party_size: 2 }
[用户点击] 事件: confirm_booking
[→ Agent] 事件: confirm_booking 数据: { booking: { date: "2026-07-22T19:00", party_size: 4 } }
[A2UI] 组件更新: title, desc, date_input, party_select, submit_btn, submit_text
```

这就是 A2UI 的完整交互闭环——Agent 用 JSON "描述" UI，客户端"渲染" UI，用户交互通过 action 事件回传，Agent 处理后再更新 UI。整个过程中没有一个 `document.write`，没有一段可执行代码从 Agent 传到客户端。**安全，是因为传的是数据，不是代码。**

---

## 十二、A2UI vs 其他方案：一张表搞清

| 方案 | 主导方 | 核心思路 | 安全性 | 跨平台 | LLM 友好 | 成熟度 |
|------|--------|---------|--------|--------|---------|--------|
| **A2UI** | Google | 声明式 JSON + 可信 Catalog | 高（纯数据） | ✅ | ✅ 扁平列表 | ⭐⭐ |
| iframe + HTML | 传统方案 | Agent 输出 HTML，沙箱隔离 | 中（沙箱） | ❌ | ❌ HTML 嵌套 | ⭐⭐⭐⭐ |
| CopilotKit / AG-UI | CopilotKit | 前端预定义组件，Agent 返回状态 | 高（前端主导） | ✅ | 中 | ⭐⭐⭐ |
| Vercel AI SDK GenUI | Vercel | React Server Components 思路 | 高（RSC 安全模型） | ❌ 仅 React | 中 | ⭐⭐⭐ |
| OpenAI Function Calling | OpenAI | Agent 返回结构化数据，前端自行渲染 | 高 | ✅ | ✅ | ⭐⭐⭐⭐ |

几个关键区别：

1. **A2UI vs iframe**：A2UI 是纯数据，iframe 是代码执行。安全性完全不在一个量级
2. **A2UI vs CopilotKit**：CopilotKit 的 UI 组件是前端预定义的，Agent 只返回状态数据；A2UI 的 UI 结构由 Agent 动态生成，灵活度更高
3. **A2UI vs Vercel AI SDK**：思路最接近，但 Vercel 的方案绑死 React，A2UI 框架无关
4. **A2UI vs Function Calling**：Function Calling 返回业务数据，前端自行决定 UI；A2UI 返回 UI 描述本身。层次不同

**我认为 A2UI 最大的差异化在于"跨信任边界"。** 其他方案都假设 Agent 和前端在同一信任域内，只有 A2UI 明确面向"远程 Agent 跨组织传递 UI"的场景。这是多 Agent 时代的刚需。

---

## 十三、总结：A2UI 的知识图谱

```
A2UI 协议核心知识图谱
│
├─ 定位：Agent-to-User Interface，声明式 UI 协议
│   ├─ 创建者：Google，2025/12/15 开源，Apache 2.0
│   ├─ 当前版本：v0.9.1（v1.0 候选）
│   └─ 核心理念：UI 如数据般安全，如代码般富有表现力
│
├─ 核心设计
│   ├─ 声明式 JSON（非可执行代码）
│   ├─ 扁平列表 + ID 引用（邻接表模型）
│   ├─ 可信 Catalog（白名单组件注册）
│   ├─ 数据绑定（JSON Pointer, RFC 6901）
│   └─ 流式渲染（组件增量发送）
│
├─ 消息类型
│   ├─ createSurface：创建渲染上下文
│   ├─ updateComponents：更新组件描述
│   ├─ updateDataModel：更新应用状态
│   └─ deleteSurface：清理 UI
│
├─ Basic Catalog（18 个基础组件）
│   ├─ 布局：Column, Row, Card, Container
│   ├─ 文本：Text
│   ├─ 输入：TextInput, DateTimeInput, SelectionInput, SliderInput
│   ├─ 交互：Button, Link, Checkbox
│   ├─ 展示：Image, Icon, List, Table, Progress
│   └─ 反馈：Alert, Tooltip
│
├─ 协议栈定位
│   ├─ MCP：Agent ↔ Tool
│   ├─ A2A：Agent ↔ Agent
│   ├─ AG-UI：传输层（Agent ↔ Frontend 通信）
│   └─ A2UI：内容层（UI 描述格式）
│
├─ 渲染器支持
│   ├─ Web：Lit ✅ / Angular ✅ / React 开发中
│   ├─ 移动：Flutter ✅ / SwiftUI 开发中 / Compose 开发中
│   └─ 自定义渲染器：支持
│
└─ 对前端工程师的意义
    ├─ 短期：React/Vue 渲染器未就绪，影响有限
    ├─ 中期：多 Agent 场景下的 UI 安全传输刚需
    └─ 长期：前端角色从"写 UI"转向"设计 UI 生成规则"
```

*参考链接：*

- [A2UI 官方文档](https://a2ui.org)
- [Google Developers Blog: Introducing A2UI](https://developers.googleblog.com/introducing-a2ui-an-open-project-for-agent-driven-interfaces)
- [A2UI 规范 v1.0 候选版](https://a2ui.org/specification/v1.0-a2ui)
- [A2UI vs AG-UI 对比](https://a2ui.sh/articles/a2ui-vs-ag-ui)
- [Google Codelabs: ADK + A2UI 前端体验](https://codelabs.developers.google.com/next26/adk-a2ui)

