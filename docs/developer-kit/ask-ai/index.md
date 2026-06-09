---
title: Ask AI
---

# Ask AI

Read-only MCP server for semantic search over BNB Chain documentation, BEPs, blogs, and FAQs. Connect from VS Code, Cursor, JetBrains, or any MCP client — no private keys required.

**Supported:** BSC documentation corpus (read-only).

## Repository

[https://github.com/bnb-chain/bnb-chain.github.io](https://github.com/bnb-chain/bnb-chain.github.io) — docs and MCP endpoint configuration

## Documentation

[DeepWiki](https://deepwiki.com/) — *link pending*

Endpoint: `https://api.superintern.ai/agent/async/mcp/mcp`

## Demo

Add to your IDE MCP config:

```json
{
  "mcpServers": {
    "bnbchain-askai-mcp": {
      "url": "https://api.superintern.ai/agent/async/mcp/mcp"
    }
  }
}
```
