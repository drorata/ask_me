# ask-me

A terminal Q&A tool powered by [opencode](https://opencode.ai). Ask technical questions and get concise answers rendered in your terminal.

## Setup

Add this to your `.zshrc`:

```zsh
ASK_DIR="$HOME/<Path to this project>"
ask() {
    (cd "$ASK_DIR" && opencode run "$@") | glow
}
```

Requires:
- [opencode](https://opencode.ai) installed and configured
- [glow](https://github.com/charmbracelet/glow) for terminal markdown rendering

## Usage

```
ask "How to rename a git branch locally and remotely?"
ask "Explain the difference between COPY and ADD in a Dockerfile"
ask "PostgreSQL query to find duplicate rows"
```

## How it works

This project defines a project-local opencode agent (`ask_me`) that is optimized for quick, direct answers. When you run `ask`, opencode launches in this workspace with the `ask_me` agent as default, and the output is piped through `glow` for readable terminal rendering.

## Configuration

### Changing the model

To list all available models, run:

```
opencode models
```

This prints every model in `<provider>/<model-id>` format. Use that string in `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "<provider>/<model-id>",
  "default_agent": "ask_me"
}
```

### Recommended models for Q&A

| Tradeoff     | Example                                                  | Notes                             |
| ------------ | -------------------------------------------------------- | --------------------------------- |
| Fast + cheap | A small/flash model (e.g. Gemini Flash, Haiku, GPT Mini) | Best for simple factual questions |
| Balanced     | A mid-tier model (e.g. Sonnet, GPT-4o)                   | Good default for most questions   |
| Heavy        | A frontier model (e.g. Opus, GPT-5, o3)                  | For complex or nuanced questions  |

Pick based on how much you value speed vs. answer depth. For most `ask` usage, a balanced model is sufficient.

### Agent guardrails

The agent (`.opencode/agents/ask_me.md`) is locked down:

| Guardrail       | Value | Effect                                               |
| --------------- | ----- | ---------------------------------------------------- |
| `steps`         | 10    | Max 10 tool-call rounds per question                 |
| `temperature`   | 0.2   | Deterministic, factual responses                     |
| `doom_loop`     | deny  | Fails fast instead of retrying                       |
| `webfetch`      | allow | Can fetch URLs for documentation                     |
| `websearch`     | allow | Can search the web for current info                  |
| Everything else | deny  | No file access, no commands, no codebase exploration |

Context7 (library documentation lookup) is available via MCP from the project-level opencode config.

## Verifying Usage of MCP and websearch

### MCP (e.g., Context7)

To confirm the agent is actually calling Context7 or web search (and not just answering from training data), run with debug logs:

```zsh
(cd "$ASK_DIR" && opencode run --print-logs --log-level DEBUG "What is the Effect library in TypeScript?") 2>ask-debug.log | glow
```

Then inspect the log for tool calls:

```zsh
grep '⚙' ask-debug.log
```

Expected output (example):

```
⚙ context7_resolve-library-id {"libraryName":"Effect","query":"What is the Effect library in TypeScript?"}
⚙ context7_query-docs {"libraryId":"/effect-ts/effect","query":"What is Effect? Quick introduction..."}
```

You can also export the session as JSON for detailed inspection:

```zsh
(cd "$ASK_DIR" && opencode export SESSION_ID) | jq '.messages[].parts[] | select(.type == "tool") | {tool, input: .state.input}'
```

If no `⚙` lines or tool parts appear, the agent answered purely from its training data (which is fine for simple questions — tools are used only when the agent determines it needs external information).

### Verifying web access

The agent has two web-related tools configured:

- **`webfetch`** — Fetches a specific URL directly.
- **`websearch`** — Performs a search query via a dedicated search tool.

In practice, the model may use either or both depending on the provider and model. Some models simulate searching by using `webfetch` against a search engine's HTML endpoint (e.g., DuckDuckGo). Both approaches result in the agent retrieving live web information.

To check whether the agent accessed the web at all, look for `WebFetch` or `websearch` lines in the debug log:

```zsh
grep 'web' ask-debug.log
```

Example output:

```
% WebFetch https://html.duckduckgo.com/html/?q=KubeCon+June+2026+announcements
% WebFetch https://opensource.microsoft.com/blog/2026/06/17/whats-new-with-microsoft...
```

Or from the session JSON:

```zsh
(cd "$ASK_DIR" && opencode export SESSION_ID) | jq '.messages[].parts[] | select(.type == "tool") | {tool, url: .state.input.url // .state.input.query // .state.input}'
```

**Test questions by tool type:**

| Expected behavior        | Example question                                |
| ------------------------ | ----------------------------------------------- |
| Direct fetch (known URL) | "What is the current status of the HTTP/3 RFC?" |
| Web discovery (search)   | "What was announced at KubeCon this week?"      |
| Web discovery (search)   | "What is the latest release of Bun?"            |
| Context7 (library docs)  | "How to use Effect.gen in Effect-TS?"           |

## Project structure

```
.
├── opencode.json                 # Project config (model, default agent)
├── .opencode/
│   └── agents/
│       └── ask_me.md             # Agent definition
└── README.md
```
