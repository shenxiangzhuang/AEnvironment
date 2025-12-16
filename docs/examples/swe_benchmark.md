# Example: SWE-bench Evaluation

This example demonstrates how to use AEnvironment for running SWE-bench evaluations at scale.

## Overview

[SWE-bench](https://www.swebench.com/) is a benchmark for evaluating LLMs on real-world software engineering tasks. AEnvironment provides:

- Fast environment creation for each task instance
- Isolated execution environments
- Parallel evaluation support
- Built-in test execution

## Quick Start

### Using Built-in SWE Environment

```python
import asyncio
from aenv import Environment

async def evaluate_instance(instance_id: str, patch: str):
    """Evaluate a single SWE-bench instance."""

    async with Environment(
        "swe-env",
        datasource=f"swe-bench/{instance_id}",
        ttl="30m"
    ) as env:
        # Apply the patch
        result = await env.call_tool(
            "apply_patch",
            {"patch": patch}
        )

        if result.is_error:
            return {"status": "patch_failed", "error": str(result.content)}

        # Run tests
        test_result = await env.call_tool(
            "run_tests",
            {"timeout": 300}
        )

        return {
            "status": "completed",
            "tests_passed": test_result.content.get("passed", False),
            "test_output": test_result.content.get("output", "")
        }

# Run evaluation
result = asyncio.run(evaluate_instance(
    "django__django-11099",
    "diff --git a/..."
))
print(result)
```

## SWE Environment Tools

The `swe-env` environment provides these tools:

### File Operations

```python
# Read file
content = await env.call_tool("read_file", {"path": "src/main.py"})

# Write file
await env.call_tool("write_file", {
    "path": "src/main.py",
    "content": "# Updated content"
})

# List directory
files = await env.call_tool("list_directory", {"path": "src/"})
```

### Code Search

```python
# Search for code patterns
results = await env.call_tool("search_code", {
    "query": "def authenticate",
    "file_pattern": "*.py"
})

# Find references
refs = await env.call_tool("find_references", {
    "symbol": "UserModel",
    "path": "src/"
})
```

### Git Operations

```python
# Apply patch
await env.call_tool("apply_patch", {"patch": patch_content})

# Get diff
diff = await env.call_tool("get_diff", {})

# Reset changes
await env.call_tool("reset_changes", {})
```

### Test Execution

```python
# Run all tests
result = await env.call_tool("run_tests", {"timeout": 300})

# Run specific tests
result = await env.call_tool("run_tests", {
    "test_path": "tests/test_auth.py",
    "timeout": 60
})
```

### Shell Commands

```python
# Execute shell command
result = await env.call_tool("execute_command", {
    "command": "python -m pytest tests/ -v",
    "timeout": 120
})
```

## Parallel Evaluation

### Batch Evaluation

```python
import asyncio
from aenv import Environment

async def evaluate_batch(instances: list[dict], max_concurrent: int = 10):
    """Evaluate multiple instances in parallel."""

    semaphore = asyncio.Semaphore(max_concurrent)

    async def evaluate_one(instance):
        async with semaphore:
            return await evaluate_instance(
                instance["id"],
                instance["patch"]
            )

    tasks = [evaluate_one(inst) for inst in instances]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    return [
        {"instance": inst["id"], "result": res}
        for inst, res in zip(instances, results)
    ]

# Run batch evaluation
instances = [
    {"id": "django__django-11099", "patch": "..."},
    {"id": "django__django-11133", "patch": "..."},
    # ... more instances
]

results = asyncio.run(evaluate_batch(instances, max_concurrent=20))
```

### Distributed Evaluation

```python
import asyncio
from aenv import Environment

class DistributedEvaluator:
    def __init__(self, aenv_url: str, workers: int = 50):
        self.aenv_url = aenv_url
        self.workers = workers
        self.queue = asyncio.Queue()
        self.results = []

    async def worker(self, worker_id: int):
        while True:
            instance = await self.queue.get()
            if instance is None:
                break

            try:
                result = await self.evaluate(instance)
                self.results.append(result)
            except Exception as e:
                self.results.append({
                    "instance": instance["id"],
                    "error": str(e)
                })
            finally:
                self.queue.task_done()

    async def evaluate(self, instance: dict):
        async with Environment(
            "swe-env",
            aenv_url=self.aenv_url,
            datasource=f"swe-bench/{instance['id']}"
        ) as env:
            # Apply patch and run tests
            await env.call_tool("apply_patch", {"patch": instance["patch"]})
            result = await env.call_tool("run_tests", {"timeout": 300})

            return {
                "instance": instance["id"],
                "passed": result.content.get("passed", False)
            }

    async def run(self, instances: list[dict]):
        # Start workers
        workers = [
            asyncio.create_task(self.worker(i))
            for i in range(self.workers)
        ]

        # Add instances to queue
        for instance in instances:
            await self.queue.put(instance)

        # Wait for completion
        await self.queue.join()

        # Stop workers
        for _ in workers:
            await self.queue.put(None)
        await asyncio.gather(*workers)

        return self.results
```

## Integration with RL Training

### AReaL Integration

```python
from aenv import Environment
from areal import RLTrainer

class SWEEnvironmentWrapper:
    """Wrapper for using SWE-env with AReaL."""

    def __init__(self, instance_id: str):
        self.instance_id = instance_id
        self.env = None

    async def reset(self):
        """Reset environment to initial state."""
        if self.env:
            await self.env.destroy()

        self.env = Environment(
            "swe-env",
            datasource=f"swe-bench/{self.instance_id}"
        )
        await self.env.initialize()

        # Get initial state
        files = await self.env.call_tool("list_directory", {"path": "."})
        return {"files": files.content}

    async def step(self, action: dict):
        """Execute action and return observation, reward, done."""

        # Execute action (e.g., edit file, run command)
        result = await self.env.call_tool(
            action["tool"],
            action["arguments"]
        )

        # Check if task is complete
        test_result = await self.env.call_tool("run_tests", {"timeout": 60})
        done = test_result.content.get("passed", False)

        # Calculate reward
        reward = await self.env.get_reward()

        return {
            "observation": result.content,
            "reward": reward,
            "done": done
        }

    async def close(self):
        if self.env:
            await self.env.destroy()

# Use with AReaL
async def train_swe_agent():
    env = SWEEnvironmentWrapper("django__django-11099")

    trainer = RLTrainer(
        env=env,
        algorithm="ppo",
        # ... other config
    )

    await trainer.train(num_episodes=1000)
```

### Reward Function

```python
from aenv import register_reward

@register_reward
def swe_reward(
    test_results: dict,
    patch_quality: dict,
    steps_taken: int
) -> float:
    """Calculate reward for SWE-bench task.

    Args:
        test_results: Test execution results
        patch_quality: Code quality metrics
        steps_taken: Number of actions taken

    Returns:
        Reward value between -1 and 1
    """
    reward = 0.0

    # Primary reward: tests passing
    if test_results.get("all_passed"):
        reward += 0.7
    elif test_results.get("some_passed"):
        pass_rate = test_results["passed"] / test_results["total"]
        reward += 0.3 * pass_rate

    # Bonus for code quality
    if patch_quality.get("lint_clean"):
        reward += 0.1

    # Penalty for too many steps
    if steps_taken > 50:
        reward -= 0.1 * (steps_taken - 50) / 100

    return max(-1.0, min(1.0, reward))
```

## Custom SWE Environment

### Creating Custom Environment

```python
# swe-custom-env/src/tools.py
from aenv import register_tool, register_reward
import subprocess
import os

@register_tool
def read_file(path: str) -> dict:
    """Read file content."""
    try:
        with open(path, 'r') as f:
            return {"content": f.read(), "path": path}
    except FileNotFoundError:
        return {"error": f"File not found: {path}"}

@register_tool
def write_file(path: str, content: str) -> dict:
    """Write content to file."""
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, 'w') as f:
        f.write(content)
    return {"success": True, "path": path}

@register_tool
def apply_patch(patch: str) -> dict:
    """Apply a git patch."""
    result = subprocess.run(
        ["git", "apply", "--check", "-"],
        input=patch.encode(),
        capture_output=True
    )

    if result.returncode != 0:
        return {"success": False, "error": result.stderr.decode()}

    subprocess.run(["git", "apply", "-"], input=patch.encode())
    return {"success": True}

@register_tool
def run_tests(test_path: str = "", timeout: int = 300) -> dict:
    """Run pytest tests."""
    cmd = ["python", "-m", "pytest", "-v"]
    if test_path:
        cmd.append(test_path)

    try:
        result = subprocess.run(
            cmd,
            capture_output=True,
            timeout=timeout
        )

        output = result.stdout.decode()
        passed = result.returncode == 0

        return {
            "passed": passed,
            "output": output,
            "returncode": result.returncode
        }
    except subprocess.TimeoutExpired:
        return {"passed": False, "error": "Test timeout"}

@register_reward
def calculate_reward(test_results: dict) -> float:
    """Calculate reward based on test results."""
    if test_results.get("passed"):
        return 1.0
    return -0.1
```

## Performance Tips

### 1. Use Environment Pools

```python
# Pre-warm environments for faster startup
async with Environment("swe-env", pool_size=10) as env:
    # Environments are pre-created
    pass
```

### 2. Batch File Operations

```python
# Instead of multiple read_file calls
files = await env.call_tool("read_files", {
    "paths": ["file1.py", "file2.py", "file3.py"]
})
```

### 3. Parallel Test Execution

```python
# Run tests in parallel across environments
results = await asyncio.gather(*[
    env.call_tool("run_tests", {"test_path": path})
    for path in test_paths
])
```

### 4. Cache Repository Data

```python
# Use cached datasource
env = Environment(
    "swe-env",
    datasource="cache://swe-bench/django__django-11099"
)
```

## Metrics and Monitoring

### Evaluation Metrics

```python
def calculate_metrics(results: list[dict]) -> dict:
    total = len(results)
    passed = sum(1 for r in results if r.get("passed"))

    return {
        "total_instances": total,
        "passed": passed,
        "pass_rate": passed / total if total > 0 else 0,
        "avg_time": sum(r.get("time", 0) for r in results) / total
    }
```

### Logging

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("swe-eval")

async def evaluate_with_logging(instance: dict):
    logger.info(f"Starting evaluation: {instance['id']}")

    start_time = time.time()
    result = await evaluate_instance(instance["id"], instance["patch"])
    elapsed = time.time() - start_time

    logger.info(
        f"Completed {instance['id']}: "
        f"passed={result.get('passed')}, time={elapsed:.2f}s"
    )

    return result
```
