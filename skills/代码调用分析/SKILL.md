---
name: 代码调用分析
version: 1.0.0
description: Analyze source code call chains, class dependencies, and API interfaces. Generates an interactive HTML layered architecture diagram. Use when the user wants to analyze code structure, understand call relationships, or generate a code architecture visualization.
description_zh: 分析源代码的调用链、类依赖和 API 接口，生成交互式 HTML 分层架构图。当用户需要分析代码结构、理解调用关系或生成代码架构图时使用。
user-invocable: true
argument-hint: 选择代码目录作为分析入口
---

# 代码调用分析

对用户指定的代码目录进行多维度静态分析，生成一个独立的交互式 HTML 分层架构图。

## 分析工作流

### 第一步：确定分析范围

用户会选择一个工作目录。按以下逻辑判断入口类型：

**情况 A：目录含项目配置文件**（setup.py / pom.xml / go.mod / package.json）
- 以项目根目录为分析范围
- 从配置文件提取项目名称、依赖、入口点信息

**情况 B：纯源码目录**
- 目录下所有源码文件为分析范围
- 递归扫描子目录

**排除目录**：`__pycache__`, `node_modules`, `.git`, `build`, `dist`, `target`, `vendor`, `.idea`, `.vscode`, `tests`, `test`（除非用户明确要求分析测试代码）

使用 Glob 工具扫描文件列表，按扩展名统计：
- Python: `.py`
- Java: `.java`
- Go: `.go`
- Shell: `.sh`
- 配置: `.yaml`, `.yml`, `.json`, `.xml`, `.ini`, `.conf`

以文件数最多的语言为主要分析语言。

### 第二步：分层架构分析

将所有源码按职责分类到以下六个架构层。不是每层都必须有——根据实际代码情况分类。

| 层级 | 标识名 | 职责 | 典型特征 |
|------|--------|------|----------|
| 入口层 | entry | 程序启动、CLI 入口、脚本入口 | `main()`, `if __name__`, setup.py entry_points, 可执行脚本 |
| 接口层 | api | HTTP/API 接口定义与路由 | Flask/Django route, Spring `@Controller`, net/http handler |
| 控制层 | controller | 请求校验、参数解析、调度转发 | `api_impl`, `handler`, `controller`, 参数校验逻辑 |
| 服务层 | service | 核心业务逻辑、编排流程 | `service`, `manager`, `engine`, `dispatcher`, 状态机 |
| 数据层 | data | 数据持久化与访问 | DAO, repository, ORM model, SQL 语句, DB 连接 |
| 模型层 | model | 数据模型、常量、配置定义 | `class XxxModel`, dataclass, enum, constants, config |
| 基础设施层 | infra | 通用工具、中间件、外部 SDK | util, helper, middleware, client SDK, 第三方库封装 |

分类方法：
1. 先按文件名/目录名关键词匹配（如含 `controller` 归控制层，含 `model` 归模型层）
2. 再按文件内容中的类定义和装饰器辅助判断
3. 无法明确分类的文件归入服务层

### 第三步：函数调用链提取

读取每个源码文件，提取以下关系：

**跨模块/跨类调用**（重点）：
- 模块 A 的函数 F 调用了模块 B 的函数 G
- 记录为：`A.F → B.G`

**类继承关系**：
- 类 X 继承类 Y
- 记录为：`X extends Y`

**外部服务调用**：
- HTTP 请求（requests, urllib, HttpClient, net/http）
- DNS 查询
- K8s API 调用
- 消息队列
- 远程过程调用

**过滤规则**：
- 跳过标准库内部调用（如 `os.path`, `json.loads`, `logging.info`）
- 跳过纯工具函数内部调用（如字符串拼接、类型转换）
- 重点保留：跨模块调用、外部服务调用、关键业务函数调用

**实现细节与异常模式提取**（针对每个模块）：

在完成调用链提取后，对每个模块额外提取以下信息，用于填充 HTML 架构图的详情区块：

1. **实现原理摘要**（2~4 句话）：
   - 核心算法或业务逻辑（如优先级队列调度、轮询检测、状态机驱动）
   - 关键数据结构（如 dict 缓存、PriorityQueue、DataFrame）
   - 并发/通信模型（如多线程+Timer超时、异步回调、进程池）
   - 缓存与持久化策略（如 pickle 文件缓存、内存 LRU、定时刷盘）

2. **常见异常与陷阱**（每个模块至少 3 条）：
   - 扫描 `try/except` 块，识别裸 `except`、吞异常（空 except body）
   - 识别无重试的网络调用（HTTP/RPC 调用没有 retry 机制）
   - 识别资源泄漏风险（未关闭的文件句柄、数据库连接、socket）
   - 识别命令注入风险（`os.system()`, `subprocess.Popen(shell=True)`, `commands.getstatusoutput()`）
   - 识别硬编码超时（或缺失超时）的网络操作
   - 识别硬编码路径、IP、凭据
   - 识别并发安全问题（共享可变状态无锁保护）

每条异常记录格式：`{问题描述} | {影响范围} | {建议修复方式}`

### 第四步：API 接口识别

扫描代码中的 HTTP/API 接口定义：

**Python**：
- Flask: `@app.route()`, `@blueprint.route()`
- Django: `urls.py` 中的 `path()`, `url()`
- 自定义: `BaseHTTPRequestHandler`, WSGI application
- 搜索关键词: `route`, `path`, `url_map`, `add_url_rule`

**Java**：
- Spring: `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`
- Servlet: `doGet`, `doPost` 方法

**Go**：
- `http.HandleFunc()`, `http.Handle()`
- `mux.HandleFunc()`, `router.Handle()`
- Gin: `r.GET()`, `r.POST()`

每个接口记录：HTTP 方法、URL 路径、处理函数、所在文件、简要描述。

### 第五步：外部依赖分析

识别代码中的外部依赖：
- 第三方库 import（区分标准库和第三方库）
- 数据库连接（MySQL, PostgreSQL, Redis 等）
- 外部服务调用（HTTP client 调用、RPC 调用）
- 配置/注册中心（ZooKeeper, Nacos, etcd 等）
- 消息中间件（Kafka, RocketMQ, RabbitMQ 等）

### 第六步：生成 HTML 架构图

生成一个完全独立的 HTML 文件，所有 CSS 和 JS 内嵌。采用暗色主题 + 横向 Tab 导航布局。

**整体页面结构**（按顺序）：

1. **标题区**：项目名称、分析时间、主要语言标识
2. **统计卡片**：总文件数、总函数/方法数、总类数、API 接口数、外部依赖数、架构层数
3. **横向 Tab 导航栏**（核心交互）：
   - 第一个 Tab 固定为「架构总览」
   - 后续 Tab 为各架构层（仅输出有模块的层）
   - 点击 Tab 切换显示对应面板，当前选中 Tab 高亮
4. **Tab 面板内容**
5. **API 接口列表**（表格，在 Tab 区域下方）
6. **外部依赖清单**（分类卡片）
7. **图例**

**「架构总览」Tab 内容**：

- **子模块对照表**：列出所有子模块名称、所属架构层、文件数、核心职责（一句话）
- **关键调用链汇总**：跨模块的完整调用路径，用 `→` 箭头串联
- **外部依赖分类**：按类型（第三方库、数据库、外部服务、中间件）分组
- **类继承关系**：列出所有 `class X extends Y` 关系

**各架构层 Tab 内容**：

每个层级的 Tab 面板内，用 CSS Grid 排列该层的模块卡片。每张模块卡片包含以下区块：

| 区块 | 内容 | 展示方式 |
|------|------|----------|
| 模块头 | 模块名 + 所属文件路径 | 卡片标题栏，点击展开/折叠 |
| 实现原理 | 核心逻辑、数据结构、通信协议的详述 | 段落文本（2~4 句话） |
| 关键函数列表 | 函数名 + 参数签名 | 等宽字体列表 |
| 调用关系 | 调用了谁、被谁调用、外部服务调用 | 箭头 `→` 表示 |
| 常见异常与陷阱 | 每个模块的典型问题 | 三列表格：问题 / 影响 / 建议 |

**异常陷阱表格**格式：
```html
<table class="err-table">
  <thead><tr><th>问题</th><th>影响</th><th>建议</th></tr></thead>
  <tbody>
    <tr>
      <td>AsoClient 无重试机制</td>
      <td>网络抖动时告警丢失</td>
      <td>增加指数退避重试（3次）</td>
    </tr>
  </tbody>
</table>
```

**暗色主题颜色方案**（CSS 变量）：
```css
:root {
  --bg: #1a1b26;
  --text: #c0caf5;
  --text-light: #565f89;
  --border: #3b4261;
  --card-bg: #24283b;
  --card-header-bg: #1f2335;
  --shadow: 0 1px 3px rgba(0,0,0,0.3);

  --layer-entry: #7aa2f7;
  --layer-api: #2ac3de;
  --layer-controller: #9ece6a;
  --layer-service: #ff9e64;
  --layer-data: #bb9af7;
  --layer-model: #7dcfff;
  --layer-infra: #a9b1d6;

  --mono: 'SF Mono', Menlo, Consolas, 'Liberation Mono', monospace;
  --sans: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'PingFang SC', 'Microsoft YaHei', sans-serif;
}
```

**Tab 导航样式要求**：
- Tab 栏使用 `display: flex` 横向排列，允许换行（`flex-wrap: wrap`）
- 每个 Tab 按钮有底边框高亮效果（选中时 `border-bottom: 3px solid` 对应层级色）
- Tab 面板默认隐藏，选中时 `display: block`
- 使用极少量 JS 实现 Tab 切换（`onclick` + class 切换）

**其他样式要求**：
- 模块卡片使用暗色卡片底（`--card-bg`）、圆角 8px、轻边框（`--border`）
- 异常表格使用暗色行交替（`#24283b` / `#1f2335`）
- 函数名用高亮色（`#f7768e`）
- 调用箭头用层级色
- 代码用等宽字体
- 支持在浏览器中直接打开、选择复制文本内容

**完整模板**参考 [references/html-template.html](references/html-template.html)。生成时严格按照模板的结构和 CSS 模式，将分析数据填入对应位置。

## 输出要求

1. **必须生成独立 HTML 文件**，不要用 widget 渲染
2. 文件名：`{项目名}-architecture.html`
3. 保存到用户可访问的输出目录
4. 生成完成后使用 `present_files` 工具呈现给用户
5. 简要说明分析结果要点

## 大项目处理策略

| 项目规模 | 文件数 | 策略 |
|----------|--------|------|
| 小型 | < 30 | 逐文件完整分析 |
| 中型 | 30–100 | 聚焦核心业务文件，工具/测试文件仅统计 |
| 大型 | > 100 | 使用多个子代理并行扫描子模块，汇总后生成 |

对于大型项目：
1. 先扫描目录结构，识别子模块边界
2. 为每个子模块分配一个子代理并行分析
3. 每个子代理需额外提取：各模块的实现原理摘要（2~3 句话描述核心逻辑、数据结构、通信协议）和常见异常列表（每个模块至少 3 个典型问题，含问题描述、影响范围、建议修复方式）
4. 主代理汇总所有子代理的分析结果
5. 统一生成 HTML 架构图

## 多语言解析要点

详见 [references/analysis-guide.md](references/analysis-guide.md)，包含 Python、Java、Go 三种语言的具体解析策略和特征识别方法。
