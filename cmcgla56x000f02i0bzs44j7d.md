---
title: "I Made My Homelab Talk to Me Using Claude and FastMCP"
datePublished: Sat Jun 28 2025 18:43:22 GMT+0000 (Coordinated Universal Time)
cuid: cmcgla56x000f02i0bzs44j7d
slug: i-made-my-homelab-talk-to-me-using-claude-and-fastmcp
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1751136102557/55969db1-9daf-457d-a290-fb7ab69eadd0.webp
tags: claudeai, model-context-protocol, mcp-server, fastmcp

---

Most of us build homelabs to tinker, automate, and take control of our infrastructure. But somewhere between Docker containers, backups, and uptime monitoring, it becomes a lot to keep track of. I didn’t want to SSH into my server every time I needed a quick answer like “How much space is left on my drive?” or “Is Tailscale still running?”

So I built something better. Now I just **ask Claude**, and it tells me.

This post walks through how I wired up my homelab using [FastMCP](https://gofastmcp.com) and [Claude Desktop](https://claude.ai), letting me run system queries through natural language and get intelligent responses from my own infrastructure.

---

## What Is FastMCP?

If you're not familiar with it, **FastMCP** is a Python framework that lets you expose tools, resources, and prompts to LLMs via the Model Context Protocol (MCP). That means you can define Python functions, decorate them with `@tool`, and suddenly they’re callable from Claude, ChatGPT, or even your own HTTP clients.

Think of it as:

* `FastAPI` for LLMs
    
* but typed
    
* and purpose-built for multi-tool interactions
    

---

## What I Wanted to Do

Here’s what I was aiming for:

* Ask Claude questions like “What’s the disk usage?” or “Are my containers healthy?”
    
* Run system-level commands via Python securely
    
* Keep everything running locally or over Tailscale, no public exposure
    
* Build it once and just keep extending it
    

---

## Step 1: Writing an MCP Server

This is where the magic starts. Here’s a basic MCP server using FastMCP:

```python
# homelab_server.py

from fastmcp.server import Server
from fastmcp.tools import tool
import psutil, subprocess

class HomelabServer(Server):
    @tool
    def disk_usage(self) -> str:
        usage = psutil.disk_usage('/')
        return f"{usage.percent}% used — {usage.used // (1024**3)}GB of {usage.total // (1024**3)}GB"

    @tool
    def tailscale_status(self) -> str:
        result = subprocess.run(['tailscale', 'status'], capture_output=True, text=True)
        return result.stdout.strip()

    @tool
    def docker_containers(self) -> str:
        result = subprocess.run(['docker', 'ps', '--format', '{{.Names}}: {{.Status}}'], capture_output=True, text=True)
        return result.stdout.strip()

server = HomelabServer()
```

You can run this locally with:

```bash
uvicorn homelab_server:server --port 7531
```

Now your homelab has a voice.

---

## Step 2: Making It Reachable

You don’t need to open ports to the world. I already run **Tailscale**, so I just connected my laptop and server to the same private network. That gave me a private IP like `100.x.x.x`, and I used that to point Claude Desktop to my MCP server.

---

## Step 3: Connecting Claude Desktop

Claude Desktop (with plugin support) makes this super easy.

1. Open Claude → `Plugins` → `Add MCP Server`
    
2. Add your MCP server URL  
    Example: [`http://100.x.x.x:7531`](http://100.x.x.x:7531)
    
3. Claude will auto-detect the available tools
    

Now I can type:

> “Call the `disk_usage` tool on the homelab server”

Or even just:

> “How much disk space do I have left?”

Claude figures out the right tool to call, runs it, and replies with a summary.

---

## Bonus: Chaining Output with Prompts

FastMCP also supports *prompt templates*, which means I can wrap raw command output in a summarization prompt and have Claude generate human-friendly summaries, great for things like:

* Failed systemd services
    
* Health reports
    
* ZFS snapshots or Btrfs status
    

You can even create tools that return JSON and let Claude reason over it.

---

## Why This Is Fun (and Actually Useful)

This setup saves me time **and** gives me a more natural way to interact with my homelab. I don’t have to mentally context-switch into "sysadmin mode" every time I want to check logs or disk stats.

It also opens the door to more advanced use cases:

* Triggering Ansible playbooks via tools
    
* Running backups and summarizing results
    
* Fetching metrics from Grafana or Prometheus
    
* Acting as a gateway for multiple machines (via [server composition](https://gofastmcp.com/servers/composition.md))
    

---

## Future Plans

Here’s what I want to add next:

* A `/status` resource that returns full server health in JSON
    
* A prompt-based tool that summarizes `journalctl` logs
    
* A Discord webhook client that uses FastMCP to send notifications
    
* Claude-triggered toolchains: ask one question, get multiple tools executed in sequence
    

---

## Final Thoughts

This isn’t just a cool hack. It’s the start of something much more interactive and intuitive. We’re entering a world where large language models can be more than chatbots; they can be *interfaces* to real systems.

If you’ve got a homelab and a few Python skills, I’d highly recommend trying out FastMCP. You’ll be surprised how far a simple tool can go when you give it a little context.