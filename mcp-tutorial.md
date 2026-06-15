# What MCP Is and How It Actually Works

*A working guide for developers who already know how to build software and just want the real picture of the Model Context Protocol.*

## Why this exists

If you have wired an LLM up to a database, a search index, and a couple of internal APIs, you already know the pain. Every integration is bespoke. You write glue code to describe each tool to the model, you hand roll the auth, you parse the responses, and then you do the whole thing again for the next model or the next product surface. Swap GPT for Claude and half of it breaks. Add a fourth tool and the prompt turns into a swamp.

The Model Context Protocol (MCP) is Anthropic's answer to that mess, released in late 2024 and since adopted by OpenAI, Google, and most of the serious agent tooling. It is an open standard for connecting AI applications to external tools and data. The usual one liner is that MCP is a USB-C port for AI: build the connector once, and any compliant application can plug into it. That analogy is fine as far as it goes, but it undersells the interesting part, which is what the protocol standardizes and what it deliberately leaves alone.

One thing worth saying up front, because it trips people up: MCP does not replace function calling. Function calling is still the mechanism by which a model decides to invoke a tool. MCP wraps that mechanism in a discoverable, portable protocol so the tool definitions, capabilities, and auth flow look the same no matter which model or client is on the other end. Same plumbing, standardized fittings.

## The mental model

Three roles, and getting them straight saves a lot of confusion later.

**Host.** The AI application the user actually interacts with. Claude Desktop, Cursor, a VS Code extension with a copilot, or your own agent runtime. The host owns the model, manages the conversation, and decides which connections to open.

**Client.** A connector that lives inside the host. The important detail is that the relationship is one to one: the host spins up exactly one client for each server it talks to. Connect to five servers and the host is running five clients. Each client holds a single stateful session with its server and handles the message translation, error handling, timeouts, and reconnects for that one connection.

**Server.** A lightweight process that wraps a specific capability and exposes it through the protocol. A Postgres server, a GitHub server, a filesystem server. The server takes structured requests from a client, does the actual work against the underlying system, and returns formatted results.

Here is the part people miss on the first read. **The server never talks to the model directly.** A Postgres MCP server has no idea whether the model on the other end is Claude or GPT or a Llama you are running on your own box. It receives a request, runs SQL, returns rows. The host and client sit in between and do the model-specific work. That separation is the whole point, and it is why a server you write today keeps working when you switch models next year.

The infographic frames this as MCP Host on the left, the services in the middle, and external tools on the right. That maps cleanly:

- The **AI Agent** and its **MCP Clients** are the host and its client connections.
- The **Database / Email / Search / Chat Service** boxes are MCP servers.
- The **External Tools** column is the real systems each server wraps: the actual database, the Gmail API, the search backend, Slack.

The "MCP Protocol" labels on the arrows between client and server are doing the real work in that diagram. That arrow is the standardized boundary. Everything to the left of it is model and host concern. Everything to the right of a server is whatever bespoke logic that server needs to talk to its backend.

## What a server can expose

A server is not just a bag of functions. The spec defines three primitives, and knowing the difference shapes how you design one.

**Tools** are functions the model can invoke to do something. Search files, run a query, send a message, open a pull request. These are the actions. The model decides when to call them based on the tool's description and schema, which the server advertises.

**Resources** are data the model can read. File contents, a database record, a config blob, a document. Think of resources as the read side: the model pulls context in rather than triggering an effect. Recent versions of the spec attach freshness metadata to list and read results, so a client knows how long a listing stays valid and whether it is safe to cache across users, instead of holding a long-lived stream open just to notice a change.

**Prompts** are reusable templates the server offers to the host, often surfaced to the user as slash commands or quick actions. A "summarize this PR" prompt that already knows how to fetch the diff and frame the request, for example. They let a server ship not just capability but a sensible way to use it.

There are also two features that run in the other direction, server to client, and they are what make genuinely agentic workflows possible:

**Sampling** lets a server ask the host's model to generate text mid-task. A migration server that detects a schema change can ask the model to reason about the blast radius before doing anything.

**Elicitation** lets a server request direct input from the user. The same migration server, having decided a change is risky, can pause and ask a human to confirm. Put sampling and elicitation together and you get clean human-in-the-loop behavior without the server needing its own model or its own UI.

## How messages move: transports

Under the hood, MCP speaks JSON-RPC 2.0. Every request, response, and notification is a JSON-RPC message. That choice matters because it is boring and well understood, which is exactly what you want in a protocol meant to outlive any particular model.

There are two transports, and you pick based on where the server runs.

**stdio.** The server runs as a local subprocess and the client talks to it over standard input and output. This is the right choice for local tools and for development. No network, no ports, no auth dance. The filesystem server reading files on your laptop runs this way.

**Streamable HTTP.** The remote transport, used for servers deployed as services. It replaced the older HTTP plus SSE approach as the recommended option, and it is built to play nicely with ordinary infrastructure. A remote server can now sit behind a plain load balancer rather than needing sticky sessions and a shared session store. Remote servers are where authentication enters the picture, and the protocol leans on OAuth 2.1 for that.

The rule of thumb: stdio while you are building and for anything local, Streamable HTTP once the server lives somewhere other than the user's machine.

## Walking through a real request

The infographic's sequence diagram is useful but it hardcodes specific tools (LangGraph as the orchestrator, OpenAI GPT as the model). Those are one team's implementation choices, not part of MCP. Here is the flow described in terms of the roles that actually matter, which keeps it true regardless of your stack.

1. **User asks something.** "What were our top five accounts by revenue last quarter, and email the list to the finance channel?" The host receives the query.

2. **Host gathers available capabilities.** Before or during the turn, the host's clients have asked their servers what they offer. The host now knows it has a Postgres server with a query tool and a Slack server with a post-message tool, along with their schemas.

3. **Model picks a tool.** The model, seeing the query and the available tools, decides to call the database query tool with a specific SQL-shaped argument. This is ordinary function calling. The host routes that decision to the right client.

4. **Client calls the server.** The client serializes the call into a JSON-RPC request and sends it to the Postgres server over its transport.

5. **Server does the work.** The server runs the query against the real database, which is the bespoke part that only this server knows how to do, and returns the rows.

6. **Result goes back to the model.** The client hands the tool result back to the host, which feeds it to the model as context.

7. **Repeat as needed.** The model now has the revenue numbers and decides to call the Slack server's post-message tool. Same path: host to client to server to Slack and back.

8. **Final answer.** With both tool results in hand, the model composes a reply, and the host returns it to the user.

The thing to notice is how little of this is MCP specific. Steps 3 and 6 are normal model behavior. The MCP part is steps 2, 4, and 5: the standardized way capabilities get advertised and invoked across a uniform boundary. That boundary is the entire value proposition.

## Building a minimal server

Talk is cheap. Here is a small server using the Python SDK that exposes one tool and one resource. The SDK handles the JSON-RPC framing so you write business logic, not protocol code.

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("inventory")

# A tool: the model can call this to take an action / fetch computed data.
@mcp.tool()
def check_stock(sku: str) -> dict:
    """Return current stock level and warehouse for a given SKU."""
    row = db.query_one(
        "SELECT quantity, warehouse FROM inventory WHERE sku = %s",
        (sku,),
    )
    if row is None:
        return {"sku": sku, "found": False}
    return {"sku": sku, "found": True, "quantity": row.quantity, "warehouse": row.warehouse}

# A resource: read-only context the model can pull in.
@mcp.resource("inventory://policy/restock")
def restock_policy() -> str:
    """The current restocking policy document."""
    return open("docs/restock_policy.md").read()

if __name__ == "__main__":
    mcp.run()  # defaults to stdio transport
```

A few things to call out, because they are easy to get wrong:

The docstring on `check_stock` is not decoration. It becomes the tool description the model reads to decide whether and how to call the tool. Write it like product copy aimed at the model. Vague descriptions produce a model that calls the wrong tool or fills arguments badly.

Type hints drive the input schema. The SDK turns the `sku: str` annotation into a JSON schema that the client advertises and validates against. Lean on this. A well typed signature is a well specified tool.

Return structured data when you can. A dict the model can parse beats a sentence it has to interpret.

To run the same server remotely, you switch the transport rather than rewriting the logic:

```python
if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

That is roughly the whole appeal in two lines. The capability is written once. Local versus remote is a deployment decision, not a rewrite.

## Connecting it to a host

For local development with a host like Claude Desktop or Cursor, you register the server in the host's config so it knows to launch the subprocess. The shape is consistent across hosts even if the file location differs:

```json
{
  "mcpServers": {
    "inventory": {
      "command": "python",
      "args": ["/path/to/inventory_server.py"]
    }
  }
}
```

Restart the host, and the inventory tools show up alongside everything else. The same server, unchanged, would work in any other compliant host. That portability is not a nice side effect. It is the reason to structure your integrations as MCP servers from day one rather than as inline tool code glued to one product.

## Why this matters, beyond the hype

The infographic lists four reasons, and they hold up, so here they are with the reasoning filled in.

**It is open and standard.** One agreed way for AI to reach tools and data. The protocol was donated to the Linux Foundation's Agentic AI Foundation in late 2025, which matters if you are making a multi-year bet, because it means no single vendor owns the thing your architecture depends on.

**Integrations get easy.** Write a server once and every compliant client can use it. The ecosystem has gone from novelty to infrastructure fast, with community servers for GitHub, Slack, Postgres, Stripe, Figma, Docker, Kubernetes, and a couple hundred others. A lot of the time the integration you need already exists.

**Development gets faster.** You stop rebuilding the same connector for each model and each surface. Reuse beats rewriting, and the standard connection method means less custom code to maintain and fewer places for it to rot.

**The AI gets more capable.** A model with no tools is a very articulate box that cannot do anything. Give it real access to live data and real systems through a uniform interface and it can actually act. That is the difference between a chatbot and an agent.

## Security, because someone has to say it

The same property that makes MCP powerful, giving models real access to real systems, is the property that should make you careful.

The one-to-one client-to-server isolation is a deliberate safety feature. A compromised or misbehaving server cannot reach across into another server's connection. Keep that boundary clean and do not invent shortcuts that route one server's traffic through another.

For remote servers, treat OAuth 2.1 as table stakes and scope tokens tightly. A server that wraps your production database should expose the narrowest set of operations the use case needs, not a generic "run any SQL" tool that a confused or manipulated model can turn into a foot gun.

Be deliberate about which servers a host is allowed to connect to, and treat tool descriptions and prompts coming from third-party servers as untrusted input, because the model reads them. A malicious server can try to influence the model through the very text it advertises. Sandbox, review, and prefer servers you or your org control for anything sensitive.

## Where it is heading

A few directions are worth keeping an eye on if you are building now. Server discovery is maturing toward registries and standardized metadata so hosts can find, install, and vet servers without hand-configuring paths. Servers are gaining the ability to ship interactive UI that hosts render in a sandbox, so a tool can present a real interface rather than only returning text. And the bundling of domain knowledge alongside tools means a server can teach a model how to use it well, not just expose the buttons.

None of that changes the core you just learned. Host, client, server. JSON-RPC over stdio or HTTP. Tools, resources, prompts. Get those right and the rest is detail.

## The short version

MCP standardizes the boundary between AI applications and the tools they use. A host runs one client per server. Servers expose tools, resources, and prompts over JSON-RPC, locally through stdio or remotely through Streamable HTTP, and they never touch the model directly. You write a server once and any compliant client can use it. That is the whole idea, and it is a good one.
