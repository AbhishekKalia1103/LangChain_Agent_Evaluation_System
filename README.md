# LangChain Agent Evaluation System

A customer-support **refund agent** built with LangChain, plus an evaluation suite that scores it with [DeepEval](https://github.com/confident-ai/deepeval) across seven metrics: tool use, planning, efficiency and answer quality.

## The agent

`agent/` contains a tool-calling agent (`langchain.agents.create_agent`) that handles refund requests while following a fixed policy:

- Identify the customer and look up their purchase history before deciding anything.
- Refund only the items the customer asks for, and only items they actually bought.
- No refunds for purchases older than 30 days or dated in the future.
- If the username doesn't exist, ask for a valid one instead of revealing any data.

| Tool | Purpose |
| --- | --- |
| `get_user_purchase_history` | Returns a user's purchases (mock data in `agent/data.py`) |
| `get_today_date` | Returns today's date, used for the 30-day check |
| `calculate_right_refund` | Sums the price of the requested items |
| `issue_refund_ticket` | Issues the refund ticket |

The agent returns a structured `Answer` (`answer` text plus a `conversation` flag that says whether the dialogue should continue) and uses an in-memory checkpointer for multi-turn chats.

## Evaluations

`evaluations/dataset.py` defines seven goldens: full and partial refunds, an information-only request, an unknown user, an expired purchase, a mixed eligible/expired request, and an item the user never bought. Each golden has an expected output and the expected tool calls.

| Metric | Script | What it checks |
| --- | --- | --- |
| Task Completion | `task_completion.py` | The agent achieved the user's goal within policy |
| Step Efficiency | `step_efficiency.py` | No redundant or unnecessary steps |
| Plan Quality | `plan_quality.py` | The agent's plan is sound |
| Plan Adherence | `plan_adherence.py` | The agent followed its own plan |
| Tool Correctness | `tool_correctness.py` | The right tools were called with the right inputs |
| Argument Correctness | `argument_correctness.py` | Tool arguments match the request |
| Answer Correctness (G-Eval) | `answer_correctness.py` | The final answer reaches the same conclusion as the expected output |

The first four are trace-based: a DeepEval `CallbackHandler` captures the agent's run. The last three are scored on single-turn test cases built from the agent's response.

`eval_all.py` runs all seven metrics while invoking the agent only once per golden.

## Setup

Requires Python 3.12+ and [uv](https://docs.astral.sh/uv/).

```bash
uv sync
cp .env.example .env   # then set OPENAI_API_KEY
```

The agent model is set in `agent/models.py` and the evaluator model in `evaluations/model.py`.

## Usage

Run the full evaluation suite:

```bash
uv run python evaluations/eval_all.py
```

Or run a single metric:

```bash
uv run python evaluations/tool_correctness.py
```

To chat with the agent interactively, uncomment `run_conversation()` at the bottom of `agent/agent.py` and run it from the `agent/` directory:

```bash
cd agent && uv run python agent.py
```

> **Note:** The refund-window checks compare purchase dates in `agent/data.py` against the real current date, so the expected outcomes in the dataset depend on when you run the suite. Update the dates in `data.py` if results drift.

## Project structure

```
agent/
  agent.py        # agent definition + interactive loop
  tools.py        # LangChain tools
  prompts.py      # system prompt / refund policy
  objects.py      # Pydantic models (Answer, PurchaseItem)
  models.py       # chat model config
  data.py         # mock purchase data
evaluations/
  dataset.py      # goldens with expected outputs and tool calls
  model.py        # evaluator model
  eval_all.py     # runs all 7 metrics
  *.py            # one script per metric
```
