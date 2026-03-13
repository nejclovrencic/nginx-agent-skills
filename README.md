# Nginx & OpenResty Development

This repository provides an agent skill definition for Nginx and OpenResty development. It follows the open [agent skills](https://agentskills.io/home) format.

Built by Nejc Lovrenčič, Systems Engineer at Automattic, running a global CDN that processes over 700 billion monthly requests.
Want to go deeper on nginx, CDN optimization, and edge computing? Read Cache & Carry, my technical blog at [nejc.blog](https://nejc.blog/), covering undocumented nginx behaviors, OpenResty patterns, and edge infrastructure at scale.

## Installation

### Claude Code & Codex CLI

Clone the repo once and symlink it into both tools:

```bash
git clone https://github.com/nejclovrencic/nginx-agent-skills.git ~/agent-skills/nginx-dev

# Claude Code
ln -s ~/agent-skills/nginx-dev ~/.claude/skills/nginx-dev

# Codex CLI
ln -s ~/agent-skills/nginx-dev ~/.codex/skills/nginx-dev
```

Updates are a single `git pull` in `~/agent-skills/nginx-dev` and both tools pick up the changes immediately.

You can also [download the ZIP from GitHub](https://github.com/nejclovrencic/nginx-agent-skills/archive/refs/heads/main.zip) and extract it instead of cloning, but you'll need to re-download and replace the files whenever the skill is updated.

### Claude web

[Download the ZIP from GitHub](https://github.com/nejclovrencic/nginx-agent-skills/archive/refs/heads/main.zip) and import it via **Settings > Capabilities > Skills > Upload skill**. The skill persists across conversations, but you'll need to re-upload it to pick up updates.

### ChatGPT web

[Download the ZIP from GitHub](https://github.com/nejclovrencic/nginx-agent-skills/archive/refs/heads/main.zip) and upload it directly to a conversation as context. You'll need to re-upload it each new conversation.

