# Spring-AI-Alibaba 源码阅读笔记

## 模块：spring-ai-alibaba-agent-framework

---

## 🌟 核心特性

- ReAct 智能体实现  
- 多智能体工作流编排（Graph）  
- 上下文工程 & 记忆管理  
- 人机协作（Human-in-the-loop）  
- Agent-to-Agent（A2A）通信  
- 模型、工具、MCP（Multi-Channel Provider）生态集成  

<span style="color:#8b8b8b; font-size:0.85em;">
↳ ⌨️ 快捷键提示：Ctrl + H 查看继承层次结构 | Ctrl + Alt + H 查看调用关系
</span>

---

## 📌 Agent 核心类解析

### 📦 `com.alibab.cloud.ai.graph.agent.Agent`

![Agent 继承关系图](./image/AgentHierarchy.png)

### 🧠 核心字段说明

| 类 / 字段                                        | 作用说明                                                         |
| ------------------------------------------------ | ---------------------------------------------------------------- |
| **`Agent`（抽象类）**                            | 所有智能体的根基类，定义身份、能力、生命周期                     |
| `protected String name`                          | Agent 唯一标识，在 Graph 中用于路由                              |
| `protected String description`                   | 大模型根据任务内容与 description 判断是否由该 Agent 执行         |
| `protected CompileConfig compileConfig`          | Graph 编译阶段全局配置：断点保存、缓存、中断点、超时、循环检测等 |
| `protected volatile CompiledGraph compiledGraph` | `AgentGraph.compile()` 后生成的可执行模板（类似 `.class` 文件）  |
| `protected volatile StateGraph graph`            | 单次请求时 new 出的「运行实例」，包含本次的记忆、变量、状态等    |
| `protected Executor executor`                    | 异步 / 并行任务线程池（工具调用、并行 Agent 等都在其中执行）     |

<span style="color:#8b8b8b; font-size:0.85em;">
↳ 记忆点：AgentGraph（设计图） → CompiledGraph（.class） → StateGraph（运行中的实例）
</span>

---

## 🔍 核心方法说明

### `public StateGraph getGraph()`
初始化 `StateGraph`，内部调用子类实现的：

```java
protected abstract StateGraph initGraph() throws GraphStateException;
```
```java
Agent.invoke()  Agent抽象类调用代理方法
     ↓
compiledGraph.invoke()
     ↓
streamFromInitialNode()     ── 创建 Flux 流 & 初始化 State
     ↓
GraphRunner.run()           ── 返回 Flux.defer()（订阅时才执行）
     ↓
MainGraphExecutor.execute() ── 判断节点执行条件：
        • 最大迭代次数是否达到
        • 是否需要停止
        • 是否需要恢复
        • 是否有外部打断（Human / System）
        • 是否是开始节点
        ↓
nodeExecutor.execute()      ── 子节点执行器
     • AsyncParallelNodeAction
     • SubCompiledGraphNodeAction
     • HumanInTheLoopHook
     ↓
processParallelResults()    ── 并行结果合并（优先级）
        GraphFlux > Flux > 普通对象


Agent.stream() 和上面流程一样 但是最后返回的Flux流对象
```
<!-- 
Agent.buildNonStreamConfig() 构建非流配置 -> builder.addMetadata("_stream_", false) 配置stream为false
Agent.buildStreamConfig() 构建流配置
Agent.applyExecutorConfig() 将执行器应用配置到RunnableConfig构建器

### Agent子类BaseAgent
protected String inputSchema;  输入格式
protected Type inputType;      输入类型

protected String outputSchema; 输出格式
protected Class<?> outputType; 输出类型

protected String outputKey; Agent输出结果的Key

protected KeyStrategy outputKeyStrategy; Key的输出策略 REPLACE 替换  APPEND 追加  MERGE 合并 支持自定义

protected boolean includeContents; 是否把Agent的结果暴露给后续的Agent或者用户（是否把结果假如到上下文）

protected boolean returnReasoningContents; 是否把Agent推理思考的内容也返回 -->


#### 1. `Agent.buildNonStreamConfig()`
构建 **非流式** 配置  
- 调用：`builder.addMetadata("_stream_", false)`  
- 作用：设置 `_stream_ = false`，表示关闭流式输出

#### 2. `Agent.buildStreamConfig()`
构建 **流式（Stream）** 配置  
- 设置 `_stream_ = true`  
- 用于逐 Token 输出结果

#### 3. `Agent.applyExecutorConfig()`
将执行器（Executor）相关配置应用到 `RunnableConfig` 构建器  
- 包含：模型参数、温度、top-k/p、最大 token、工具调用设置等

---

### BaseAgent 字段说明

#### 输入相关字段
- `protected String inputSchema;`  
  - 输入格式（JSON Schema）

- `protected Type inputType;`  
  - 输入类型（Java 类型，用于反序列化）

---

#### 输出相关字段
- `protected String outputSchema;`  
  - 输出格式（Schema）

- `protected Class<?> outputType;`  
  - 输出类型（反序列化用）

- `protected String outputKey;`  
  - Agent 输出结果在上下文中的存放 Key

---

#### 输出 Key 策略
- `protected KeyStrategy outputKeyStrategy;`  
  - 输出内容写入上下文的方式  
  - `REPLACE`：替换原值  
  - `APPEND`：追加内容  
  - `MERGE`：合并（Map/JSON）  
  - 支持自定义策略

---

#### 执行内容控制
- `protected boolean includeContents;`  
  - 是否将 Agent 结果写入上下文（暴露给后续 Agent 或用户）

- `protected boolean returnReasoningContents;`  
  - 是否返回 Agent 的推理过程（ReAct / 思考链路）
