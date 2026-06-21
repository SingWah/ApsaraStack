# 多语言代码解析指南

## Python 解析策略

### 入口识别

| 特征 | 说明 |
|------|------|
| `if __name__ == '__main__'` | 脚本入口 |
| `setup.py` 中的 `entry_points` | 安装后的命令行入口 |
| `def main()` | 常见主函数 |
| `app.run()` | Web 应用入口 |
| `argparse` / `click` / `optparse` | CLI 框架入口 |

### 函数/方法提取

- `def function_name(params):` → 普通函数
- `def method_name(self, params):` → 实例方法
- `@classmethod def name(cls):` → 类方法
- `@staticmethod def name():` → 静态方法
- `@property def name(self):` → 属性访问器

### 类关系

- `class Child(Parent):` → 继承关系
- `class Child(Parent, Mixin):` → 多继承
- `self.other = OtherClass()` → 组合关系（在 `__init__` 中）
- `isinstance(obj, SomeClass)` → 类型依赖

### API 接口识别

- `@app.route('/path', methods=['GET', 'POST'])`
- `@blueprint.route('/path')`
- `url_patterns = [...]` (Django)
- `class MyHandler(BaseHTTPRequestHandler):`
- 搜索 `add_url_rule`, `url_map`, `wsgi_app`

### Import 依赖分析

```python
import module_name           # 标准库 / 第三方 / 项目内部
from package import name     # 需要判断来源
from . import sibling        # 相对导入，项目内部
from ..parent import name    # 上级包导入
```

判断标准库 vs 第三方 vs 项目内部：
- 以 `.` 开头 → 项目内部
- 在 `sys.builtin_module_names` 中 → 标准库
- 在 `setup.py` 的 `install_requires` 中 → 第三方
- 其余 → 需要结合上下文判断

---

## Java 解析策略

### 入口识别

| 特征 | 说明 |
|------|------|
| `public static void main(String[] args)` | 标准入口 |
| `@SpringBootApplication` | Spring Boot 入口 |
| `extends HttpServlet` | Servlet 入口 |

### 函数/方法提取

- `public/private/protected [static] ReturnType methodName(params)` 
- 注解辅助：`@Override`, `@Transactional`, `@Async`

### 类关系

- `class Child extends Parent` → 继承
- `class Child implements Interface1, Interface2` → 实现接口
- `@Autowired private Service svc;` → 依赖注入
- `new ClassName()` → 直接实例化

### API 接口识别

- `@RequestMapping("/path")`
- `@GetMapping("/path")`, `@PostMapping("/path")`
- `@PutMapping`, `@DeleteMapping`, `@PatchMapping`
- `@RestController` 标记的类中所有 public 方法
- `@Controller` + `@ResponseBody`

### 包结构惯例

```
src/main/java/com/company/project/
├── controller/   → 控制层
├── service/      → 服务层（接口 + impl）
├── dao/          → 数据层
├── model/        → 模型层
├── config/       → 配置
└── util/         → 工具
```

### Import 依赖分析

- `import java.*` / `import javax.*` → 标准库
- `import org.springframework.*` → Spring 框架
- `import com.company.project.*` → 项目内部
- 其他 → 第三方库

---

## Go 解析策略

### 入口识别

| 特征 | 说明 |
|------|------|
| `func main()` in `main` package | 标准入口 |
| `func init()` | 包初始化 |

### 函数/方法提取

- `func FunctionName(params) returnType` → 普通函数
- `func (r *Receiver) MethodName(params) returnType` → 方法（关联到 Receiver 类型）
- `func (r Receiver) MethodName(params) returnType` → 值接收者方法

### 类型/结构体关系

- `type Child struct { Parent }` → 嵌入（类似继承）
- `type MyInterface interface { ... }` → 接口定义
- 隐式实现：结构体实现了接口的所有方法即为实现该接口

### API 接口识别

- `http.HandleFunc("/path", handler)`
- `mux.HandleFunc("/path", handler)`
- `r.GET("/path", handler)` (Gin)
- `r.POST("/path", handler)` (Gin)
- 自定义 `ServeHTTP(w, r)` 方法

### Import 依赖分析

```go
import (
    "fmt"                        // 标准库
    "github.com/gin-gonic/gin"   // 第三方
    "myproject/internal/handler" // 项目内部
)
```

判断规则：
- 无域名前缀（如 `fmt`, `net/http`）→ 标准库
- 有域名前缀且非本项目 module path → 第三方
- 匹配 `go.mod` 中的 module path → 项目内部

---

## Shell 脚本解析策略

Shell 脚本通常不需要深度分析，但以下信息有价值：

- `source` / `.` 命令 → 脚本依赖
- 函数定义 `function_name() { ... }` → 函数
- 外部命令调用 → 外部依赖
- 环境变量引用 → 配置依赖

---

## 通用分析技巧

### 目录结构推断层级

大多数项目遵循一定的目录命名惯例：

| 目录名关键词 | 推断层级 |
|-------------|---------|
| `cmd/`, `bin/`, `main/`, `entry/` | 入口层 |
| `api/`, `routes/`, `handlers/` | 接口层 |
| `controller/`, `action/` | 控制层 |
| `service/`, `biz/`, `logic/`, `engine/` | 服务层 |
| `dao/`, `repository/`, `store/`, `db/` | 数据层 |
| `model/`, `entity/`, `domain/`, `schema/` | 模型层 |
| `util/`, `helper/`, `common/`, `lib/`, `middleware/` | 基础设施层 |

### 调用关系追踪

1. **函数调用**: 搜索 `函数名(` 模式，结合 import 判断是否跨模块
2. **方法调用**: 搜索 `对象.方法名(` 或 `self.方法名(`
3. **HTTP 调用**: 搜索 `requests.`, `http.`, `HttpClient`, `urllib`
4. **SQL 执行**: 搜索 `execute(`, `query(`, `cursor.`, `session.`
5. **消息发送**: 搜索 `publish(`, `send(`, `produce(`

### 大文件处理

对于超过 500 行的文件：
1. 先读前 50 行，获取 import 和类定义概览
2. 使用 Grep 搜索关键模式（def, class, @route 等）
3. 只读关键函数的实现代码
4. 避免一次性读取全部内容

---

## 实现细节与异常模式提取指南

本章节指导如何从源码中提取模块实现原理和识别常见异常/陷阱，这些信息将填入 HTML 架构图的模块详情区块。

### 提取实现原理

对每个模块，用 2~4 句话概括其实现原理，关注以下维度：

| 维度 | 提取方法 | 示例 |
|------|---------|------|
| 核心算法/逻辑 | 阅读主函数的执行流程，理解 if/else 分支和循环结构 | "使用优先级队列调度任务，按紧急程度排序执行" |
| 数据结构 | 关注类属性初始化、全局变量、缓存容器 | "使用 dict 作为 2 小时 pickle 缓存的内存索引" |
| 并发模型 | 搜索 Thread、Process、Pool、async、goroutine 关键词 | "4 个工作线程 + Timer 超时机制（300s）" |
| 通信协议 | 搜索 HTTP client、socket、MQ producer 等 | "通过 HTTP POST 发送告警到天机 API" |
| 缓存/持久化 | 搜索 pickle、json.dump、redis、db.write 等 | "每 2 小时将检测结果 pickle 序列化到本地文件" |

**提取技巧**：
- 优先阅读 `__init__` / `__new__` / 构造函数，了解核心数据结构
- 阅读 `run` / `start` / `main` / `execute` 等主函数，了解执行流程
- 关注模块级全局变量和常量定义
- 关注 import 的第三方库（暗示通信方式和数据格式）

### 识别异常处理模式

扫描代码中的异常处理，识别以下模式：

**Python**：
```python
# 模式 1：裸 except（吞异常）— 标记为陷阱
try:
    do_something()
except:       # ← 无异常类型，吞掉所有异常
    pass

# 模式 2：空 except body — 标记为陷阱
try:
    result = api_call()
except Exception as e:
    pass        # ← 捕获了但没处理

# 模式 3：仅在日志中记录但不上报 — 标记为陷阱
try:
    send_alert(msg)
except Exception as e:
    logging.error(e)  # ← 仅记日志，告警可能丢失
```

**Java**：
```java
// 模式 1：空 catch — 标记为陷阱
try { ... } catch (Exception e) { }

// 模式 2：吞 InterruptedException — 标记为陷阱
try { Thread.sleep(1000); } catch (InterruptedException e) { }

// 模式 3：资源未关闭 — 标记为陷阱
Connection conn = getConnection();  // ← 没有 try-with-resources
conn.query(sql);
// conn.close() 可能因异常跳过
```

**Go**：
```go
// 模式 1：忽略 error — 标记为陷阱
result, _ := doSomething()  // ← 忽略 error

// 模式 2：defer 关闭顺序错误 — 标记为陷阱
defer rows.Close()  // ← 如果 rows 为 nil 则 panic
```

### 常见陷阱扫描清单

以下搜索模式可用于快速定位潜在问题：

| 搜索模式（grep/正则） | 对应的陷阱 | 语言 |
|---|---|---|
| `except:` 或 `except Exception` + `pass` | 吞异常 | Python |
| `os.system\(` 或 `commands.getstatusoutput` | 命令注入风险 | Python |
| `subprocess.Popen.*shell=True` | 命令注入风险 | Python |
| `requests\.(get\|post\|put\|delete)\(` 无 retry | 无重试网络调用 | Python |
| `open\(` 无 `with` 上下文 | 文件句柄泄漏 | Python |
| `pickle.load\(` | 反序列化安全风险 | Python |
| `catch (Exception` + 空 body | 吞异常 | Java |
| `catch (InterruptedException` + 空 body | 中断处理不当 | Java |
| `new FileInputStream` 无 try-with-resources | 资源泄漏 | Java |
| `_, _ := ` 或 `err` 变量未检查 | 忽略错误 | Go |
| `defer .*\.Close()` 未检查 nil | 潜在 panic | Go |
| `time.Sleep` 在循环中无 context 取消 | goroutine 泄漏 | Go |
| 硬编码 IP 地址（`\d+\.\d+\.\d+\.\d+`） | 部署灵活性差 | 通用 |
| 硬编码密码/密钥（`password =`, `secret =`） | 安全风险 | 通用 |
| `timeout` 缺失的 HTTP/socket 调用 | 请求永久阻塞 | 通用 |

### 异常记录格式

每条异常/陷阱使用以下三列格式记录：

| 列 | 内容 | 示例 |
|----|------|------|
| 问题 | 简明描述具体问题是什么 | "AsoClient.send 无重试机制，单次 HTTP POST 失败即丢弃" |
| 影响 | 该问题会导致什么后果 | "网络抖动时告警消息丢失，运维无法及时收到异常通知" |
| 建议 | 推荐的修复方式 | "增加指数退避重试（最多 3 次，间隔 1s/2s/4s）" |

每个模块至少提取 3 条典型问题。对于核心业务模块（如状态机、调度器、核心检查脚本），建议提取 5~7 条。
