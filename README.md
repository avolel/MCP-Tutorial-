# MCP Tutorial

A practical explainer on the Model Context Protocol (MCP) written for mid to senior developers. It skips the marketing and gets to the architecture, the message flow, a working server example, and the security tradeoffs you actually need to think about.

## What's in here

| File | What it is |
|------|------------|
| `mcp-tutorial.md` | The full tutorial. Start here. |
| `README.md` | This file. |

## Who this is for

People who already build software and have probably wired an LLM up to at least one external system the hard way. If you have never touched function calling or an agent loop, the tutorial will still make sense, but it assumes you are comfortable with client-server architecture, JSON, and reading a code sample without hand-holding.

## What you'll walk away with

- A clear mental model of the three MCP roles: host, client, and server, and why the server never talks to the model directly.
- The three things a server can expose: tools, resources, and prompts, plus the two reverse features (sampling and elicitation) that make agentic workflows work.
- How a request actually flows from a user question to a tool call and back.
- The two transports, stdio and Streamable HTTP, and when to reach for each.
- A minimal working server in Python you can adapt.
- The security considerations that matter once a model has real access to real systems.

## How to read it

Top to bottom the first time. The sections build on each other, and the architecture section sets up vocabulary the rest of the tutorial leans on. After that it works fine as a reference you jump around in.

If you only have five minutes, read the "Mental model" section and the "Short version" at the end.

## Running the example

The tutorial includes a small server built on the Python SDK. To try it:

```bash
pip install mcp
python inventory_server.py
```

Then register it with a compatible host (Claude Desktop, Cursor, or similar) using the config snippet in the tutorial, and restart the host so it picks up the new server.

## A note on the source

This tutorial was built from an infographic on MCP and then checked against the current protocol details, since infographics tend to simplify and occasionally bake in implementation choices that aren't part of the standard. Where the graphic and the actual spec disagreed, the tutorial follows the spec and says so.

## Going deeper

- Official specification and SEPs: `modelcontextprotocol.io/specification/latest`
- Spec source, schema, and proposals: `github.com/modelcontextprotocol/modelcontextprotocol`
- TypeScript SDK: `github.com/modelcontextprotocol/typescript-sdk`
- Python SDK: `github.com/modelcontextprotocol/python-sdk`
