---
layout: post
title: "Multi-Agent系统架构：让多个AI智能体协同工作"
subtitle: "深入讲解Multi-Agent架构设计、Agent间通信协议、任务分解与协调，提供完整的多智能体协作系统实现代码"
date: 2026-04-13
category: ai-agent
category_name: 🤖 AI Agent
tags: [Multi-Agent, AI Agent架构, 智能体协作, LangGraph, CrewAI, AutoGen, Agent通信]
excerpt: "本文深入讲解Multi-Agent（多智能体）系统的架构设计与实现，包括Agent通信协议、任务分解、协作决策等核心概念，提供基于LangGraph和CrewAI的完整实现代码。"
keywords: Multi-Agent系统, AI Agent架构, 智能体协作, LangGraph, CrewAI, Agent设计模式, 分布式AI
---

# Multi-Agent系统架构：让多个AI智能体协同工作

当单Agent能力有限时，Multi-Agent系统通过多个AI Agent的协作，可以完成更复杂的任务。本文将深入讲解Multi-Agent的架构设计与实现。

## Multi-Agent 架构模式

### 1. Supervisor模式

一个主管Agent协调多个专业Agent：

```
用户请求 → Supervisor Agent → 分解任务
                                ↓
                    ┌──────────┴──────────┐
                    ↓         ↓          ↓
               Researcher  Writer    Coder
                    ↓         ↓          ↓
                    └──────────┴──────────┘
                                ↓
                         Supervisor汇总
```

### 2. 完整实现

```python
from typing import List, Dict, Any, Optional, Callable
from dataclasses import dataclass, field
from enum import Enum
import json
import asyncio

class AgentRole(Enum):
    SUPERVISOR = "supervisor"
    RESEARCHER = "researcher"
    WRITER = "writer"
    CODER = "coder"
    REVIEWER = "reviewer"

@dataclass
class Message:
    """Agent间消息"""
    sender: str
    receiver: str  # "*" 表示广播
    content: Any
    msg_type: str  # "task", "result", "status", "error"
    metadata: Dict = field(default_factory=dict)

class BaseAgent:
    """Agent基类"""
    
    def __init__(self, name: str, role: AgentRole, model: str = "gpt-4o"):
        self.name = name
        self.role = role
        self.model = model
        self.message_queue: asyncio.Queue = asyncio.Queue()
        self.tools: Dict[str, Callable] = {}
    
    async def receive(self, message: Message):
        """接收消息"""
        await self.message_queue.put(message)
    
    async def process_message(self, message: Message) -> Any:
        """处理消息 - 子类实现"""
        raise NotImplementedError
    
    async def run(self):
        """Agent主循环"""
        while True:
            message = await self.message_queue.get()
            if message.msg_type == "shutdown":
                break
            result = await self.process_message(message)
            if result:
                yield result

class SupervisorAgent(BaseAgent):
    """主管Agent - 负责任务分解与协调"""
    
    def __init__(self, name: str = "Supervisor"):
        super().__init__(name, AgentRole.SUPERVISOR)
        self.sub_agents: Dict[str, BaseAgent] = {}
    
    def register_agent(self, agent: BaseAgent):
        self.sub_agents[agent.name] = agent
    
    async def decompose_task(self, task: str) -> List[Dict[str, Any]]:
        """将任务分解为子任务"""
        # 使用LLM进行任务分解
        decomposition_prompt = f"""
        将以下任务分解为可并行执行的子任务：
        任务: {task}
        
        返回JSON格式的子任务列表，每个子任务包含:
        - id: 唯一标识
        - description: 任务描述
        - required_capability: 所需能力 (research/writing/coding/review)
        - dependencies: 依赖的其他子任务ID列表
        """
        
        # 实际使用时调用LLM API
        # 这里返回示例结构
        return [
            {
                "id": "task_1",
                "description": "研究主题背景和相关技术",
                "required_capability": "research",
                "dependencies": []
            },
            {
                "id": "task_2", 
                "description": "撰写技术文档",
                "required_capability": "writing",
                "dependencies": ["task_1"]
            },
            {
                "id": "task_3",
                "description": "编写示例代码",
                "required_capability": "coding",
                "dependencies": ["task_1"]
            },
            {
                "id": "task_4",
                "description": "审核并优化",
                "required_capability": "review",
                "dependencies": ["task_2", "task_3"]
            }
        ]
    
    async def coordinate(
        self, 
        tasks: List[Dict], 
        message_bus: 'MessageBus'
    ) -> Dict[str, Any]:
        """协调子任务执行"""
        results = {}
        completed = set()
        
        while len(completed) < len(tasks):
            # 找到可执行的任务（依赖都已完成）
            for task in tasks:
                task_id = task["id"]
                if task_id in completed:
                    continue
                
                deps = task.get("dependencies", [])
                if all(d in completed for d in deps):
                    # 分发任务给对应Agent
                    agent_name = self._get_agent_for_capability(task["required_capability"])
                    
                    await message_bus.send(Message(
                        sender=self.name,
                        receiver=agent_name,
                        content={"task_id": task_id, "description": task["description"]},
                        msg_type="task",
                        metadata={"parent_task": task.get("parent")}
                    ))
                    completed.add(task_id)
        
        # 收集所有结果
        return results
    
    def _get_agent_for_capability(self, capability: str) -> str:
        capability_map = {
            "research": "Researcher",
            "writing": "Writer", 
            "coding": "Coder",
            "review": "Reviewer"
        }
        return capability_map.get(capability, "Writer")


class MessageBus:
    """Agent间消息总线"""
    
    def __init__(self):
        self.agents: Dict[str, BaseAgent] = {}
    
    def register(self, agent: BaseAgent):
        self.agents[agent.name] = agent
    
    async def send(self, message: Message):
        if message.receiver == "*":
            # 广播
            for agent in self.agents.values():
                if agent.name != message.sender:
                    await agent.receive(message)
        else:
            # 单播
            if message.receiver in self.agents:
                await self.agents[message.receiver].receive(message)
    
    async def broadcast_status(self, sender: str, status: Dict):
        await self.send(Message(
            sender=sender,
            receiver="*",
            content=status,
            msg_type="status"
        ))


class MultiAgentOrchestrator:
    """Multi-Agent编排器"""
    
    def __init__(self):
        self.message_bus = MessageBus()
        self.agents: Dict[str, BaseAgent] = {}
    
    def add_agent(self, agent: BaseAgent):
        self.agents[agent.name] = agent
        self.message_bus.register(agent)
        
        if isinstance(agent, SupervisorAgent):
            for sub_agent in self.agents.values():
                if sub_agent.name != agent.name:
                    agent.register_agent(sub_agent)
    
    async def execute_task(self, task: str) -> Dict[str, Any]:
        """执行复杂任务"""
        supervisor = self.agents.get("Supervisor")
        if not supervisor:
            raise ValueError("需要注册Supervisor Agent")
        
        # 分解任务
        sub_tasks = await supervisor.decompose_task(task)
        
        # 协调执行
        results = await supervisor.coordinate(sub_tasks, self.message_bus)
        
        # 汇总结果
        final_result = await supervisor.aggregate_results(results)
        
        return {
            "task": task,
            "sub_tasks": len(sub_tasks),
            "results": results,
            "final": final_result
        }
```

---

## 框架对比

| 框架 | 特点 | 适用场景 |
|------|------|---------|
| **LangGraph** | 细粒度状态机控制 | 复杂工作流 |
| **CrewAI** | 角色扮演+任务协作 | 自动化流程 |
| **AutoGen** | 多Agent对话协作 | 研究/原型 |
| **Swarm** | 轻量级Agent编排 | 简单场景 |

---

## 💼 需要Multi-Agent系统开发？

**联系方式：13315868800（微信同号）**
