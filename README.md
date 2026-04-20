# One config, every MCP server: a tour of 1Server and the mcp-engine

If you've spent any real time with the Model Context Protocol, you already know the shape of the problem. You find a cool MCP server. You paste a JSON block into your Claude Desktop or Cursor config. You restart the client. Great.

Then you find another. And another. Now you have five `npx` commands, three environment variables, two OAuth flows you don't fully remember setting up, and a config file that needs a full client restart every time you touch it.

That's the problem [1Server](https://1server.ai) was built to solve — and this post is the end-to-end tour.

By the end, you'll understand:

- What the [1Server marketplace](https://1server.ai/marketplace) is and why curation matters
- How the flagship [`1server-mcp-engine`](https://1server.ai/setup) turns dozens of servers into one clean connection
- The complete workflow, from sign-up to installing and managing servers from inside your AI chat
- How publishers can ship their own servers to the marketplace

Grab a coffee. This one's detailed on purpose.

---

## The problem: MCP is amazing, config files are not

MCP is one of the best things to happen to AI assistants. It finally gives Claude, Cursor, VS Code, and friends a standard way to talk to the rest of your stack — GitHub, Slack, your database, your filesystem, your company wiki.

The trouble is the plumbing.

**Without an engine, a real MCP setup looks like this:**

```json
{
  "mcpServers": {
    "memory":     { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-memory"] },
    "github":     { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"], "env": { "GITHUB_TOKEN": "ghp_..." } },
    "filesystem": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me/code"] },
    "slack":      { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-slack"], "env": { "SLACK_TOKEN": "xoxb-..." } },
    "brave":      { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-brave-search"], "env": { "BRAVE_API_KEY": "..." } }
  }
}
```

Every new server is another block. Every token is in plaintext on disk. Every change is a client restart. And if one server crashes mid-session, you'll find out the hard way — when your assistant stops being able to search your inbox.

This is fine at two servers. It's miserable at ten.

---

## The 1Server ecosystem, in one picture

Two things work together:

1. **The [1Server marketplace](https://1server.ai/marketplace)** — a curated catalog of MCP servers. Browse, search, install, configure. Secrets go into an encrypted vault, not your config file.
2. **[`1server-mcp-engine`](https://1server.ai/setup)** — a single MCP server your client connects to. It reads your marketplace installs, spawns the underlying servers, aggregates their tools, and exposes the whole thing through one connection.

Put plainly: **the marketplace is how you choose servers. The engine is how you run them.**

Your MCP client never sees the mess. It sees one server called `1server`, with every tool namespaced and ready to use.

---

## Why curation actually matters

The MCP ecosystem is exploding, which is great — and also means a growing pile of servers of varying quality, maintenance levels, and security postures. A curated marketplace is not a nice-to-have here; it's the difference between "I installed a thing" and "I installed a thing I trust."

On the [1Server marketplace](https://1server.ai/marketplace) you get:

- **Vetted servers** with clear descriptions, required credentials, and install counts
- **Search and categories** so you're not digging through a random registry
- **Credential hints up front** — you know before you install that a server needs, say, a `SLACK_TOKEN`
- **Publisher pages** so you can see who's behind a server and what else they maintain

And because installs are tied to your account — not to a JSON file on a specific laptop — your setup follows you. New machine? Connect the engine, your servers are there.

---

## The end-to-end workflow

Let's walk through exactly what it looks like to go from zero to "my AI assistant can manage its own tools."

### Step 1 — Create an account

Head to [1server.ai](https://1server.ai) and sign in. You can use Google, GitHub, or email. This is your control plane: installs, secrets, API keys, and (if you're a publisher) your own servers all live here.

### Step 2 — Browse the marketplace

Open the [marketplace](https://1server.ai/marketplace) and pick a server. Say you want GitHub integration. You'll see:

- What tools it exposes
- What credentials it needs (for GitHub: a personal access token or OAuth)
- Install count and publisher
- Links to source and docs

Hit **Install**. If the server needs a secret, 1Server will prompt you — either paste the value (it's encrypted at rest on the backend, AES-256) or pick an existing secret from your vault. Values never land in a config file on your laptop.

### Step 3 — Generate an API key

From your [dashboard](https://1server.ai/dashboard/api-keys), create an API key. This is how the engine authenticates back to 1Server to fetch your installs and pull secrets when it needs them.

Treat this key like any other production credential. You can rotate or revoke it any time from the same page.

### Step 4 — Point your MCP client at the engine

This is the part that used to be a five-server JSON block. Now it's this:

```json
{
  "mcpServers": {
    "1server": {
      "command": "npx",
      "args": ["-y", "1server-mcp-engine"],
      "env": {
        "ONESERVER_API_KEY": "your-api-key"
      }
    }
  }
}
```

One entry. Works in Claude Desktop, Cursor, VS Code, and any MCP-compatible client. The full setup walkthrough — including per-client paths — lives at [1server.ai/setup](https://1server.ai/setup).

Restart your client *once*. You're done restarting.

### Step 5 — Watch the engine come up

On first launch, the engine:

1. Authenticates with your API key
2. Pulls your list of installed servers from 1Server
3. Decrypts the secrets it needs at runtime
4. Starts accepting client connections *immediately* — no waiting for servers to boot
5. Connects to your servers in parallel (up to 3 at a time), with fast servers online in seconds and slower ones streaming in behind them

That last point is a small detail with a big consequence. Your AI assistant isn't stuck staring at a spinner while a sluggish server finishes handshaking. The engine is already talking. Tools appear as they come online.

### Step 6 — Use your tools

Every tool is namespaced by server name — `github.create_issue`, `slack.post_message`, `filesystem.read_file` — so there are no collisions even if two servers define a similarly-named tool. Your client sees a single, coherent toolset.

And now for the part that makes people grin.

---

## Manage your servers from inside the chat

The engine exposes a small set of meta-tools — prefixed with `1server_` — that let your AI assistant manage its own inventory. No tab-switching. No config editing. Just ask.

| Tool | What it does |
|------|-------------|
| `1server_list_servers` | See everything currently installed |
| `1server_search_servers` | Browse the marketplace from chat |
| `1server_install_server` | Install (or update) a server |
| `1server_uninstall_server` | Permanently remove a server |
| `1server_toggle_server` | Disable without losing config |
| `1server_restart_server` | Kick an unhealthy server |
| `1server_health` | Real-time status for every server |
| `1server_list_secrets` | Show saved credentials by name |

In practice, a session looks like this:

> **You:** Find me a Slack MCP server.
> **Claude:** Found **slack-mcp** by Slack — 2,340 installs. Needs a `SLACK_TOKEN`. Want me to install it?
>
> **You:** Do I have a Slack token saved?
> **Claude:** You have three secrets: GitHub-Token, Slack-Bot-Token, Bright-Data-Key. I can use Slack-Bot-Token.
>
> **You:** Install with that one.
> **Claude:** Done. slack-mcp is live with your saved secret. Twelve new tools available.

No restart. No config file. No leaving the conversation.

This is where the engine earns its keep — it collapses the gap between *wanting a capability* and *having it*.

---

## Hot-reload: the feature you don't know you want until you have it

Every MCP tutorial online says some variation of "edit the config, then restart the client." The engine deletes that step.

When the marketplace says your installs changed — you installed a server, toggled one off, rotated a secret — the engine notices and reacts in place:

- **New server?** It spins it up and registers its tools.
- **Removed server?** It tears it down and unregisters the tools.
- **Toggled off?** It disconnects cleanly but keeps the config.
- **Secret changed?** It reconnects with the new value.

Your MCP client stays connected the entire time. Your AI assistant notices new tools appearing mid-conversation. This is what "config changes apply instantly" actually means.

---

## Health monitoring, because servers crash

Every connected server is monitored continuously. If one dies — process crash, network blip, upstream API meltdown — the engine:

1. Marks it unhealthy
2. Attempts to reconnect with exponential backoff (up to 10 tries)
3. Surfaces the state via `1server_health` so your assistant can tell you

Ask the engine "are all my servers healthy?" and you'll get a proper answer: which are up, which are degraded, uptime, failure counts, tool counts. If something's wedged, `1server_restart_server` forces a clean reconnect without touching any other server.

For anything long-lived — a full coding session, a research task that spans hours — this is the difference between "my tools just silently stopped working" and "my tools healed themselves."

---

## Encrypted secrets, not plaintext config

Secrets live in an AES-256 encrypted vault on the 1Server backend and are referenced by ID when you install a server. Your config file doesn't contain them. Your logs don't contain them. Server processes receive them at spawn time and nowhere else.

You can manage the vault from the [secrets dashboard](https://1server.ai/dashboard/secrets): add, rotate, delete. Updates propagate to the engine on the next reconnect — which, thanks to hot-reload, happens without a restart.

---

## OAuth, handled

Plenty of MCP servers authenticate with OAuth rather than static tokens. The engine handles the full flow: it opens the provider's auth page, captures the callback, and persists tokens at `~/.oneserver-mcp/auth/` so you don't re-authenticate every session. HTTP-based MCP servers connect remotely; stdio servers run as local processes. You don't have to care which is which — the engine abstracts the transport.

---

## For publishers: ship your own server to the marketplace

If you *build* MCP servers (or want to), the [publish flow](https://1server.ai/publish) is the other half of the story. You can:

- Submit a server with metadata, required credentials, and install instructions
- Track installs and usage from your [dashboard](https://1server.ai/dashboard/published)
- Push updates that reach every installed user on their next engine reconnect
- Get your server in front of an audience that's actively searching for capabilities — not scrolling a generic registry

Curation goes both ways: users get servers they can trust, and publishers get a real distribution channel instead of shouting into the npm void.

---

## A mental model for when to use what

- **Just getting started?** Open the [marketplace](https://1server.ai/marketplace), install two or three servers, follow the [setup guide](https://1server.ai/setup), and get one-config MCP running.
- **Already have a tangled config?** Move the servers you use into 1Server, point your client at the engine, and delete the old JSON. Your sessions get hot-reload, health monitoring, and in-chat management for free.
- **Building your own server?** Publish it at [1server.ai/publish](https://1server.ai/publish) and reach users already set up to install with one click.

---

## The short version

MCP gives AI assistants real capabilities. 1Server makes those capabilities manageable.

- **[The marketplace](https://1server.ai/marketplace)** is your curated catalog.
- **[The engine](https://1server.ai/setup)** is the one connection that runs every server behind the scenes, with hot-reload, health checks, encrypted secrets, and in-chat management.
- **[Your dashboard](https://1server.ai/dashboard/installations)** is where it all ties together.

You stop editing JSON. You stop restarting your client. You stop babysitting processes. Your AI assistant even learns to manage its own tools.

One config. Every server. That's the whole pitch.

**Ready to try it?**

- Create an account → [1server.ai](https://1server.ai)
- Browse servers → [1server.ai/marketplace](https://1server.ai/marketplace)
- Install the engine → [1server.ai/setup](https://1server.ai/setup)
- Publish your own → [1server.ai/publish](https://1server.ai/publish)
- More posts → [1server.ai/blog](https://1server.ai/blog)

---

*Built by the team at [1Server](https://1server.ai). Questions or feedback? We read everything — start at [1server.ai](https://1server.ai) and follow the contact link in the footer.*
