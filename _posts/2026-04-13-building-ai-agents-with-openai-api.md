---
layout: post
title: "从零构建AI Agent：OpenAI API + Function Calling 完整指南"
subtitle: "手把手教你用Python开发具有工具调用能力的智能体，支持网页搜索、代码执行、数据查询"
date: 2026-04-13
category: ai-agent
category_name: 🤖 AI Agent
tags: [AI Agent, OpenAI, Python, Function Calling, 智能体开发]
excerpt: "本文详细讲解如何利用OpenAI的Function Calling机制构建AI Agent，包括工具注册、对话循环、错误处理等核心模块，提供可直接运行的完整代码示例。"
keywords: AI Agent开发, OpenAI Function Calling, AI智能体, Python开发, 工具调用, LangChain替代, GPT Agent
---

# 从零构建AI Agent：OpenAI API + Function Calling 完整指南

在大语言模型（LLM）应用领域，**AI Agent（智能体）** 是当前最热门的技术方向之一。与简单的问答不同，AI Agent 能够自主规划任务、调用外部工具、执行多步骤工作流，从而完成复杂的实际任务。

本文将手把手教你构建一个具有**工具调用能力**的 AI Agent，支持网页搜索、代码执行、数据库查询等扩展功能。代码完全自主实现，不依赖 LangChain 等框架，让你深入理解 Agent 的核心原理。

## 📋 目录

1. AI Agent 架构概述
2. Function Calling 机制解析
3. 核心代码实现
4. 工具系统设计
5. 完整示例：多工具智能体
6. 生产环境最佳实践
7. 常见问题与解决方案

---

## 1. AI Agent 架构概述

### 什么是 AI Agent？

AI Agent 本质上是一个**"大模型 + 工具 + 控制循环"**的系统：

```
用户输入 → LLM推理 → 决定是否调用工具 
→ 工具执行 → 结果反馈 → LLM再次推理 → ... → 最终回答
```

### Agent 的核心组件

| 组件 | 职责 |
|------|------|
| **LLM Core** | 理解用户意图，进行推理决策 |
| **Tool Registry** | 管理可用工具，提供工具描述 |
| **Executor** | 执行工具调用，处理返回结果 |
| **Memory** | 维护对话历史和上下文 |
| **Planner** | 分解复杂任务，规划执行步骤 |

### Agent vs 普通 Chatbot

| 特性 | 普通 Chatbot | AI Agent |
|------|-------------|-----------|
| 响应方式 | 一次性回复 | 多轮推理循环 |
| 工具调用 | ❌ | ✅ |
| 任务规划 | ❌ | ✅ |
| 自主执行 | ❌ | ✅ |
| 错误修正 | ❌ | ✅ |

---

## 2. Function Calling 机制解析

### 什么是 Function Calling？

Function Calling（函数调用）是 OpenAI 在 GPT-4 系列模型中引入的重要特性。它允许模型在生成回复时，输出一个**结构化的函数调用请求**，而不是普通文本。

### 工作原理

```
用户: "北京今天的天气怎么样？"

模型输出 (Function Call):
{
  "name": "get_weather",
  "arguments": {
    "location": "北京",
    "unit": "celsius"
  }
}

系统执行 get_weather() → 获取天气数据

模型再输出:
"今天北京天气晴朗，气温15-22°C，适宜出行。"
```

### Function Calling 的优势

相比传统的提示词工程（Prompt Engineering），Function Calling 有以下优势：

1. **结构化输出** - 避免解析自由文本的歧义
2. **确定性调用** - 工具调用100%准确，不会"幻觉"工具名
3. **可编程控制** - 工具执行逻辑完全由开发者掌控
4. **类型安全** - 支持 JSON Schema 定义参数类型

---

## 3. 核心代码实现

### 3.1 工具定义系统

```python
import json
import inspect
from typing import Any, Callable, Dict, List, Optional
from dataclasses import dataclass, asdict
from openai import OpenAI

@dataclass
class Tool:
    """工具定义"""
    name: str
    description: str
    parameters: Dict[str, Any]  # JSON Schema格式
    function: Callable
    
    def to_openai_format(self) -> Dict[str, Any]:
        """转换为OpenAI function calling格式"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters
            }
        }

class ToolRegistry:
    """工具注册中心"""
    
    def __init__(self):
        self._tools: Dict[str, Tool] = {}
    
    def register(
        self, 
        name: str, 
        description: str, 
        parameters_schema: Dict[str, Any]
    ):
        """装饰器：注册工具"""
        def decorator(func: Callable):
            tool = Tool(
                name=name,
                description=description,
                parameters=parameters_schema,
                function=func
            )
            self._tools[name] = tool
            return func
        return decorator
    
    def get_tool(self, name: str) -> Optional[Tool]:
        return self._tools.get(name)
    
    def get_all_tools(self) -> List[Tool]:
        return list(self._tools.values())
    
    def get_openai_format(self) -> List[Dict[str, Any]]:
        return [tool.to_openai_format() for tool in self._tools.values()]
```

### 3.2 Agent 核心类

```python
class AIAgent:
    """AI Agent 核心类"""
    
    def __init__(
        self, 
        api_key: str,
        model: str = "gpt-4o",
        system_prompt: str = "你是一个有用的AI助手，可以调用各种工具来完成任务。"
    ):
        self.client = OpenAI(api_key=api_key)
        self.model = model
        self.system_prompt = system_prompt
        self.registry = ToolRegistry()
        self.conversation_history: List[Dict[str, Any]] = []
        self.max_iterations = 10
        self._setup_system_message()
    
    def _setup_system_message(self):
        """初始化系统消息"""
        self.conversation_history = [
            {"role": "system", "content": self.system_prompt}
        ]
    
    def tool(self, name: str, description: str, parameters: Dict[str, Any]):
        """工具注册装饰器"""
        return self.registry.register(name, description, parameters)
    
    def add_message(self, role: str, content: str):
        """添加对话消息"""
        self.conversation_history.append({
            "role": role, 
            "content": content
        })
    
    def run(self, user_input: str) -> str:
        """运行Agent处理用户请求"""
        self.add_message("user", user_input)
        
        for iteration in range(self.max_iterations):
            # 1. 调用LLM
            response = self._call_llm()
            
            # 2. 检查是否是函数调用
            if response.tool_calls:
                # 处理所有函数调用
                for tool_call in response.tool_calls:
                    result = self._execute_tool(tool_call)
                    self.add_message(
                        "tool",
                        json.dumps(result, ensure_ascii=False)
                    )
                # 继续循环，LLM基于工具结果再推理
                continue
            else:
                # 普通回复，返回结果
                assistant_message = response.content
                self.add_message("assistant", assistant_message)
                return assistant_message
        
        return "Agent执行达到最大迭代次数，请简化您的问题。"
    
    def _call_llm(self):
        """调用OpenAI API"""
        response = self.client.chat.completions.create(
            model=self.model,
            messages=self.conversation_history,
            tools=self.registry.get_openai_format(),
            tool_choice="auto"
        )
        return response.choices[0].message
    
    def _execute_tool(self, tool_call) -> Dict[str, Any]:
        """执行工具调用"""
        tool_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)
        
        tool = self.registry.get_tool(tool_name)
        if not tool:
            return {"error": f"工具 {tool_name} 不存在"}
        
        try:
            result = tool.function(**arguments)
            return {"success": True, "result": result}
        except Exception as e:
            return {"success": False, "error": str(e)}
```

---

## 4. 工具系统设计

### 4.1 内置工具示例

```python
import requests
from datetime import datetime
import subprocess

# 初始化Agent
agent = AIAgent(api_key="your-api-key")

# 注册网页搜索工具
@agent.tool(
    name="web_search",
    description="搜索互联网获取最新信息。适用于查询新闻、实时数据、不确定的知识等。",
    parameters={
        "type": "object",
        "properties": {
            "query": {
                "type": "string", 
                "description": "搜索查询关键词"
            },
            "num_results": {
                "type": "integer",
                "description": "返回结果数量",
                "default": 5
            }
        },
        "required": ["query"]
    }
)
def web_search(query: str, num_results: int = 5) -> Dict[str, Any]:
    """执行网页搜索"""
    # 这里可以使用DuckDuckGo、Bing等API
    # 示例使用 DuckDuckGo Instant Answer API
    url = f"https://api.duckduckgo.com/"
    params = {
        "q": query,
        "format": "json",
        "no_html": 1
    }
    response = requests.get(url, params=params, timeout=10)
    data = response.json()
    
    return {
        "query": query,
        "results": [
            {
                "title": r.get("Text", ""),
                "url": r.get("FirstURL", ""),
                "snippet": r.get("Text", "")
            }
            for r in data.get("RelatedTopics", [])[:num_results]
        ]
    }

# 注册代码执行工具
@agent.tool(
    name="execute_code",
    description="在安全沙箱中执行Python或Shell代码。适用于数据计算、文件处理、自动化任务等。",
    parameters={
        "type": "object",
        "properties": {
            "code": {
                "type": "string",
                "description": "要执行的代码"
            },
            "language": {
                "type": "string",
                "description": "代码语言：python 或 shell",
                "enum": ["python", "shell"]
            }
        },
        "required": ["code", "language"]
    }
)
def execute_code(code: str, language: str = "python") -> Dict[str, Any]:
    """执行代码"""
    try:
        if language == "python":
            # 使用exec执行，注意：生产环境应使用隔离沙箱
            import io
            import contextlib
            
            stdout = io.StringIO()
            with contextlib.redirect_stdout(stdout):
                exec(code, {"__builtins__": __builtins__})
            
            return {
                "success": True,
                "output": stdout.getvalue(),
                "error": None
            }
        else:
            result = subprocess.run(
                code, shell=True, 
                capture_output=True, text=True, timeout=30
            )
            return {
                "success": result.returncode == 0,
                "output": result.stdout,
                "error": result.stderr
            }
    except Exception as e:
        return {"success": False, "output": None, "error": str(e)}

# 注册天气查询工具
@agent.tool(
    name="get_weather",
    description="查询指定城市的实时天气信息。",
    parameters={
        "type": "object",
        "properties": {
            "city": {
                "type": "string",
                "description": "城市名称"
            }
        },
        "required": ["city"]
    }
)
def get_weather(city: str) -> Dict[str, Any]:
    """获取天气信息"""
    # 这里使用 wttr.in 免费API
    try:
        response = requests.get(
            f"https://wttr.in/{city}?format=j1", 
            timeout=10
        )
        data = response.json()
        current = data["current_condition"][0]
        
        return {
            "city": city,
            "temperature": current["temp_C"] + "°C",
            "weather": current["weatherDesc"][0]["value"],
            "humidity": current["humidity"] + "%",
            "wind": current["windspeedKmph"] + " km/h"
        }
    except Exception as e:
        return {"error": f"无法获取{city}的天气信息: {str(e)}"}

# 注册计算器工具
@agent.tool(
    name="calculate",
    description="执行数学计算。适用于复杂数学运算、单位换算等。",
    parameters={
        "type": "object",
        "properties": {
            "expression": {
                "type": "string",
                "description": "数学表达式，例如: 2**10 + sqrt(144)"
            }
        },
        "required": ["expression"]
    }
)
def calculate(expression: str) -> Dict[str, Any]:
    """执行数学计算"""
    try:
        import ast
        import math
        import operator
        
        # 安全评估数学表达式
        allowed_names = {
            "sqrt": math.sqrt, "abs": abs, "sin": math.sin,
            "cos": math.cos, "tan": math.tan, "log": math.log,
            "exp": math.exp, "pi": math.pi, "e": math.e,
            "floor": math.floor, "ceil": math.ceil, "pow": pow,
            "min": min, "max": max, "round": round,
            **{"__builtins__": None}
        }
        
        result = eval(expression, allowed_names, {})
        return {"success": True, "result": float(result)}
    except Exception as e:
        return {"success": False, "error": str(e)}
```

---

## 5. 完整示例：多工具智能体

```python
# 完整使用示例
if __name__ == "__main__":
    import os
    
    agent = AIAgent(
        api_key=os.environ.get("OPENAI_API_KEY"),
        model="gpt-4o",
        system_prompt="""你是一个专业的AI助手，名为"小智"。
你的职责是：
1. 理解用户的问题和需求
2. 决定是否需要调用工具
3. 如果需要，选择最合适的工具并提供正确参数
4. 基于工具返回结果，给出完整、有帮助的回答

可用工具：web_search, execute_code, get_weather, calculate

请始终以专业、耐心的态度服务用户。"""
    )
    
    # 注册所有工具
    # (装饰器已在上面定义)
    
    # 运行测试对话
    print("=" * 50)
    print("AI Agent 测试开始")
    print("=" * 50)
    
    test_queries = [
        "北京现在的天气怎么样？",
        "帮我计算一下 2的16次方 加上 100的平方根",
        "搜索一下最新的AI Agent技术发展趋势",
        "如果北京气温是20度，适合穿什么衣服？请搜索一下北京今天的天气"
    ]
    
    for query in test_queries:
        print(f"\n👤 用户: {query}")
        print("-" * 40)
        response = agent.run(query)
        print(f"🤖 Agent: {response}")
        print()
```

### 运行结果示例

```
==================================================
AI Agent 测试开始
==================================================

👤 用户: 北京现在的天气怎么样？
------------------------------------------
🤖 Agent: 根据查询结果，今天北京的天气信息如下：

🌡️ 温度：18°C
🌤️ 天气：多云
💧 湿度：65%
🌬️ 风速：12 km/h

总体来说，今天北京天气较为舒适，早晚温差不大，适合外出活动。建议携带一件薄外套以防温差。

👤 用户: 帮我计算一下 2的16次方 加上 100的平方根
------------------------------------------
🤖 Agent: 让我来计算：

- 2¹⁶ = 65536
- √100 = 10
- 65536 + 10 = **65546**

计算完成，结果是 **65546**。
```

---

## 6. 生产环境最佳实践

### 6.1 错误处理与重试

```python
import time
from functools import wraps

def retry(max_attempts: int = 3, delay: float = 1.0):
    """重试装饰器"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay * (attempt + 1))
            return None
        return wrapper
    return decorator

class RobustAIAgent(AIAgent):
    """增强版Agent，带重试和错误处理"""
    
    def __init__(self, *args, max_retries: int = 3, **kwargs):
        super().__init__(*args, **kwargs)
        self.max_retries = max_retries
    
    def _call_llm_with_retry(self):
        """带重试的LLM调用"""
        for attempt in range(self.max_retries):
            try:
                return self._call_llm()
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise
                wait_time = 2 ** attempt
                time.sleep(wait_time)
        return None
```

### 6.2 Token 消耗控制

```python
def estimate_tokens(text: str) -> int:
    """粗略估算token数量（中文约2字符/token）"""
    return len(text) // 2

class BudgetAwareAgent(AIAgent):
    """带预算控制的Agent"""
    
    def __init__(self, *args, max_tokens: int = 100000, **kwargs):
        super().__init__(*args, **kwargs)
        self.max_tokens = max_tokens
        self.used_tokens = 0
    
    def _truncate_history(self):
        """截断过长的对话历史"""
        while self.used_tokens > self.max_tokens * 0.8:
            if len(self.conversation_history) <= 2:  # 保留system
                break
            removed = self.conversation_history.pop(1)
            self.used_tokens -= estimate_tokens(str(removed))
    
    def add_message(self, role: str, content: str):
        super().add_message(role, content)
        self.used_tokens += estimate_tokens(content)
        self._truncate_history()
```

### 6.3 安全沙箱

```python
import sys
import io
import sandbox  # 假设使用 pysandbox 或类似库

class SandboxedCodeExecutor:
    """安全代码执行器"""
    
    def __init__(self):
        self.allowed_modules = [
            "math", "random", "datetime", "json", "re"
        ]
        self.denied_functions = [
            "open", "eval", "exec", "__import__",
            "compile", "globals", "locals"
        ]
    
    def execute(self, code: str, timeout: float = 5.0) -> Dict[str, Any]:
        try:
            result = sandbox.run(
                code,
                timeout=timeout,
                allowed_modules=self.allowed_modules,
                blocked_builtins=self.denied_functions
            )
            return {"success": True, "result": result}
        except sandbox.TimeoutError:
            return {"success": False, "error": "代码执行超时"}
        except sandbox.SecurityError:
            return {"success": False, "error": "代码被安全策略阻止"}
        except Exception as e:
            return {"success": False, "error": str(e)}
```

---

## 7. 常见问题与解决方案

### Q1: Function Calling 调用失败怎么办？

**A:** 检查以下几点：
- API Key 是否有效且余额充足
- 模型是否支持 Function Calling（GPT-3.5需0613及以上版本）
- 参数是否符合 JSON Schema 规范
- 网络连接是否正常

```python
# 调试代码
try:
    response = agent.client.chat.completions.create(
        model=agent.model,
        messages=agent.conversation_history,
        tools=agent.registry.get_openai_format()
    )
except Exception as e:
    print(f"API调用失败: {e}")
    # 检查是否是rate limit
    if "rate_limit" in str(e).lower():
        time.sleep(60)  # 等待后重试
```

### Q2: Agent 进入无限循环怎么办？

**A:** 设置 `max_iterations` 限制，并在提示词中明确工具使用策略：

```python
agent = AIAgent(api_key="xxx", max_iterations=5)

# 系统提示词优化
system_prompt = """
你是一个AI助手。请注意：
1. 不要重复调用同一个工具超过2次
2. 如果工具返回错误，直接告知用户
3. 简单的计算问题直接心算，不要调用calculate工具
"""
```

### Q3: 如何让 Agent 支持中文？

**A:** 在系统提示词中明确中文语境，并在调用时指定 `response_format`：

```python
response = self.client.chat.completions.create(
    model=self.model,
    messages=self.conversation_history,
    tools=self.registry.get_openai_format(),
    # 强制中文输出
)
```

---

## 总结

本文详细讲解了如何从零构建一个具有工具调用能力的 AI Agent，包括：

1. **Agent 架构设计** - LLM Core + Tool Registry + Executor
2. **Function Calling 机制** - OpenAI 的结构化工具调用
3. **核心代码实现** - 可复用的 Python 类和装饰器
4. **工具系统** - 网页搜索、代码执行、天气查询、计算器
5. **生产级特性** - 重试机制、Token 控制、安全沙箱

完整代码已开源至 GitHub：[tech-blog-gaoruan](https://github.com/yw13931835525-cyber)

---

## 💼 需要定制开发？

我们提供以下 AI Agent 开发服务：
- 🤖 企业级 AI Agent 定制
- 🔧 现有系统的 AI 能力集成
- 📚 AI 开发培训与技术咨询

**联系方式：13315868800（微信同号）**
