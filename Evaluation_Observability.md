# Lab: Build an Evaluation and Observability Pipeline for an Agent

### What you will build

You will create this flow:

```text
User Question
     ↓
Claude Code Agent
     ↓
Agent Response
     ↓
┌──────────────────────────────┐
│ Evaluation                   │
│                              │
│ • Correctness                │
│ • Completeness               │
│ • Response quality           │
└──────────────────────────────┘
     ↓
┌──────────────────────────────┐
│ Observability                │
│                              │
│ • Execution time             │
│ • Token usage                │
│ • Tool calls                 │
│ • Errors                     │
│ • Evaluation score           │
└──────────────────────────────┘
     ↓
evaluation-results.json
```

The important distinction is:

**Evaluation = "Was the agent's answer good?"**

**Observability = "What happened while the agent was running?"**

---

# 1. Lab Objectives

By the end of this lab, participants will be able to:

- Run an agent using Claude Code.
- Create a set of evaluation test cases.
- Capture agent execution information.
- Evaluate responses against expected results.
- Calculate an evaluation score.
- Record execution metrics.
- Generate a JSON observability report.
- Identify failed evaluations.
- Understand how this can become a CI/CD evaluation pipeline.

---

# 2. Prerequisites

You need:

- Windows
- PowerShell
- Python 3.x
- Node.js
- npm
- Claude Code
- Claude API key

You can verify:

```powershell
node --version
```

```powershell
npm --version
```

```powershell
python --version
```

Check Claude Code:

```powershell
claude --version
```

---

# 3. Set Claude API Key

In PowerShell:

```powershell
$env:ANTHROPIC_API_KEY="your-api-key"
```

Verify:

```powershell
$env:ANTHROPIC_API_KEY
```

For a permanent Windows environment variable:

```powershell
[System.Environment]::SetEnvironmentVariable(
    "ANTHROPIC_API_KEY",
    "your-api-key",
    "User"
)
```

Then open a **new PowerShell window**.

> For a training lab, avoid committing the API key into Git.

---

# 4. Create Lab Directory

```powershell
mkdir claude-agent-evaluation
cd claude-agent-evaluation
```

Create the following structure:

```text
claude-agent-evaluation/
│
├── agent.py
├── evaluator.py
├── observability.py
├── run_evaluation.py
├── test_cases.json
└── results/
```

Create the results directory:

```powershell
mkdir results
```

---

# 5. Create Evaluation Test Cases

Create:

```text
test_cases.json
```

Add:

```json
[
  {
    "id": "TC001",
    "question": "What is the capital of France?",
    "expected": "Paris"
  },
  {
    "id": "TC002",
    "question": "What is 10 multiplied by 5?",
    "expected": "50"
  },
  {
    "id": "TC003",
    "question": "Name the largest planet in our solar system.",
    "expected": "Jupiter"
  },
  {
    "id": "TC004",
    "question": "What programming language is primarily used with Django?",
    "expected": "Python"
  },
  {
    "id": "TC005",
    "question": "What is the boiling point of water at sea level in Celsius?",
    "expected": "100"
  }
]
```

These are our **evaluation datasets**.

---

# 6. Create the Agent

Create:

```text
agent.py
```

Use:

```python
import os
import anthropic


client = anthropic.Anthropic(
    api_key=os.environ["ANTHROPIC_API_KEY"]
)


def run_agent(question):

    response = client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=500,
        system="Answer the user's question accurately and concisely.",
        messages=[
            {
                "role": "user",
                "content": question
            }
        ]
    )

    answer = response.content[0].text

    return {
        "answer": answer,
        "input_tokens": response.usage.input_tokens,
        "output_tokens": response.usage.output_tokens
    }
```

Install the Python SDK:

```powershell
pip install anthropic
```

---

# 7. Test the Agent

Run Python:

```powershell
python
```

Then:

```python
from agent import run_agent

result = run_agent("What is the capital of France?")

print(result)
```

You should get something similar to:

```text
{
    'answer': 'The capital of France is Paris.',
    'input_tokens': 20,
    'output_tokens': 12
}
```

Exit:

```python
exit()
```

---

# 8. Create the Evaluator

Now we need to answer:

> How do we know whether the agent produced a correct answer?

Create:

```text
evaluator.py
```

```python
def evaluate_answer(actual, expected):

    actual = actual.lower()
    expected = expected.lower()

    if expected in actual:
        return {
            "passed": True,
            "score": 1.0,
            "reason": "Expected answer found in agent response"
        }

    return {
        "passed": False,
        "score": 0.0,
        "reason": "Expected answer not found in agent response"
    }
```

This is a very simple evaluator.

For example:

```text
Expected:
Paris

Agent:
The capital of France is Paris.

Result:
PASS
```

But:

```text
Expected:
Paris

Agent:
The capital of France is London.

Result:
FAIL
```

---

# 9. Create Observability

Now we want to record what happened during execution.

Create:

```text
observability.py
```

```python
import time


class AgentTrace:

    def __init__(self, test_id):
        self.test_id = test_id
        self.start_time = time.time()

    def finish(self, result, evaluation):

        end_time = time.time()

        return {
            "test_id": self.test_id,
            "duration_seconds": round(
                end_time - self.start_time,
                3
            ),
            "input_tokens": result["input_tokens"],
            "output_tokens": result["output_tokens"],
            "total_tokens":
                result["input_tokens"] +
                result["output_tokens"],
            "answer": result["answer"],
            "evaluation": evaluation
        }
```

Now we are capturing:

```text
Test ID
Duration
Input tokens
Output tokens
Total tokens
Agent response
Evaluation result
```

---

# 10. Create the Evaluation Pipeline

Create:

```text
run_evaluation.py
```

```python
import json
import os

from agent import run_agent
from evaluator import evaluate_answer
from observability import AgentTrace


with open("test_cases.json", "r") as file:
    test_cases = json.load(file)


results = []


for test in test_cases:

    print(f"\nRunning {test['id']}...")

    trace = AgentTrace(test["id"])

    try:

        result = run_agent(test["question"])

        evaluation = evaluate_answer(
            result["answer"],
            test["expected"]
        )

        trace_result = trace.finish(
            result,
            evaluation
        )

        trace_result["question"] = test["question"]
        trace_result["expected"] = test["expected"]
        trace_result["status"] = "success"

        results.append(trace_result)

        print(
            f"Result: "
            f"{'PASS' if evaluation['passed'] else 'FAIL'}"
        )

    except Exception as e:

        results.append({
            "test_id": test["id"],
            "question": test["question"],
            "status": "error",
            "error": str(e)
        })

        print(f"ERROR: {e}")


total = len(results)

passed = sum(
    1
    for r in results
    if r.get("evaluation", {}).get("passed") is True
)


score = passed / total * 100


summary = {
    "total_tests": total,
    "passed": passed,
    "failed": total - passed,
    "evaluation_score": round(score, 2),
    "results": results
}


os.makedirs("results", exist_ok=True)


with open(
    "results/evaluation-results.json",
    "w"
) as file:

    json.dump(
        summary,
        file,
        indent=2
    )


print("\n==============================")
print("Evaluation Complete")
print("==============================")

print(f"Total Tests : {total}")
print(f"Passed      : {passed}")
print(f"Failed      : {total - passed}")
print(f"Score       : {score:.2f}%")

print(
    "\nReport:"
    " results/evaluation-results.json"
)
```

---

# 11. Run the Pipeline

Execute:

```powershell
python run_evaluation.py
```

You should see something similar to:

```text
Running TC001...
Result: PASS

Running TC002...
Result: PASS

Running TC003...
Result: PASS

Running TC004...
Result: PASS

Running TC005...
Result: PASS

==============================
Evaluation Complete
==============================

Total Tests : 5
Passed      : 5
Failed      : 0
Score       : 100.00%

Report: results/evaluation-results.json
```

---

# 12. View the Observability Report

Open:

```powershell
notepad results\evaluation-results.json
```

You will see something similar to:

```json
{
  "total_tests": 5,
  "passed": 5,
  "failed": 0,
  "evaluation_score": 100.0,
  "results": [
    {
      "test_id": "TC001",
      "duration_seconds": 1.42,
      "input_tokens": 24,
      "output_tokens": 15,
      "total_tokens": 39,
      "answer": "The capital of France is Paris.",
      "evaluation": {
        "passed": true,
        "score": 1.0,
        "reason": "Expected answer found in agent response"
      },
      "question": "What is the capital of France?",
      "expected": "Paris",
      "status": "success"
    }
  ]
}
```

This is your basic **agent observability data**.

---

# 13. Understand Evaluation vs Observability

This is the most important part of the lab.

### Evaluation

Evaluation asks:

```text
Did the agent do the right thing?
```

For example:

```text
Expected = Paris

Agent response = The capital of France is Paris.

Evaluation = PASS
```

We can calculate:

```text
Evaluation Score =
Passed Tests / Total Tests × 100
```

Example:

```text
9 passed
1 failed

9 / 10 × 100

= 90%
```

---

### Observability

Observability asks:

```text
What happened while the agent was working?
```

For example:

```text
Test ID          TC001
Duration         1.42 seconds
Input Tokens     24
Output Tokens    15
Total Tokens     39
Status           success
Evaluation       PASS
```

So:

> **Evaluation tells us the quality of the result.**

> **Observability tells us what happened during execution.**

---

# 14. Add a Failure Test

Now let's intentionally introduce a failure.

Change TC003:

```json
{
  "id": "TC003",
  "question": "Name the largest planet in our solar system.",
  "expected": "Mars"
}
```

Run:

```powershell
python run_evaluation.py
```

You should now get:

```text
Total Tests : 5
Passed      : 4
Failed      : 1
Score       : 80.00%
```

This demonstrates why evaluations are useful.

The agent may execute successfully:

```text
Agent execution = SUCCESS
```

but the answer can still be wrong:

```text
Evaluation = FAIL
```

That's an important concept in **agentic AI systems**.

---

# 15. Add Quality Evaluation

Our first evaluator only checks whether the expected word exists.

Real agent evaluation can have multiple dimensions:

```text
                 Agent Response
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
      Correctness  Relevance   Completeness
          │            │            │
          └────────────┼────────────┘
                       ↓
                  Final Score
```

For example:

```text
Correctness     40%
Relevance       30%
Completeness    20%
Format          10%
```

You can calculate:

```text
Final Score =
Correctness × 0.40
+ Relevance × 0.30
+ Completeness × 0.20
+ Format × 0.10
```

This is closer to how production agent evaluation works.

---

# 16. Make Claude the Evaluator

A more advanced version is **LLM-as-a-Judge**.

Instead of Python simply checking:

```python
if expected in actual:
```

we can ask Claude:

```text
Evaluate the following agent response.

Question:
What is the capital of France?

Expected answer:
Paris

Agent response:
The capital of France is Paris.

Give a score from 0 to 10 and explain why.
```

Claude could return:

```json
{
  "score": 10,
  "reason": "The response correctly identifies Paris as the capital of France."
}
```

This is called:

**LLM-as-a-Judge evaluation.**

It is useful when answers are open-ended and cannot be evaluated using simple string matching.

---

# 17. Introduce an Evaluation Threshold

Production systems normally define a minimum acceptable score.

For example:

```text
Minimum acceptable score = 80%
```

Add this concept to the pipeline:

```text
Evaluation
    ↓
Score = 85%
    ↓
85 >= 80
    ↓
PASS
```

But:

```text
Evaluation
    ↓
Score = 65%
    ↓
65 < 80
    ↓
FAIL
```

This allows the evaluation pipeline to act as a **quality gate**.

---

# 18. Final Architecture

Your completed lab concept becomes:

```text
                   ┌───────────────┐
                   │ Test Dataset  │
                   │               │
                   │ Questions     │
                   │ Expected      │
                   └───────┬───────┘
                           │
                           ↓
                  ┌─────────────────┐
                  │  Claude Agent   │
                  └────────┬────────┘
                           │
                           ↓
                  ┌─────────────────┐
                  │ Agent Response  │
                  └────────┬────────┘
                           │
              ┌────────────┴────────────┐
              ↓                         ↓
       ┌──────────────┐         ┌────────────────┐
       │  Evaluation  │         │ Observability  │
       │              │         │                │
       │ Correctness  │         │ Duration       │
       │ Quality      │         │ Tokens         │
       │ Score        │         │ Errors         │
       └──────┬───────┘         │ Trace          │
              │                 └───────┬────────┘
              │                         │
              └────────────┬────────────┘
                           ↓
                  ┌──────────────────┐
                  │ Evaluation       │
                  │ Report           │
                  │                  │
                  │ PASS / FAIL     │
                  │ Score           │
                  │ Metrics         │
                  └──────────────────┘
```

## Suggested participant challenge

After completing the basic lab, give participants this task:

> **Modify the pipeline to evaluate 10 questions and calculate an overall agent quality score. Add an 80% quality gate. If the score is below 80%, the Python program should exit with a failure status. Also record execution time and token usage for every test case.**

Then introduce the production concept:

```text
Agent
  ↓
Evaluation
  ↓
Quality Gate
  ↓
Observability
  ↓
Report
  ↓
CI/CD Pipeline
```

