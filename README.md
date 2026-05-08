# agent-chatbot
> Consolidated KĀDI chat agent that listens to and sends messages for Discord and Slack.

Overview
========
agent-chatbot is a KĀDI (KĀDI Agents) multi-platform chat agent that consolidates inbound event listeners and outbound tools for Discord and Slack. It replaces separate repos for platform clients and servers and uses a single Kadi broker connection for both event publishing and tool registration. Platforms can be enabled/disabled via config.toml or environment variables.

Quick Start
===========
1. Clone the repository and install dependencies:
```bash
git clone <repo-url>
cd agent-chatbot
npm install
```

2. (Optional) Compile TypeScript for production / when running the compiled entrypoint:
```bash
npm run setup    # runs npx tsc
# or
npm run build
```

3. Install the agent into your KĀDI environment (requires kadi CLI):
```bash
kadi install
```

4. Configure the agent. The agent reads configuration from config.toml (see repo config.toml). Secrets should be provided via Kadi secrets vaults (recommended) or environment variables. Example minimal config.toml (see config.toml in repo for full example):
```toml
[broker.remote]
URL = "wss://broker.kadi.build/kadi"
NETWORKS = ["chatbot"]

[bot.discord]
ENABLED = true
USER_ID = "1438685741751210025"

[bot.slack]
ENABLED = true
USER_ID = "U09SCDV78AK"
HTTP_PORT = 3700

[logging]
LEVEL = "debug"
```
Secrets required (provide via Kadi vault or env):
- DISCORD_TOKEN
- SLACK_BOT_TOKEN
- SLACK_SIGNING_SECRET
- ARCADE_USERNAME
- ARCADE_PASSWORD

5. Start the agent:
- In a Kādi-managed environment (recommended / deploy): this runs the configured start script (node dist/index.js)
```bash
kadi run start
```
- Locally, after building (or for production-like run):
```bash
npm run start   # runs node dist/index.js (requires dist/index.js to exist)
```
- For active local development with TypeScript watcher:
```bash
npm run dev   # uses tsx watch src/index.ts
```

Tools
=====
The agent registers platform-specific tools with the KĀDI broker so other agents or services can invoke messaging and channel operations.

| Tool | Description |
|------|-------------|
| registerDiscordTools | Registers Discord-specific tools with Kādi (outbound message sending, channel management and utility tools for Discord). Implemented in ./platforms/discord/tools.js and registered on broker connection. |
| registerSlackTools | Registers Slack-specific tools with Kādi (outbound message sending, block/message utilities and workspace interactions). Implemented in ./platforms/slack/tools.js and registered on broker connection. |

Configuration
=============
Configuration is driven primarily by config.toml (see repo config.toml). Environment variables can override specific settings. The agent uses agents-library.readConfig and loadVaultCredentials to load non-secret and secret configuration respectively.

Broker configuration
- Define brokers in config.toml under [broker.local] and/or [broker.remote] with URL and NETWORKS.
- Environment overrides:
  - KADI_BROKER_URL_LOCAL — override broker.local.URL
  - KADI_BROKER_URL_REMOTE — override broker.remote.URL
  - KADI_NETWORK_LOCAL — comma-separated list to override broker.local.NETWORKS
  - KADI_NETWORK_REMOTE — comma-separated list to override broker.remote.NETWORKS

Broker resolution rules (src/index.ts):
- If broker.local is present, it is used as the primary broker.
- If both local and remote are configured, the non-primary broker becomes an additional broker connection (additionalBrokerUrl/additionalBrokerNetworks).

Platform toggles and non-secret fields (config.toml keys with optional env overrides)
- bot.discord.ENABLED (boolean) — overridden by DISCORD_ENABLED env (set to "true" or "false")
- bot.slack.ENABLED (boolean) — overridden by SLACK_ENABLED env (set to "true" or "false")
- bot.discord.USER_ID — overridden by DISCORD_BOT_USER_ID env
- bot.discord.GUILD_ID — overridden by DISCORD_GUILD_ID env
- bot.slack.USER_ID — overridden by SLACK_BOT_USER_ID env
- bot.slack.HTTP_PORT — overridden by SLACK_HTTP_PORT env

Secrets
- Secrets are loaded via agents-library.loadVaultCredentials (Kadi secrets vaults / secrets.toml). The deploy configuration in agent.json requires these vaults for production deployments.
- The repository config.toml includes a [secrets] section where VAULTS and KEYS can be declared (see config.toml). This lists expected vault names and secret keys that the agent will attempt to load.
- Required secret names:
  - DISCORD_TOKEN
  - SLACK_BOT_TOKEN
  - SLACK_SIGNING_SECRET
  - ARCADE_USERNAME
  - ARCADE_PASSWORD
- Deploy configuration (agent.json) declares two vaults:
  - "chatbot" (required: DISCORD_TOKEN, SLACK_BOT_TOKEN, SLACK_SIGNING_SECRET)
  - "arcadedb" (required: ARCADE_USERNAME, ARCADE_PASSWORD)
- In production the container entrypoint runs:
  kadi secret receive --vault chatbot --vault arcadedb && kadi run start
- You may also provide these via environment variables for local development.

Deploy notes
------------
- agent.json contains an akash deploy configuration (image: agent-chatbot:0.1.7) that runs the secret fetch command above and then starts the agent.
- The akash service exposes port 3000 and sets ARCADE_HOST and ARCADE_PORT environment variables for the container (see agent.json deploy section).

ArcadeDB configuration
- The agent includes optional ArcadeDB configuration in config.toml under [arcadedb]:
  - HOST — ArcadeDB host (e.g. arcadedb.dadavidtseng.com)
  - PORT — ArcadeDB port (e.g. 443)
  - USERNAME — DB username
  - DATABASE — DB name
- Deploy/containers may supply ARCADE_HOST and ARCADE_PORT environment variables (see agent.json deploy).

Logging and identity
- logging.LEVEL in config.toml controls log level (e.g. debug, info). The agent sets its tag from agent.ID in config.toml.

Files of interest:
- agent.json — agent metadata, scripts (preflight/setup/start/dev/build/type-check/lint/test), build config, deploy and secrets config (includes deploy command that fetches chatbot + arcadedb vaults). Entrypoint: dist/index.js.
- config.toml — primary agent configuration (broker, bots, logging, secrets/vaults, arcadedb)
- src/index.ts — main agent bootstrap and configuration (broker resolution, platform enablement)
- ./platforms/discord/client.js — Discord platform client implementation
- ./platforms/discord/listener.js — Discord event listener
- ./platforms/discord/tools.js — Discord tool registrations
- ./platforms/slack/client.js — Slack platform client implementation
- ./platforms/slack/listener.js — Slack event listener
- ./platforms/slack/tools.js — Slack tool registrations

Architecture
============
High-level data flow and key components:

- KadiClient (src/index.ts)
  - Single connection to the Kādi broker (primary broker from config.toml or env).
  - Responsible for registering tools and publishing/subscribing to events across configured networks.
  - Broker connection is shared for both inbound event publishing and outbound tool invocation. If both local and remote brokers are configured, the agent can connect to both (primary + additional).

- Platform Clients
  - DiscordPlatformClient (./platforms/discord/client.js)
  - SlackPlatformClient (./platforms/slack/client.js)
  - Responsible for low-level interactions with the respective platform SDKs (discord.js, @slack/bolt / @slack/web-api).
  - Provide APIs used by listeners and tool registrars to send messages, fetch channels/users, etc.

- Listeners
  - DiscordListener (./platforms/discord/listener.js)
  - SlackListener (./platforms/slack/listener.js)
  - Listen for inbound platform events (messages, interactions).
  - Translate platform events into Kādi events (published via KadiClient) and route them onto configured Kādi networks.

- Tool Registration
  - registerDiscordTools (./platforms/discord/tools.js)
  - registerSlackTools (./platforms/slack/tools.js)
  - Register outbound actions with the Kādi broker so remote callers can invoke platform actions (send message, update message, fetch channel info, etc).

Typical runtime flow:
1. Agent loads configuration from config.toml via agents-library readConfig; environment variables may override specific fields.
2. Agent loads secrets via loadVaultCredentials (Kadi secrets vaults) or from environment variables.
3. KadiClient connects to the broker(s) and joins networks specified.
4. Enabled platform clients are initialized (Discord and/or Slack).
5. Platform-specific listeners attach to platform SDKs and publish inbound events to Kādi networks.
6. Platform-specific tools are registered with the broker so remote callers can invoke outbound actions. Calls go

---

---