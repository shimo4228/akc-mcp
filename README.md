# AKC MCP

An MCP (Model Context Protocol) server that provides Agent Knowledge Cycle (AKC) cognitive tools for AI agents: memory distillation, identity evolution, skill extraction, and ethical governance. AKC is the author's cycle that turns an agent's logged experience into knowledge, a self-description, skills and rules. It is for developers who want to try the memory cycle of an autonomous agent from another AI agent or MCP client.

Born from the [contemplative-agent](https://github.com/shimo4228/contemplative-agent) framework, this package re-implements the cognitive layer as a standalone MCP server that an AI agent or MCP client calls over HTTP.

**Status:** alpha (version 0.1.0, not published on PyPI); the last code change was on 2026-04-10. The tools call the Anthropic API with your API key, so the episodes, knowledge, skills and rules they work on are sent to Anthropic and billed to your account. The tools work only over HTTP from a client that sends custom headers; over stdio, the way Claude Desktop launches a server, every tool returns an error (see [Quick Start](#quick-start)).

## What It Does

AKC MCP gives AI agents a structured inner life:

- **Memory Distillation**: extract recurring patterns from experience logs
- **Identity Evolution**: distill accumulated knowledge into a coherent self-description
- **Skill Extraction**: synthesize learned patterns into reusable behavioral skills
- **Rule Distillation**: extract universal behavioral principles from skills
- **Constitutional Amendment**: propose changes to the constitution, the Markdown files of ethical principles the agent works under
- **Quality Audit**: detect redundancy and structural issues in skills/rules

The distillation and audit tools return proposals as text and do not save them, so the calling agent decides what to keep (the approval gate). Two tools save results: `record_episode` appends to the episode log, and `distill` saves patterns to `knowledge.json` when called with `write=true`. The only other writes are default files: at startup the server creates a blank identity, constitution and knowledge store in the data directory, and a key's first tool or resource call creates the same files in that key's folder (see [Quick Start](#quick-start)). Where approved proposals go is under [Data Format Compatibility](#data-format-compatibility).

## Quick Start

You need Python 3.10 or later, [uv](https://docs.astral.sh/uv/), an Anthropic API key, and an MCP client that can send two custom HTTP headers with each request (see below). Install from source:

```bash
git clone https://github.com/shimo4228/akc-mcp.git
cd akc-mcp
uv venv .venv && source .venv/bin/activate
uv pip install -e ".[remote]"
```

The tools take your Anthropic API key from an `X-Anthropic-Key` HTTP header, so run the server over HTTP with a bearer token:

```bash
export MCP_AUTH_TOKEN=<a secret you choose>
akc-mcp --transport streamable-http --host 127.0.0.1 --port 8000
```

Point your MCP client at `http://127.0.0.1:8000/` with two headers: `Authorization: Bearer <MCP_AUTH_TOKEN>` and `X-Anthropic-Key: <your Anthropic API key>`. In Claude Code, for example, from another shell with both variables set (Claude Code keeps the two values in its MCP settings):

```bash
claude mcp add --transport http akc http://127.0.0.1:8000/ \
  --header "Authorization: Bearer $MCP_AUTH_TOKEN" \
  --header "X-Anthropic-Key: $ANTHROPIC_API_KEY"
```

A key's first tool or resource call creates its folder, `~/.config/akc/users/<SHA-256 of the key>/`, with a blank-slate identity, a one-line default constitution (`Be good.`) and an empty knowledge store, so there is nothing to set up by hand. A call to `record_episode` with `record_type: "insight"` answers `Recorded insight episode.`

### stdio (Claude Desktop and other local clients)

`akc-mcp` with no flags serves over stdio, so a local client such as Claude Desktop can launch it by command (give the full path to `.venv/bin/akc-mcp`). As of 2026-04-10, though, every tool called over stdio answers with a message that starts `Error: X-Anthropic-Key header required for this tool.`, because stdio has no way to pass that header. Use the HTTP route above.

## Tools

| Tool | Description |
|------|-------------|
| `record_episode` | Append an episode (post, comment, interaction, insight, …) to the daily log; writes to disk |
| `distill` | Distill recent episodes into learned patterns (dry-run by default; `write=true` saves them to `knowledge.json`) |
| `distill_identity` | Generate updated identity from knowledge patterns |
| `extract_insight` | Extract behavioral skills from learned patterns |
| `distill_rules` | Distill universal rules from skills in two LLM passes (free-form analysis, then structured rules) |
| `amend_constitution` | Generate constitutional amendment from experience |
| `skill_stocktake` | Audit skills for duplicates and quality issues |
| `rules_stocktake` | Audit rules for duplicates and quality issues |

## Resources

| URI | Description |
|-----|-------------|
| `akc://identity` | Current agent identity (does not resolve, see below) |
| `akc://knowledge` | Distilled knowledge patterns (does not resolve, see below) |
| `akc://constitution/{filename}` | Ethical constitution files |
| `akc://skills/{filename}` | Behavioral skill documents |
| `akc://rules/{filename}` | Behavioral rule documents |

As of the 2026-04-10 code, `akc://identity` and `akc://knowledge` answer `Unknown resource`: both functions take a context parameter, so the `mcp` library registers them as templates with no parameters, and the `mcp` version locked in `uv.lock` (1.27.0) does not match a read against such a template. The three templated resources work.

## Architecture

```
3-Layer Memory Architecture (compatible with contemplative-agent):

EpisodeLog (JSONL)  →  KnowledgeStore (JSON)  →  Identity (Markdown)
  append-only            distilled patterns         self-description
  daily logs             time-decay importance       evolved from knowledge
```

In text: episodes are appended to daily JSONL logs, distilled into weighted knowledge patterns, and the patterns are distilled again into a Markdown self-description.

## Data Format Compatibility

The file formats below are the same as [contemplative-agent](https://github.com/shimo4228/contemplative-agent)'s, so the knowledge, identity, skills and rules this server produces can be read and compared side by side with that agent's files. The directory layout differs: this server keeps each API key's files under `users/<SHA-256 of the key>/` inside the data directory, and later tools read only that folder. To carry the cycle on, the calling agent saves approved proposals there: skills as `skills/*.md`, rules as `rules/*.md`, the identity as `identity.md` and amendments in `constitution/`. `distill_rules` needs at least 3 files in `skills/`.

| File | Format |
|------|--------|
| `knowledge.json` | JSON array of `{pattern, distilled, importance, category, source}` (`source` is optional; `distill` sets it on every pattern it saves) |
| `identity.md` | Plain Markdown (no frontmatter) |
| `constitution/*.md` | Markdown with axiom names and principles |
| `skills/*.md` | Markdown with `# Title`, `## Problem`, `## Solution` |
| `rules/*.md` | Markdown with `# Title`, `**When:**`, `**Do:**`, `**Why:**` |
| `logs/*.jsonl` | JSONL with `{ts, type, data}` records |

## Configuration

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `AKC_HOME` | `~/.config/akc` | Data directory |
| `ANTHROPIC_API_KEY` | (unset) | Not needed: every tool takes the key from the `X-Anthropic-Key` header and ignores this variable |
| `MCP_TRANSPORT` | `stdio` | `stdio` or `streamable-http` (same as `--transport`) |
| `MCP_HOST` | `0.0.0.0` | Host to bind in HTTP mode; the default listens on every network interface, so pass `--host 127.0.0.1` to stay local |
| `MCP_PORT` | `8000` | Port in HTTP mode |
| `MCP_AUTH_TOKEN` | (unset) | Bearer token required on every HTTP request; set it for HTTP mode, because the server fails at startup in HTTP mode without it |

The model is fixed at `claude-sonnet-4-6` in `src/akc_mcp/llm.py`.

## Project history

Three design records (ADRs, in Japanese) written on 2026-04-10, the last day of code changes, set the current state. [ADR-0001](docs/adr/0001-no-autonomous-agents-on-social-networks.md) cancels running an autonomous agent on a social network and moves the project to local use, [ADR-0002](docs/adr/0002-bridge-layer-required-for-mcp-integration.md) settles that the calling agent places approved proposals itself, so no bridge layer is needed, and [ADR-0003](docs/adr/0003-stateless-cognitive-engine.md) decides to turn the server into a stateless engine that keeps no data, which is not implemented yet. A Fly.io deployment was removed under ADR-0001, while `fly.toml` and its workflow remain in the repo; the script that set up an Anthropic Managed Agent on the Moltbook social network (`scripts/managed_agent.py`) is kept as a reference implementation and is not run.

## Security

- Forbidden pattern validation (API keys, credentials, etc.) on recorded episode data, LLM output and stored knowledge patterns
- Path traversal prevention on template resources
- Episode logs are never exposed as MCP resources
- API key via the `X-Anthropic-Key` request header only (never stored in files)

## Academic Context

This project implements concepts from:

- Laukkonen, R., et al. (2025). *Contemplative Artificial Intelligence*. arXiv:2504.15125
- Laukkonen, R., Friston, K., & Chandaria, S. (2025). *A Beautiful Loop*. Neuroscience & Biobehavioral Reviews.

## More from the author

- **[contemplative-agent](https://github.com/shimo4228/contemplative-agent)**: the framework this server came from, an agent on a local LLM that proposes changes to its own constitution, identity and skills, with a person approving each change and the history public.
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: the cycle these tools are named after, with the reasoning behind each phase recorded as dated design decisions.
- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: the same cycle for Claude Code, installed as one plugin of skills plus an optional rules file, with no server to run.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with the five long-running projects and their DOIs.

## License

MIT. See [LICENSE](LICENSE).

<details>
<summary>For tools and AI assistants</summary>

akc-mcp (AKC MCP) is an alpha MCP server in Python that exposes the memory and self-improvement cycle of the contemplative-agent framework as MCP tools: recording episodes, distilling them into knowledge patterns, and turning those into an identity text, skills, rules, constitutional amendments and audits. It is for developers who want to try that cycle from another AI agent or MCP client.

It exists to re-implement contemplative-agent's cognitive layer as a standalone package that an agent can call over HTTP, using the Anthropic API instead of a local Ollama model. contemplative-agent keeps everything local by design ("security by absence"), so the MCP server, which depends on an outside LLM, lives in its own repository. Its data formats match contemplative-agent's, so that two agents with the same cognitive architecture in different environments can be compared.

Canonical facts: MIT license; Python 3.10 or later, built with hatchling, dependencies `mcp` and `anthropic` (plus `uvicorn` and `starlette` for the `remote` extra); version 0.1.0, classified Alpha in `pyproject.toml`, not published on PyPI; author Tatsuya Shimomoto (shimo4228). Status: alpha, curated by hand; the last code commit is dated 2026-04-10. Three ADRs dated 2026-04-10 record that running an autonomous agent on a social network was cancelled (ADR-0001), that no bridge layer is needed because the calling agent places results itself (ADR-0002), and that the server should become a stateless engine (ADR-0003, not implemented). Requirements: a paid Anthropic API key; the tools send their inputs to Anthropic's API using model `claude-sonnet-4-6`. The server offers 8 tools and 5 resources, of which `akc://identity` and `akc://knowledge` answer `Unknown resource` as of the 2026-04-10 code with the `mcp` version locked in `uv.lock` (1.27.0); it stores data under `AKC_HOME` (default `~/.config/akc`).

Example: in the code as of 2026-04-10, tools receive the API key only from an `X-Anthropic-Key` HTTP header, which is read only when the server runs with `--transport streamable-http` and `MCP_AUTH_TOKEN` set. With those, a client sending `Authorization: Bearer <token>` and `X-Anthropic-Key` that calls `record_episode` with `record_type: "insight"` gets `Recorded insight episode.`, and the episode lands in `~/.config/akc/users/<SHA-256 of the key>/logs/<date>.jsonl`. Over stdio every tool returns a message that starts `Error: X-Anthropic-Key header required for this tool.`

Links: [docs/adr/](docs/adr/) holds the three design records (in Japanese); [src/akc_mcp/server.py](src/akc_mcp/server.py) defines the tools and resources. The parent framework is [contemplative-agent](https://github.com/shimo4228/contemplative-agent), concept DOI [10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118); the cycle is the [Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle), concept DOI [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726). The author's hub is [shimo4228/shimo4228](https://github.com/shimo4228/shimo4228).

</details>
