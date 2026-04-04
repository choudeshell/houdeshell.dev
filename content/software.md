+++
categories = []
date = "2020-06-01"
modified = "2025-08-10"
description = ""
keywords = []
title = "Software that I <span style='color:var(--accent-bright);font-size:1.3em'>❤</span>"
menu = "main"
weight = 1
section = "software"

+++

I bounce between Mac, Windows, and Linux daily — and honestly, the tools matter more than the OS at this point. Great software is great software, and I've been lucky enough to build a career on top of things that other people built well.

This is the stuff I actually reach for. Not a "best of" list, not sponsored, not comprehensive. Just software that makes me better at what I do — or at least makes the work more fun.

<style>
.sw-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 16px;
  margin: 1.5em 0;
}
.sw-card {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 1.2em;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}
.sw-card:hover {
  border-color: var(--accent);
  box-shadow: 0 0 16px rgba(57, 255, 110, 0.08);
}
.sw-card h3 {
  margin: 0 0 0.3em !important;
  font-size: 1em !important;
  border: none !important;
  padding: 0 !important;
}
.sw-card h3 a {
  text-decoration: none;
}
.sw-card p {
  margin: 0;
  font-size: 0.85em;
  color: var(--text-secondary);
  line-height: 1.5;
}
.sw-card .sw-tag {
  display: inline-block;
  font-size: 0.7em;
  color: var(--accent);
  border: 1px solid var(--accent);
  border-radius: 3px;
  padding: 1px 6px;
  margin-bottom: 0.5em;
  opacity: 0.7;
}
</style>

# Software Development

<div class="sw-grid">

<div class="sw-card">
<span class="sw-tag">IDE</span>

### [JetBrains Rider](https://www.jetbrains.com/rider/)
Cross-platform .NET IDE. Fast, smart, and doesn't need Visual Studio's weight to get the job done.
</div>

<div class="sw-card">
<span class="sw-tag">IDE</span>

### [Visual Studio](https://visualstudio.microsoft.com/)
The OG. Heavy, but when you need the full .NET debugging experience, nothing else comes close.
</div>

<div class="sw-card">
<span class="sw-tag">editor</span>

### [VS Code](https://code.visualstudio.com/)
Extension ecosystem is unmatched. Somehow Electron done right.
</div>

<div class="sw-card">
<span class="sw-tag">editor</span>

### [Neovim](https://neovim.io/)
Vim reborn. Lua config, LSP native, and a plugin ecosystem that won't quit. `:wq` is a lifestyle.
</div>

<div class="sw-card">
<span class="sw-tag">AI</span>

### [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
AI pair programmer in your terminal. It's writing this page right now.
</div>

<div class="sw-card">
<span class="sw-tag">version control</span>

### [Git](https://git-scm.com/)
The version control system that won. Love it or hate it, you can't ship without it.
</div>

<div class="sw-card">
<span class="sw-tag">version control</span>

### [lazygit](https://github.com/jesseduffield/lazygit)
Terminal UI for git that makes interactive rebases feel like cheating.
</div>

<div class="sw-card">
<span class="sw-tag">JSON</span>

### [jq](https://jqlang.github.io/jq/)
`sed` for JSON. Once you learn the syntax, you'll pipe everything through it.
</div>

<div class="sw-card">
<span class="sw-tag">database</span>

### [JetBrains DataGrip](https://www.jetbrains.com/datagrip/)
SQL IDE that actually understands your schema. Autocomplete that works across joins.
</div>

<div class="sw-card">
<span class="sw-tag">database</span>

### [DBeaver](https://dbeaver.io/)
Universal database tool that actually works. Connect to anything, query everything.
</div>

<div class="sw-card">
<span class="sw-tag">database</span>

### [SSMS](https://learn.microsoft.com/en-us/sql/ssms/)
Microsoft SQL Management Studio. If you're in SQL Server land, you already know.
</div>

<div class="sw-card">
<span class="sw-tag">database</span>

### [PostgreSQL](https://www.postgresql.org/)
The database that keeps getting better. Extensions, JSON support, and rock-solid reliability.
</div>

<div class="sw-card">
<span class="sw-tag">containers</span>

### [Docker](https://www.docker.com/)
"Works on my machine" became "works on every machine." Changed how we ship software.
</div>

</div>

# Infrastructure

<div class="sw-grid">

<div class="sw-card">
<span class="sw-tag">orchestration</span>

### [Kubernetes](https://kubernetes.io/)
Container orchestration that makes you mass produce YAML for a living.
</div>

<div class="sw-card">
<span class="sw-tag">orchestration</span>

### [K3s](https://k3s.io/)
All of Kubernetes in a single binary. Perfect for homelab, edge, and production deployments.
</div>

<div class="sw-card">
<span class="sw-tag">IaC</span>

### [Terraform](https://www.terraform.io/) / [OpenTofu](https://opentofu.org/)
Infrastructure as code. OpenTofu if you prefer the open-source fork without the license drama.
</div>

<div class="sw-card">
<span class="sw-tag">networking</span>

### [Tailscale](https://tailscale.com/)
WireGuard-based mesh VPN that just works. Connect everything without opening a single port.
</div>

<div class="sw-card">
<span class="sw-tag">observability</span>

### [Grafana](https://grafana.com/)
Dashboards for everything. Pairs with Prometheus, Loki, Tempo — the whole observability stack.
</div>

<div class="sw-card">
<span class="sw-tag">Kubernetes</span>

### [k9s](https://k9scli.io/)
Terminal UI for Kubernetes. Makes cluster management feel like a video game. Way faster than raw `kubectl`.
</div>

<div class="sw-card">
<span class="sw-tag">Kubernetes</span>

### [Lens](https://k8slens.dev/)
The Kubernetes IDE. When you want to point and click your way through a cluster without shame.
</div>

<div class="sw-card">
<span class="sw-tag">it's complicated</span>

### [Cloudflare](https://www.cloudflare.com/)
You love them. You hate them. Your DNS is already there. Their edge network is everywhere, their pricing is unbeatable, and their product sprawl is terrifying. Stockholm syndrome as a service.
</div>

</div>

# Shell & Terminal

<div class="sw-grid">

<div class="sw-card">
<span class="sw-tag">shell</span>

### [Bash](https://www.gnu.org/software/bash/)
The shell that's been there since before you were born. Simple, portable, everywhere.
</div>

<div class="sw-card">
<span class="sw-tag">shell</span>

### [Zsh](https://www.zsh.org/)
Bash's cooler sibling. Tab completion, globbing, and plugin support that actually makes the terminal enjoyable.
</div>

<div class="sw-card">
<span class="sw-tag">prompt</span>

### [Starship](https://github.com/starship/starship)
Cross-shell prompt written in Rust. Fast, pretty, and infinitely configurable.
</div>

<div class="sw-card">
<span class="sw-tag">multiplexer</span>

### [tmux](https://github.com/tmux/tmux)
Terminal multiplexer. SSH into a box, detach, come back tomorrow. Your session is still there.
</div>

<div class="sw-card">
<span class="sw-tag">coreutils</span>

### [eza](https://github.com/eza-community/eza)
Modern replacement for `ls`. Colors, git status, and tree view out of the box. Written in Rust, obviously.
</div>

</div>

# Utilities / Apps

<div class="sw-grid">

<div class="sw-card">
<span class="sw-tag">macOS</span>

### [AltTab](https://alt-tab-macos.netlify.app/)
Why does macOS think a minimized window is an inactive window and not let you switch to it? This fixes that.
</div>

</div>
