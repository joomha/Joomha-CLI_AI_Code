# joomha

**AI-powered CLI for understanding any codebase through conversation.**

[![PyPI](https://img.shields.io/pypi/v/joomha)](https://pypi.org/project/joomha/)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](https://pypi.org/project/joomha/)
[![License](https://img.shields.io/badge/license-MIT-green)](#license)

joomha lets you explore an unfamiliar Git repository by asking questions in natural language. It is built on a RAG (Retrieval-Augmented Generation) pipeline with two retrieval engines you can use alone or compare side by side: **Vector Retrieval** and **Graph Retrieval**.

Supported languages (parsed with Tree-sitter and Python's `ast`): **Python, JavaScript (`.js`, `.jsx`, `.mjs`, `.cjs`) and TypeScript (`.ts`, `.tsx`)**.

---

## Features

- **Vector Retrieval**: cosine similarity search over `all-MiniLM-L6-v2` embeddings (LanceDB).
- **Graph Retrieval**: relational traversal over AST nodes and edges, plus Git co-change analysis (SQLite).
- **Compare Mode**: run both retrievers on the same question and compare the answers.
- **Auto Fallback**: Graph falls back to Vector when no node matches.
- **Multi-provider LLM**: Gemini, OpenAI, Anthropic, DeepSeek, OpenRouter, or any OpenAI-compatible endpoint.
- **Runtime switching**: change provider and model without leaving the session.
- **Rich terminal UI**: banner, panels, tables, spinners and Markdown answers.
- **Hotspots**: list the most frequently changed files from Git history.

---

## Installation

```bash
pip install joomha
```

Requires Python 3.10 or newer.

From source:

```bash
git clone https://github.com/joomha/Joomha-CLI_AI_Code.git
cd Joomha-CLI_AI_Code
pip install -e .
```

---

## Quick Start

```bash
# 1. Save an API key (Gemini is the default provider)
joomha config set gemini YOUR_API_KEY

# 2. Run inside any Git repository
cd /path/to/your/repo
joomha
```

On the first run joomha indexes the repository automatically:

1. Parses the AST of every supported source file.
2. Analyzes Git history (co-changes and hotspots).
3. Builds vector embeddings of code chunks.

The index is stored in a `.joomha/` folder inside the repository and is git-ignored automatically.

---

## Configuration

### Option 1: Environment variables

| Provider | Variable |
|----------|----------|
| Gemini | `GEMINI_API_KEY` |
| OpenAI | `OPENAI_API_KEY` |
| Anthropic | `ANTHROPIC_API_KEY` |
| DeepSeek | `DEEPSEEK_API_KEY` |
| OpenRouter | `OPENROUTER_API_KEY` |
| Custom endpoint | `CUSTOM_LLM_API_KEY` |

```bash
export GEMINI_API_KEY=your-key-here
```

### Option 2: `joomha config` commands

Settings are saved to `~/.joomha/config.json`.

| Command | Description |
|---------|-------------|
| `joomha config set <provider> <key>` | Save an API key. Providers: `gemini`, `openai`, `anthropic`, `deepseek`, `openrouter`, `custom`, `hf_token` |
| `joomha config use <provider>` | Set the default active provider |
| `joomha config model <provider> <model_id>` | Set the default model for a provider |
| `joomha config base-url <url>` | Set the base URL for the `custom` provider |
| `joomha config show` | Show configured providers, active models and key status |

Examples:

```bash
joomha config set openai sk-...
joomha config use openai
joomha config model openai gpt-4o
joomha config base-url http://localhost:11434/v1
joomha config set hf_token hf_...      # optional, for Hugging Face model downloads
joomha config show
```

---

## Command Line Options

| Option | Description |
|--------|-------------|
| `joomha` | Start the interactive session in the current repository |
| `joomha --reindex` | Force a full re-index of the repository |
| `joomha -p, --provider <name>` | Override the provider for this session |
| `joomha -m, --model <id>` | Override the model for this session |
| `joomha -v, --version` | Print the version |

```bash
joomha --provider anthropic --model claude-sonnet-4-6
```

---

## Interactive Commands

Type a question to ask about the code, or use a slash command:

| Command | Description |
|---------|-------------|
| `<question>` | Ask anything about the codebase |
| `/mode vector` | Use Vector Retrieval |
| `/mode graph` | Use Graph Retrieval |
| `/mode compare` | Run both retrievers and compare |
| `/provider` | Pick a provider and model from a list, or type a custom model ID |
| `/info` | Show current provider, model and mode |
| `/hotspots` | Show the 10 most frequently changed files |
| `/help` | Show help |
| `/q`, `/quit` | Exit |

### Example session

```text
[graph|gemini] ❯ How does the authentication system work?

╭──── joomha [graph] ────╮
│  The authentication... │
╰─ Mode: graph │ Context: 3 │ Latency: 1.23s ─╯

[graph|gemini] ❯ /mode compare
✓ Mode changed to: compare

[compare|gemini] ❯ Explain the request handling flow
```

---

## Available Models

The first model of each provider is its default. `openrouter` and `custom` accept any model ID.

| Provider | Models |
|----------|--------|
| Gemini | `gemini-2.0-flash`, `gemini-2.0-flash-lite`, `gemini-2.5-flash`, `gemini-3-flash`, `gemini-3.1-pro` |
| OpenAI | `gpt-4o-mini`, `gpt-4o`, `gpt-4.1`, `o3-mini`, `o4-mini` |
| Anthropic | `claude-3-haiku-20240307`, `claude-3-5-sonnet-20241022`, `claude-haiku-4-5-20251001-v1`, `claude-sonnet-4-6`, `claude-opus-4-7` |
| DeepSeek | `deepseek-chat`, `deepseek-reasoner` |
| OpenRouter | Any model ID, e.g. `google/gemini-2.0-flash-exp:free` |
| Custom | Any OpenAI-compatible endpoint (set with `config base-url`) |

---

## Architecture

```text
Question
   ├── VectorRetriever ── LanceDB cosine search ── Top-5 chunks
   └── GraphRetriever  ── SQLite (AST nodes, edges, co-changes)
                              └── Falls back to VectorRetriever if empty
                                      │
                                      ▼
                  PromptBuilder (identical structure for both modes)
                                      │
                                      ▼
              LLMClient (Gemini / OpenAI / Anthropic / DeepSeek / ...)
                                      │
                                      ▼
                          Rich Markdown output
```

### Project structure

```text
joomha/
├── joomha/
│   ├── cli.py              # Entry point, config commands, REPL
│   ├── config.py           # API keys, providers, model registry
│   ├── orchestrator.py     # RAG pipeline coordinator
│   ├── indexer/
│   │   ├── parsers/        # Language parsers (Python, JS, TS)
│   │   ├── ast_parser.py   # Parser dispatcher, writes to SQLite
│   │   ├── git_analyzer.py # Git history, writes to SQLite
│   │   └── vector_builder.py  # Chunking and embeddings, writes to LanceDB
│   ├── retriever/
│   │   ├── vector.py       # Vector similarity retrieval
│   │   └── graph.py        # Graph retrieval
│   ├── llm/
│   │   ├── client.py       # Multi-provider LLM client
│   │   └── prompt_builder.py
│   └── ui/
│       ├── display.py      # Rich panels and tables
│       └── input_handler.py   # prompt-toolkit session
├── joomhalab/              # Research and evaluation variant
├── pyproject.toml
└── README.md
```

---

## Tech Stack

| Layer | Library |
|-------|---------|
| CLI | typer, rich, prompt-toolkit |
| Parsing | tree-sitter (JS/TS), `ast` (Python) |
| Git | gitpython |
| Embeddings | sentence-transformers (`all-MiniLM-L6-v2`) |
| Vector DB | lancedb, pyarrow |
| Graph DB | sqlite3 |
| LLM SDKs | google-generativeai, openai, anthropic |

---

## License

MIT
