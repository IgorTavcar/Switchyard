# Coding-Agent Authentication

Claude Code, Codex CLI, and OpenClaw each ship their own sign-in flow, and each
can authenticate either against the vendor's subscription (an OAuth login tied
to a Claude, ChatGPT, or xAI plan) or against a provider API key billed per
token. `switchyard launch` overrides that choice: it points the agent at the
local proxy and hands it a placeholder credential, so the agent never contacts
the vendor's auth service. Billing moves off the subscription and onto whichever
API key the selected route's LLM client resolves.

This is deliberate — the proxy is the upstream from the agent's point of view,
and it needs the agent to send requests rather than refuse to start. It is also
easy to forget, because the agent's own `/login` state stays on disk untouched
and looks unchanged when the session ends.

## What each launcher overrides

| Launcher | Override | Value |
|---|---|---|
| Claude Code | `ANTHROPIC_BASE_URL` | `http://127.0.0.1:<port>` |
| | `ANTHROPIC_AUTH_TOKEN` | `switchyard` — an opaque token that skips Console OAuth |
| | `ANTHROPIC_API_KEY` | `""` — silences the auth-conflict warning |
| Codex CLI | `model_providers.switchyard.base_url` | `http://127.0.0.1:<port>/v1` |
| | `model_providers.switchyard.env_key` | `OPENAI_API_KEY` |
| | `model_providers.switchyard.requires_openai_auth` | `false` |
| | `OPENAI_API_KEY` | `switchyard` |
| OpenClaw | `models.providers.switchyard.baseUrl` | `http://127.0.0.1:<port>/v1` |
| | `models.providers.switchyard.apiKey` | `${SWITCHYARD_API_KEY}` |
| | `SWITCHYARD_API_KEY` | `switchyard` |

The Codex overrides are transient `-c` flags on the `codex` command line and the
OpenClaw provider block is written to a temporary workspace, so neither
launcher mutates the agent's persisted configuration. See
[`claude_code_launcher.py`](../../switchyard/cli/launchers/claude_code_launcher.py),
[`codex_cli_launcher.py`](../../switchyard/cli/launchers/codex_cli_launcher.py),
and [`openclaw_launcher.py`](../../switchyard/cli/launchers/openclaw_launcher.py).

The placeholder value is never validated by Switchyard. It exists because each
agent refuses to start without *some* credential; the proxy accepts any value.

## Native sign-in versus launching through Switchyard

| Agent | Native sign-in | Under `switchyard launch` |
|---|---|---|
| Claude Code | OAuth against a Claude subscription, or `ANTHROPIC_API_KEY` for pay-as-you-go | Placeholder token; billed to the route's upstream key |
| Codex CLI | ChatGPT account sign-in, or `OPENAI_API_KEY` for pay-as-you-go | Placeholder key; billed to the route's upstream key |
| OpenClaw | Per-provider credentials from its own configuration | Placeholder key; billed to the route's upstream key |

For all three, setting the vendor's API-key environment variable during a native
session overrides subscription auth and reverts that session to per-token
billing — the same mechanism the launchers use, just aimed at the vendor instead
of the proxy.

Note that a vendor subscription and that vendor's developer API are separate
billing tracks in every case: a Claude, ChatGPT, or xAI plan does not fund API
requests, and API credit does not raise subscription limits. Routing an agent
through Switchyard therefore does not draw down subscription usage at all, and
routing it back does not consume API credit.

## Choosing between them

Run the agent directly when you want its own subscription to cover the session
and do not need routing, translation, or multi-endpoint serving. Run it under
`switchyard launch` when you want a Switchyard route to pick the backend — for
A/B tests, health-aware failover, stage routing, or serving a model the agent
has no native support for. The tradeoff is billing, not capability.

There is no supported way to keep the vendor subscription while routing through
the proxy. The subscription credential authenticates against the vendor's own
endpoint, which the proxy replaces.

## Where the upstream key comes from

Once a request reaches Switchyard, the placeholder is discarded and the route's
LLM client supplies the real credential from the environment variable named by
its `api_key_env`:

```toml
[llm_clients.openrouter]
format = "openai_chat"
base_url = "https://openrouter.ai/api/v1"
api_key_env = "OPENROUTER_API_KEY"
```

The packaged launcher deployment uses OpenRouter and reads `OPENROUTER_API_KEY`.
A custom deployment passed with `--config` bills to whatever keys its clients
name. If that variable is unset, the call fails upstream rather than falling
back to the agent's own credentials.
