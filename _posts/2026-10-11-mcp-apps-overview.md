---
layout: post
title: "MCP Apps 是什么：为什么 MCP 工具需要一个界面"
date: 2026-10-11 12:00:00 +0800
categories: ["AI","MCP"]
tags: ["MCP Apps","Agent","交互界面"]
description: "MCP Apps 是 MCP 的可选 UI 扩展。本文从工具结果的交互需求出发，解释它适合什么任务、如何工作，并用一个简单的 Python 示例说明工具与界面的关系。"
source_url: "https://github.com/modelcontextprotocol/ext-apps/blob/main/docs/overview.md"
---

你让 Agent 展示一份分析结果，它返回了一段 JSON。

数据是对的，但接下来还要继续问：能不能只看华东？能不能点开某个城市？能不能换成同比趋势？如果每一次操作都要重新发一条消息，工具虽然能工作，交互却很慢。

MCP Apps 解决的就是这类问题：让 MCP 工具的结果，在对话里变成可以点击、筛选、填写和持续查看的界面。

## 为什么需要 MCP Apps

普通 MCP Tool 的输出通常是文本、图片、资源或结构化数据。这种方式很适合回答一个明确的问题，也适合把结果交给模型继续推理。

但有些任务从一开始就不是“返回一个答案”这么简单。

比如用户说：“把本季度销售额按地区展示出来。”

工具可以返回一组数字，模型也可以把它整理成表格。可用户很快还会想看地区明细、切换指标、筛选时间范围，或者比较两个区域。让用户把每一次点击都重新翻译成自然语言，会让本来很直观的操作变得绕远。

MCP Apps 为工具增加了一层可交互的 UI。它让 Server 继续负责工具和业务逻辑，同时允许 Host 在当前对话中加载一个 HTML 界面，把结果呈现成地图、表单、播放器或仪表盘。

这里的关键是“增加一层”。MCP Apps 是 MCP 的可选扩展，不会替代普通 MCP Server，也不会把工具的授权和业务执行交给界面。

## MCP Apps 是什么

MCP Apps 通常由三个角色组成：

- **MCP Server**：提供工具、资源和业务逻辑，同时注册 UI 资源；
- **Host**：聊天客户端或 AI 应用，负责发现工具、获取 UI、创建沙箱 `iframe`，并代理通信；
- **View / App**：运行在 `iframe` 中的 HTML 界面，负责展示结果和接收用户操作。

Server 会注册一个 `ui://` 资源，再在工具元数据里声明工具和 UI 的关系：

```json
{
  "_meta": {
    "ui": {
      "resourceUri": "ui://sales/analytics"
    }
  }
}
```

因此，MCP App 不是一个脱离 MCP Server 的新后端。它是“标准 MCP 工具 + 可选 UI + Host 代理通信”组成的一套交互体验。

## 哪些任务适合使用 MCP Apps

### 探索复杂数据

地图、图表和多维报表很适合用 UI 展示。用户可以点击区域查看明细，悬停查看数据，在不同指标之间切换，而不必为每个动作重新组织一段提示词。

### 配置选项很多的任务

部署配置、资源选择和表单提交往往包含很多相互关联的选项。界面可以一次展示这些选项，提供默认值和校验，减少来回确认。

### 查看富媒体

当用户需要查看 PDF、图片、视频或 3D 模型时，文字描述通常不够。嵌入式查看器可以直接提供缩放、旋转、平移或播放控制。

### 持续观察系统状态

实时指标、日志和监控面板需要持续更新。MCP App 可以保持界面状态，在数据变化时更新，而不要求用户反复询问“现在怎么样”。

### 多步骤工作流

审批费用、审查代码变更、分诊问题等任务需要逐项检查。导航控件、操作按钮和状态提示可以把这些步骤放在同一个工作面里。

## MCP Apps 是如何工作的

下面这张图展示了一次典型交互。图中的 Agent 承担了 Host 的角色，负责连接 MCP Server，并把工具结果交给 MCP App iframe。

![MCP Apps 的一次交互流程](/assets/images/08-mcp-apps/mcp-apps-flow.png)

流程可以拆成几步：

1. 用户提出需求，Agent 根据工具描述选择工具；
2. Agent 向 MCP Server 发起 `tools/call`；
3. Server 返回工具输入对应的结果；
4. Agent 把结果推送给已经挂载的 App；
5. 用户在 App 中筛选、翻页或点击操作；
6. App 通过 Host 发起新的工具调用；
7. Host 将请求转发给 MCP Server，拿到新数据后再推送回 App；
8. App 更新界面，也可以请求更新模型上下文。

App 和 Host 之间使用基于 JSON-RPC 的 MCP Apps bridge 通信。App 不直接访问聊天客户端的 DOM、Cookie 或本地存储，涉及 Server 的请求也由 Host 代理。

这个扩展是可选的。支持 MCP Apps 的 Host 可以预加载 `ui://` 资源，在沙箱 `iframe` 中渲染 HTML；不支持的 Host 仍然应该拿到普通文本或结构化结果。

## 哪些任务不适合使用 MCP Apps

如果结果只需要一句话、一个 JSON，或者主要交给模型继续推理，普通 MCP Tool 往往已经足够。增加 UI 会带来资源注册、Host 适配和安全策略，不必为了简单答案引入这些复杂度。

还要注意几个边界：

- `iframe` 提供的是隔离能力，不是完整的业务安全方案；
- UI 能访问哪些外部资源，需要由 CSP 等策略明确控制；
- Host 对最终显示模式拥有决定权，View 的请求不一定会被接受；
- UI 不负责替代输入校验、OAuth、数据权限和写操作确认；
- 一个客户端支持普通 MCP，不代表它已经支持 MCP Apps UI。

涉及发送、删除、发布或付款等不可逆动作时，仍然需要在 Server 和 Host 层设计授权、确认和回滚路径。

## 一个简单的 Demo

下面的例子提供一个时钟工具，并把一个简单的 HTML 页面和它关联起来。它展示的是 MCP Apps 的核心关系，真正的按钮、表单和分页交互还需要接入完整的浏览器端 bridge。

先安装 MCP Python SDK：

```bash
mkdir clock-mcp-app
cd clock-mcp-app
uv init
uv add "mcp[cli]"
```

`server.py`：

```python
from datetime import datetime, timezone

from mcp.server.apps import Apps, client_supports_apps
from mcp.server.mcpserver import MCPServer
from mcp.server.mcpserver.context import Context


CLOCK_HTML = """\
<!doctype html>
<title>Clock</title>
<h1 id="now">等待时间...</h1>
<script>
  window.addEventListener("message", (event) => {
    const text = event.data?.result?.content?.[0]?.text;
    if (text) document.getElementById("now").textContent = text;
  });
</script>
"""

apps = Apps()


@apps.tool(
    resource_uri="ui://clock/app.html",
    description="返回当前 UTC 时间。",
)
def get_time(ctx: Context) -> str:
    now = datetime.now(timezone.utc).isoformat().replace("+00:00", "Z")

    # 不支持 MCP Apps 的客户端仍得到有意义的文本结果。
    if not client_supports_apps(ctx):
        return f"当前 UTC 时间是 {now}。"

    return now


apps.add_html_resource(
    "ui://clock/app.html",
    CLOCK_HTML,
    title="Clock",
)

mcp = MCPServer("clock", extensions=[apps])
```

这个 Demo 里有三条关键关系：

- `@apps.tool(resource_uri=...)` 把工具和 `ui://clock/app.html` 关联起来；
- `apps.add_html_resource(...)` 注册真正要渲染的 HTML；
- `client_supports_apps(ctx)` 保证没有 UI 的客户端仍能得到文本结果。

启动服务：

```bash
uv run mcp run server.py --transport streamable-http
```

这个例子没有把 UI 当成工具的唯一出口。工具仍然可以独立执行，UI 只是让结果多了一种更适合用户操作的呈现方式。

MCP Apps 的价值也在这里：需要探索、配置、查看和持续操作时，让工具在对话里长出界面；只需要一个答案时，普通 MCP Tool 仍然是更简单的选择。

参考资料：[MCP Apps Overview](https://github.com/modelcontextprotocol/ext-apps/blob/main/docs/overview.md)、[MCP Apps 扩展规范](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx)、[MCP 官方 Apps 页面](https://modelcontextprotocol.io/extensions/apps/overview)、[MCP Python SDK：MCP Apps](https://py.sdk.modelcontextprotocol.io/advanced/apps/)。
