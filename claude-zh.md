# CLAUDE-zh

本文件是 `CLAUDE.md` 的中文版，作为在此仓库中工作的指导说明。

## 这个项目是什么

IDA Pro MCP Server：向 MCP 客户端暴露 IDA Pro / idalib 功能。

主要组成：
- `src/ida_pro_mcp/server.py`：MCP 服务入口
- `src/ida_pro_mcp/idalib_server.py`：无头 idalib 服务
- `src/ida_pro_mcp/ida_mcp/`：IDA / 插件侧 API

重要 API 模块：
- `api_core.py`：IDB 元数据、函数、字符串、导入
- `api_analysis.py`：反编译、反汇编、交叉引用、路径、模式搜索
- `api_memory.py`：字节/整数/字符串读写、补丁
- `api_types.py`：结构体、类型推断、类型应用
- `api_modify.py`：注释、重命名、汇编补丁
- `api_stack.py`：栈帧操作
- `api_debug.py`：调试器控制（测试中不安全/低优先级）
- `api_python.py`：在 IDA 上下文执行 Python
- `api_resources.py`：`ida://` MCP 资源

## 核心实现规则

### IDA 线程安全

所有 IDA SDK 调用都必须在主线程执行。
使用：

```python
from .rpc import tool
from .sync import idasync

@tool
@idasync
def my_tool(...):
    ...
```

### API 约定

- 优先采用批处理优先（batch-first）的 API 设计。
- 很多函数接受逗号分隔字符串或列表。
- 使用完整类型标注与 `Annotated[...]` 描述。
- 函数 docstring 会成为 MCP 工具描述。

示例：

```python
def my_api(addrs: Annotated[str, "Addresses (0x401000, main) or list"]) -> list[dict]:
    ...
```

### 常用辅助函数

- 用 `parse_address()` 解析地址
- 用 `normalize_list_input()` / `normalize_dict_list()` 归一化批量输入
- 使用 `utils.py` 中共享的分页/过滤辅助函数

### 不安全操作

调试器或破坏性操作应标记为 unsafe：

```python
from .rpc import tool, unsafe

@unsafe
@tool
@idasync
def dangerous_op(...):
    ...
```

## 开发命令

### 运行

```bash
uv run ida-pro-mcp
uv run ida-pro-mcp --transport http://127.0.0.1:8744/sse
uv run idalib-mcp --host 127.0.0.1 --port 8745 path/to/binary
uv run idalib-mcp --isolated-contexts --host 127.0.0.1 --port 8745 path/to/binary
uv run ida-pro-mcp --unsafe
```

### MCP inspector

```bash
uv run mcp dev src/ida_pro_mcp/server.py
```

### 安装 / 卸载

```bash
uv run ida-pro-mcp --install
uv run ida-pro-mcp --uninstall
```

## 测试与覆盖率

### 运行测试

使用无头测试运行器：

```bash
uv run ida-mcp-test tests/crackme03.elf -q
uv run ida-mcp-test tests/typed_fixture.elf -q
uv run ida-mcp-test tests/crackme03.elf -c api_analysis
uv run ida-mcp-test tests/typed_fixture.elf -p "*stack*"
```

说明：
- 使用 `uv run ...`
- 非交互输出应仅显示失败与汇总
- 二进制相关测试应使用 `@test(binary="...")`，并填写可执行文件 basename

### 覆盖率

在两套维护中的 fixture 上测量覆盖率：

```bash
uv run coverage erase
uv run coverage run -m ida_pro_mcp.test tests/crackme03.elf -q
uv run coverage run --append -m ida_pro_mcp.test tests/typed_fixture.elf -q
uv run coverage report --show-missing
```

当前 fixture 目标：
- `tests/crackme03.elf`：紧凑型通用回归 fixture
- `tests/typed_fixture.elf`：覆盖 typed globals / structs / locals / stack 的 fixture

### 测试期望

- 优先语义断言，避免弱化为“字段存在”检查
- 对可变更 API，优先 round-trip 测试
- 若测试暴露 API 行为明显错误，应修 API，而非削弱测试
- 聚焦 IDA 侧模块，而非 server/config 管线细节
- 对 IDA / Hex-Rays 差异可使用守卫断言或运行时跳过（需有合理理由）

### 通用测试健全性检查

新增通用测试时，也在非 fixture 二进制上试跑，避免 ELF 特化假设：

```bash
uv run ida-mcp-test "C:\CodeBlocks\x64dbg\bin\x64\x64dbg.dll" -q
```

## 范围优先级

高优先级：
- `api_analysis.py`
- `api_types.py`
- `api_modify.py`
- `api_stack.py`
- `api_memory.py`
- `api_core.py`
- `api_resources.py`
- `utils.py`
- `framework.py`

低优先级：
- `api_debug.py`
- MCP 传输 / 托管细节
- 安装 / 配置修改逻辑

## 实用备注

- Server / 插件 Python：3.11+
- IDA Pro 8.3+；推荐 9.0
- 不支持 IDA Free
- 若 IDA 绑定了错误 Python，请使用 `idapyswitch`
