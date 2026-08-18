# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

AutoHedge is an autonomous "agent hedge fund": a pipeline of LLM agents (built on the
[`swarms`](https://swarms.ai) framework) that generate a trading thesis, run quant/risk analysis, and
execute swaps on Solana via the Jupiter Ultra API. Distributed on PyPI as `autohedge`; the primary
user interface is the `autohedge` CLI (an interactive REPL).

## Commands

Dependencies are managed with Poetry (`pyproject.toml`); `requirements.txt` is a stale superset used
only by some CI workflows.

```bash
poetry install                  # install runtime + lint deps
poetry install --with lint      # explicit lint group

python -m autohedge             # run the CLI REPL from a source checkout
autohedge                       # same, once installed (entry point -> autohedge.cli:main)
python example.py               # minimal library usage

python -m autohedge.workers     # run the director agent directly against a hardcoded task

ruff check .                    # lint  (line-length 70)
black .                        # format (line-length 70, preview mode)
```

There is **no test suite** in the repo (no `tests/`, no pytest dependency). Several `.github/workflows/`
files are inherited boilerplate from other Swarms repos and reference things that don't exist here —
`tests/`, a `Makefile`, a `Dockerfile`, a `swarms_torch` package, and `.github/actions/*` composite
actions. Do not treat those workflows as a description of how this project is built or tested. The
workflows that actually do something meaningful are `ruff.yml` and `pylint.yml`.

## Architecture

The whole system is four files in `autohedge/` plus a tools directory.

**`workers.py` is the heart of the system.** Agents are module-level singletons constructed at import
time — not classes, not factories. `director_agent` is a `swarms.Agent` with `handoffs=ALL_AGENTS`,
so the swarms framework itself decides when to delegate to the sentiment, quant, risk, and execution
agents. There is no explicit orchestration loop in this codebase; the README's linear
Director → Quant → Risk → Execution diagram describes intent, while the actual control flow is
handoff-driven and non-deterministic.

Two consequences to keep in mind:
- Importing `autohedge.workers` (transitively, `autohedge`) instantiates agents and stamps the current
  date/time into every system prompt (`_SYSTEM_SUFFIX`). Import is not side-effect-free.
- `AutoHedge.run()` in `main.py` is a thin wrapper: it appends the task to a `swarms.Conversation`,
  calls `director_agent.run(task)` once, and returns the conversation in the shape given by
  `output_type` (`"list"` | `"dict"` | `"str"`). All multi-agent behavior happens inside swarms.

**Prompts live only in `prompts.py`.** Agent system prompts are assembled in `workers.py` by
concatenating a `*_PROMPT` constant with an inline "when you receive a message it will contain…"
addendum plus `_SYSTEM_SUFFIX`. Several `*_PROMPT` templates in `prompts.py`
(`RISK_ASSESSMENT_PROMPT`, `EXECUTION_ORDER_PROMPT`, `DIRECTOR_THESIS_PROMPT`,
`QUANT_ANALYSIS_PROMPT`, `DIRECTOR_DECISION_PROMPT`, `DIRECTOR_TICKER_DISCOVERY_PROMPT`) are dead
code left from an earlier explicit-orchestration design. When editing prompts, change `prompts.py`,
not string literals in `workers.py`.

**Tool wiring is currently incomplete.** Every tool follows the same convention: a plain module-level
function with an exhaustive numpy-style docstring (swarms turns the docstring into the tool schema),
returning a **JSON string**, reading its own credentials from the environment. But only
`exa_search` is actually attached to an agent (`sentiment_agent`). `tools/tools_registry.get_tools()`
and `tools/__init__.py` expose the Solana trading and market-data tools, and nothing calls them — so
despite `execute_trade` existing, no agent can currently place a trade. If you're asked why trades
don't execute, this is why.

Tool groups in `autohedge/tools/`:
- `jupiter_search.py` / `jupiter_price.py` — Solana token lookup and pricing (`JUPITER_API_KEY`)
- `ultra_tools.py` — Jupiter Ultra swaps. `get_order()` returns an unsigned base64 versioned
  transaction + `requestId`; `execute_trade()` signs it locally with `solders` and POSTs to `/execute`.
  This is the only code path that spends real funds.
- `polygon_api.py` — equity fundamentals/OHLC. Named "polygon" but points at `https://api.massive.com`
  and authenticates with `MASSIVE_API_KEY`.
- `yahoo_api.py` — `yfinance` wrapper; deliberately fetches history before `quoteSummary` and returns
  partial data on Yahoo 429s
- `exa_search_tool.py` — web/news search (`EXA_API_KEY`)

**`experimental/`** is not importable and not part of the package: `crypto_agent_wrapper.py` imports a
`cryptoagent` package that isn't a dependency, and `btc_agent.py`/`market_making.py` need
`websocket`/`aiohttp`/`pandas` which aren't declared either. Treat it as reference material.

## Environment

`autohedge/__init__.py` and `cli.py` call `env_loader.load_env()` on import, which walks up from the
CWD to find the nearest `.env` (so the CLI works from any subdirectory) and loads it **without**
overriding already-set variables.

`.env.example` is out of date relative to the code. Actual variables read at runtime:

| Variable | Read by | Notes |
|---|---|---|
| `OPENAI_API_KEY` | swarms (all agents use `gpt-4.1` / `gpt-4o-mini`) | CLI warns if unset |
| `JUPITER_API_KEY` | jupiter_search, jupiter_price, ultra_tools | |
| `SOLANA_PRIVATE_KEY` | `ultra_tools._get_keypair()` | base58. **`.env.example` and the README call this `WALLET_PRIVATE_KEY`, which the code never reads.** |
| `EXA_API_KEY` | exa_search | missing from `.env.example` |
| `MASSIVE_API_KEY` | polygon_api | missing from `.env.example` |
| `WORKSPACE_DIR` | swarms | agent scratch dir |

## Conventions

- Line length is **70** for both ruff and black — much narrower than typical. Match the existing
  wrapping style rather than reformatting to 79/88.
- Logging is `loguru` (`from loguru import logger`) everywhere except the CLI, which uses `rich`.
- Tool functions raise `ValueError` for bad arguments and let `httpx.HTTPError` propagate after
  logging; they never return error strings.
- Agents specify `model_name` as a plain string on the `Agent` constructor. The `swarm-models`
  `OpenAIChat` style only appears in `experimental/`.
