# Example: Agent as Environment

This example demonstrates AEnvironment's unique "Agent as Env" feature, enabling Agent-to-Agent (A2A) communication.

## Overview

In AEnvironment, agents can be treated as environments, allowing:

- **A2A Communication**: Agents interact with other agents as tools
- **Multi-Agent Systems**: Build complex agent ecosystems
- **Agent Testing**: Use agents to test other agents
- **Hierarchical Agents**: Create agent hierarchies

## Basic A2A Pattern

### Creating an Agent Environment

```python
# code_agent/tools.py
from aenv import register_tool

@register_tool
async def analyze_code(code: str, language: str = "python") -> dict:
    """Analyze code for issues and improvements.

    Args:
        code: Source code to analyze
        language: Programming language

    Returns:
        Analysis results with issues and suggestions
    """
    # Agent logic here (could use LLM)
    return {
        "issues": [
            {"line": 5, "type": "style", "message": "Line too long"},
            {"line": 12, "type": "bug", "message": "Potential null reference"}
        ],
        "suggestions": [
            "Consider using type hints",
            "Add docstrings to functions"
        ],
        "quality_score": 7.5
    }

@register_tool
async def fix_code(code: str, issue: dict) -> dict:
    """Fix a specific issue in code.

    Args:
        code: Original code
        issue: Issue to fix

    Returns:
        Fixed code and explanation
    """
    # Agent fixes the code
    return {
        "fixed_code": code,  # Modified code
        "explanation": f"Fixed {issue['type']} at line {issue['line']}"
    }
```

### Using Agent as Environment

```python
from aenv import Environment

async def main():
    # Use code agent as an environment
    async with Environment("code-agent") as code_agent:
        # Call agent capabilities as tools
        analysis = await code_agent.call_tool(
            "analyze_code",
            {
                "code": "def foo():\n    x = None\n    return x.value",
                "language": "python"
            }
        )

        print(f"Analysis: {analysis.content}")

        # Fix issues
        for issue in analysis.content["issues"]:
            fix = await code_agent.call_tool(
                "fix_code",
                {"code": "...", "issue": issue}
            )
            print(f"Fix: {fix.content}")
```

## Multi-Agent Orchestration

### Orchestrator Pattern

```python
from aenv import Environment

class AgentOrchestrator:
    """Orchestrate multiple agent environments."""

    def __init__(self):
        self.agents = {}

    async def register_agent(self, name: str, env_name: str):
        """Register an agent environment."""
        env = Environment(env_name)
        await env.initialize()
        self.agents[name] = env

    async def call_agent(self, agent_name: str, tool: str, args: dict):
        """Call a tool on a specific agent."""
        agent = self.agents.get(agent_name)
        if not agent:
            raise ValueError(f"Agent not found: {agent_name}")
        return await agent.call_tool(tool, args)

    async def pipeline(self, steps: list[dict]):
        """Execute a pipeline of agent calls."""
        results = []
        context = {}

        for step in steps:
            # Substitute context variables
            args = self._substitute(step["args"], context)

            result = await self.call_agent(
                step["agent"],
                step["tool"],
                args
            )

            results.append(result)
            context[step.get("output_key", f"step_{len(results)}")] = result.content

        return results

    def _substitute(self, args: dict, context: dict) -> dict:
        """Substitute context variables in arguments."""
        result = {}
        for key, value in args.items():
            if isinstance(value, str) and value.startswith("$"):
                var_name = value[1:]
                result[key] = context.get(var_name, value)
            else:
                result[key] = value
        return result

    async def close(self):
        for agent in self.agents.values():
            await agent.destroy()

# Usage
async def code_review_pipeline():
    orchestrator = AgentOrchestrator()

    # Register agents
    await orchestrator.register_agent("analyzer", "code-analyzer-agent")
    await orchestrator.register_agent("reviewer", "code-reviewer-agent")
    await orchestrator.register_agent("fixer", "code-fixer-agent")

    # Define pipeline
    pipeline = [
        {
            "agent": "analyzer",
            "tool": "analyze_code",
            "args": {"code": source_code},
            "output_key": "analysis"
        },
        {
            "agent": "reviewer",
            "tool": "review_analysis",
            "args": {"analysis": "$analysis"},
            "output_key": "review"
        },
        {
            "agent": "fixer",
            "tool": "apply_fixes",
            "args": {"code": source_code, "review": "$review"},
            "output_key": "fixed_code"
        }
    ]

    results = await orchestrator.pipeline(pipeline)
    await orchestrator.close()

    return results[-1].content  # Return fixed code
```

### Hierarchical Agents

```python
class HierarchicalAgent:
    """Agent that delegates to sub-agents."""

    def __init__(self, name: str):
        self.name = name
        self.sub_agents = {}

    async def add_sub_agent(self, role: str, env_name: str):
        """Add a sub-agent with a specific role."""
        env = Environment(env_name)
        await env.initialize()
        self.sub_agents[role] = env

    async def delegate(self, task: dict) -> dict:
        """Delegate task to appropriate sub-agent."""
        role = self._determine_role(task)
        agent = self.sub_agents.get(role)

        if not agent:
            return {"error": f"No agent for role: {role}"}

        result = await agent.call_tool(
            task["tool"],
            task["args"]
        )

        return {
            "role": role,
            "result": result.content
        }

    def _determine_role(self, task: dict) -> str:
        """Determine which sub-agent should handle the task."""
        # Logic to route tasks to appropriate agents
        task_type = task.get("type", "default")
        role_mapping = {
            "code": "coder",
            "test": "tester",
            "review": "reviewer",
            "deploy": "deployer"
        }
        return role_mapping.get(task_type, "default")

# Usage
async def hierarchical_example():
    manager = HierarchicalAgent("project-manager")

    await manager.add_sub_agent("coder", "code-agent")
    await manager.add_sub_agent("tester", "test-agent")
    await manager.add_sub_agent("reviewer", "review-agent")

    # Manager delegates tasks
    tasks = [
        {"type": "code", "tool": "write_code", "args": {"spec": "..."}},
        {"type": "test", "tool": "write_tests", "args": {"code": "..."}},
        {"type": "review", "tool": "review_code", "args": {"code": "..."}}
    ]

    results = []
    for task in tasks:
        result = await manager.delegate(task)
        results.append(result)

    return results
```

## Agent Framework Integration

### OpenAI Agents SDK

```python
from openai import OpenAI
from aenv import Environment

class OpenAIAgentEnv:
    """Wrap OpenAI agent as AEnvironment."""

    def __init__(self, model: str = "gpt-4"):
        self.client = OpenAI()
        self.model = model

    async def as_tool(self, name: str, description: str):
        """Create a tool that calls this agent."""

        async def agent_tool(prompt: str) -> dict:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[{"role": "user", "content": prompt}]
            )
            return {"response": response.choices[0].message.content}

        return agent_tool

# Use OpenAI agent as environment
async def use_openai_agent():
    # Create agent environment
    async with Environment("openai-agent-env") as agent:
        # Call agent
        result = await agent.call_tool(
            "chat",
            {"prompt": "Explain quantum computing in simple terms"}
        )
        print(result.content)
```

### CAMEL Integration

```python
from camel.agents import ChatAgent
from aenv import Environment, register_tool

class CAMELAgentEnv:
    """Wrap CAMEL agent as AEnvironment."""

    def __init__(self, system_message: str):
        self.agent = ChatAgent(system_message=system_message)

    @register_tool
    async def chat(self, message: str) -> dict:
        """Chat with the CAMEL agent."""
        response = self.agent.step(message)
        return {"response": response.msg.content}

    @register_tool
    async def reset(self) -> dict:
        """Reset agent conversation."""
        self.agent.reset()
        return {"status": "reset"}

# Multi-agent debate
async def agent_debate():
    async with Environment("camel-agent-1") as agent1:
        async with Environment("camel-agent-2") as agent2:
            topic = "AI will surpass human intelligence by 2030"

            # Agent 1 argues for
            arg1 = await agent1.call_tool(
                "chat",
                {"message": f"Argue FOR: {topic}"}
            )

            # Agent 2 argues against
            arg2 = await agent2.call_tool(
                "chat",
                {"message": f"Argue AGAINST: {topic}. Counter: {arg1.content}"}
            )

            # Continue debate...
            return {"for": arg1.content, "against": arg2.content}
```

## A2A Communication Patterns

### Request-Response

```python
async def request_response():
    """Simple request-response between agents."""
    async with Environment("agent-a") as agent_a:
        async with Environment("agent-b") as agent_b:
            # Agent A sends request to Agent B
            request = {"task": "analyze", "data": "..."}
            response = await agent_b.call_tool("process", request)

            # Agent A uses response
            result = await agent_a.call_tool(
                "use_analysis",
                {"analysis": response.content}
            )
            return result
```

### Publish-Subscribe

```python
class AgentPubSub:
    """Publish-subscribe pattern for agents."""

    def __init__(self):
        self.subscribers = {}

    def subscribe(self, topic: str, agent: Environment):
        if topic not in self.subscribers:
            self.subscribers[topic] = []
        self.subscribers[topic].append(agent)

    async def publish(self, topic: str, message: dict):
        """Publish message to all subscribers."""
        agents = self.subscribers.get(topic, [])
        results = await asyncio.gather(*[
            agent.call_tool("on_message", {"topic": topic, "message": message})
            for agent in agents
        ])
        return results

# Usage
pubsub = AgentPubSub()

async with Environment("logger-agent") as logger:
    async with Environment("monitor-agent") as monitor:
        pubsub.subscribe("events", logger)
        pubsub.subscribe("events", monitor)

        await pubsub.publish("events", {"type": "error", "msg": "..."})
```

### Consensus

```python
async def agent_consensus(agents: list[Environment], question: str) -> dict:
    """Get consensus from multiple agents."""

    # Collect votes
    votes = await asyncio.gather(*[
        agent.call_tool("vote", {"question": question})
        for agent in agents
    ])

    # Count votes
    vote_counts = {}
    for vote in votes:
        answer = vote.content.get("answer")
        vote_counts[answer] = vote_counts.get(answer, 0) + 1

    # Determine consensus
    total = len(votes)
    for answer, count in vote_counts.items():
        if count > total / 2:
            return {"consensus": answer, "confidence": count / total}

    return {"consensus": None, "votes": vote_counts}
```

## Testing Agents with Agents

### Agent Test Framework

```python
class AgentTester:
    """Use agents to test other agents."""

    def __init__(self, target_env: str, tester_env: str):
        self.target_env = target_env
        self.tester_env = tester_env

    async def run_tests(self, test_cases: list[dict]) -> dict:
        """Run test cases against target agent."""
        async with Environment(self.target_env) as target:
            async with Environment(self.tester_env) as tester:
                results = []

                for test in test_cases:
                    # Tester generates input
                    input_data = await tester.call_tool(
                        "generate_input",
                        {"test_case": test}
                    )

                    # Target processes input
                    output = await target.call_tool(
                        test["tool"],
                        input_data.content
                    )

                    # Tester evaluates output
                    evaluation = await tester.call_tool(
                        "evaluate_output",
                        {
                            "expected": test["expected"],
                            "actual": output.content
                        }
                    )

                    results.append({
                        "test": test["name"],
                        "passed": evaluation.content["passed"],
                        "details": evaluation.content
                    })

                return {
                    "total": len(results),
                    "passed": sum(1 for r in results if r["passed"]),
                    "results": results
                }

# Usage
tester = AgentTester("code-agent", "test-agent")
results = await tester.run_tests([
    {
        "name": "test_simple_function",
        "tool": "generate_code",
        "expected": {"has_docstring": True, "passes_lint": True}
    },
    # More test cases...
])
```

### Adversarial Testing

```python
async def adversarial_test(target_env: str, adversary_env: str):
    """Test agent against adversarial inputs."""
    async with Environment(target_env) as target:
        async with Environment(adversary_env) as adversary:
            # Adversary generates challenging inputs
            challenges = await adversary.call_tool(
                "generate_challenges",
                {"difficulty": "hard", "count": 10}
            )

            results = []
            for challenge in challenges.content:
                # Target attempts challenge
                response = await target.call_tool(
                    "solve",
                    {"challenge": challenge}
                )

                # Adversary evaluates
                score = await adversary.call_tool(
                    "evaluate",
                    {"challenge": challenge, "response": response.content}
                )

                results.append(score.content)

            return {
                "challenges": len(results),
                "avg_score": sum(r["score"] for r in results) / len(results)
            }
```
